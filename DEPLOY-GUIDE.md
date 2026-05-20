# 子域名部署新前端项目 · 全流程速查手册

> 场景：已有一台 AWS Lightsail 服务器，主域 `groundedglow.cc` 已通过 Cloudflare + Nginx + Let's Encrypt + Docker + GitHub Actions + ECR 跑起来。现在要新加一个子域名（如 `attention.groundedglow.cc`）指向一个**全新的前端项目**，不影响原有服务。本文为本次部署 `know-your-attention` 的全过程复盘，下一个项目按此模板替换名字即可。

## 0. 整体架构（先建立心智模型）

```
浏览器 ───HTTPS───► Cloudflare ───►  Lightsail 服务器
                  (DNS + CDN)        │
                                     ├─ 宿主机 Nginx (443/80)
                                     │    ├─ groundedglow.cc          → 127.0.0.1:3000 → light-blog-fe 容器
                                     │    ├─ groundedglow.cc/api/     → 127.0.0.1:8081 → light-blog-api 容器
                                     │    └─ attention.groundedglow.cc → 127.0.0.1:8090 → attention-fe  容器  ← 新增
                                     │
                                     ├─ Docker 容器
                                     │    ├─ /home/ubuntu/light-blog/docker-compose.yml          (原有)
                                     │    └─ /home/ubuntu/know-your-attention/docker-compose.yml (新增)
                                     │
                                     └─ Certbot 自动续期证书 /etc/letsencrypt/live/...

GitHub 仓库 ───push main───► GitHub Actions
                              ├─ 1. docker build
                              ├─ 2. docker push → AWS ECR
                              └─ 3. SSH 到服务器 → docker compose pull + up
```

**4 个角色各自的职责：**

| 角色 | 负责什么 |
|---|---|
| **Cloudflare** | DNS 解析（把 `attention.xxx.cc` 解到服务器 IP）+ 可选 CDN |
| **宿主机 Nginx** | HTTPS 终止 + 按 `Host` 头分发到不同容器端口 |
| **Docker 容器（内层 Nginx）** | 服务静态文件 + SPA fallback |
| **GitHub Actions + ECR** | CI/CD：构建镜像、推到镜像仓库、远程触发服务器拉新镜像 |

> 关键认知：**宿主机 Nginx 才是"网关"**，所有外部请求都先经过它，再根据子域名转给某个容器端口。容器之间互不知道彼此存在。

## 1. 项目隔离策略

| 资源 | 命名规则 | 本次值 | 下个项目套用 |
|---|---|---|---|
| 子域名 | `<项目>.groundedglow.cc` | `attention.groundedglow.cc` | `xxx.groundedglow.cc` |
| 宿主机端口 | 从 8090 递增 | `8090` | `8091` `8092` ... |
| ECR 仓库 | `<项目>/<项目>-fe` | `know-your-attention/know-your-attention-fe` | 同 |
| 服务器部署目录 | `/home/ubuntu/<项目>` | `/home/ubuntu/know-your-attention` | 同 |
| Nginx vhost 文件 | `/etc/nginx/sites-available/<子域名>` | `/etc/nginx/sites-available/attention.groundedglow.cc` | 同 |
| 容器名 | `<项目>-fe` | `know-your-attention-fe` | 同 |

**核心原则：每个项目一个独立目录 + 独立 docker-compose.yml**。不要把多个项目的 service 混到同一份 compose 文件里，否则越加越乱。

## 2. 全流程步骤

### Step 1 · Cloudflare 加 DNS 记录

| 操作 | 值 |
|---|---|
| Type | `A` |
| Name | `attention`（不要写完整域名） |
| IPv4 | Lightsail 公网 IP |
| Proxy status | **先设为 DNS only（灰云）** ← 关键 |
| TTL | Auto |

**踩坑点**：如果一开始就开橙云代理，certbot 申请证书时 HTTP-01 校验可能失败。**先灰云签证书 → 切橙云**。
**验证**：`dig +short attention.groundedglow.cc` 应返回服务器 IP。

### Step 2 · 本地项目添加 4 个文件

在项目根目录新增：

#### 2.1 `Dockerfile`

**先理解一个关键概念：多阶段构建（multi-stage build）**。一个 `Dockerfile` 可以写多个 `FROM`，每一段从一个基础镜像开始，叫做一个 `stage`。最后一个 stage 才是最终镜像；前面 stage 只是用来产出文件，可以从里面 `COPY --from=xxx` 拷东西过来。这样做的好处是：构建过程中要用到的 Node、npm、源码等都不会进最终镜像，最终镜像只剩 nginx + 编译好的静态文件，**体积小、攻击面小**。

下面是完整文件，逐段拆解：

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --prefer-offline --no-audit --progress=false

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG VITE_API_BASE_URL=
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL
RUN npm run build

FROM nginx:alpine AS runner
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**第 1 行：`# syntax=docker/dockerfile:1`**
告诉 Docker 用最新的 Dockerfile 语法解析器（启用 BuildKit 特性，比如下面的 `--mount=type=cache`）。不写也能跑，但缓存挂载就用不了。

**Stage 1 · `deps`（只装依赖）**
- `FROM node:20-alpine AS deps`：基于 Node 20 的 alpine 镜像（小，~50MB），给这个 stage 起名 `deps` 方便后面引用
- `WORKDIR /app`：之后所有命令的工作目录是 `/app`（相当于 `cd /app`）
- `COPY package.json package-lock.json ./`：**只**拷 lockfile，不拷源码。这是关键优化——只要这两个文件没变，这一层的缓存就有效，下次构建直接跳过 `npm ci`
- `RUN --mount=type=cache,target=/root/.npm`：把 npm 的全局缓存目录 `/root/.npm` 挂载成持久缓存，**跨多次 build 复用下载好的包**，避免每次都从 npm 源拉
- `npm ci`：严格按 lockfile 安装（比 `npm install` 更快更确定）；`--prefer-offline`（优先用缓存）、`--no-audit`（不做安全审计）、`--progress=false`（不打印进度条）都是为了加速 + 静音

**为什么要单独一个 deps stage？** 因为 `COPY . .`（拷源码）一定会经常变，如果把 `npm ci` 放在拷源码之后，Docker 缓存就失效了，每次都要重装依赖（慢得要命）。把 deps 拆出来后，源码变化不会影响 deps 层的缓存。

**Stage 2 · `builder`（编译项目）**
- `FROM node:20-alpine AS builder`：新开一个 stage，同样基于 Node
- `COPY --from=deps /app/node_modules ./node_modules`：把 deps stage 里装好的 `node_modules` **整个目录**拷过来（这就是多阶段构建的妙用）
- `COPY . .`：把项目源码全部拷进来（`.dockerignore` 排除的文件不会进来）
- `ARG VITE_API_BASE_URL=`：声明一个**构建参数**，默认值空字符串。`ARG` 只在构建期可用
- `ENV VITE_API_BASE_URL=$VITE_API_BASE_URL`：把构建参数提升为环境变量，让 Vite 在执行 `npm run build` 时能读到。Vite 约定所有 `VITE_*` 开头的环境变量会被打进前端 bundle
- `RUN npm run build`：执行 `package.json` 里的 `build` 脚本（`tsc -b && vite build`），产出 `/app/dist/`

**Stage 3 · `runner`（最终运行）**
- `FROM nginx:alpine AS runner`：换基础镜像！这次用 nginx alpine（~25MB），因为运行期只需要一个 web server 服务静态文件，**不需要 Node 了**
- `COPY nginx.conf /etc/nginx/conf.d/default.conf`：把项目里的 `nginx.conf` 放到 nginx 的默认配置位置（覆盖默认的 server block）
- `COPY --from=builder /app/dist /usr/share/nginx/html`：从 builder stage 把构建产物 `dist/` 拷到 nginx 的默认网站根目录
- `EXPOSE 80`：声明容器内部监听 80 端口（只是文档作用，实际端口映射由 `docker run -p` 决定）
- `CMD ["nginx", "-g", "daemon off;"]`：容器启动时执行的命令——以**前台模式**跑 nginx（Docker 容器要求主进程必须前台运行，否则容器会立即退出）

**最终镜像里有什么？** 只有 nginx + `dist/` 静态文件 + 配置文件。**没有** Node、没有 npm、没有 `node_modules`、没有源码、没有 lockfile。

> **Node 版本选择**：用 `node:20-alpine` 而不是 22。Node 22 + 自带 npm 10.x 在 BuildKit cache mount 下有已知 bug（`npm error Exit handler never called!`），构建直接失败。Node 20 LTS 是当前最稳的选择。如果非要用 22，需要在 `npm ci` 前加一步 `npm install -g npm@latest` 升级到 npm 11.x。
>
> **Vite vs Next.js 注意**：Next.js 用 `output:'standalone'` 出 `server.js`，runtime 必须 Node；Vite 出纯静态 `dist/`，用 nginx 更小更快。**runtime stage 不能照搬 Next.js 项目的写法**。

#### 2.2 `nginx.conf`（容器内）

```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;
  location / { try_files $uri $uri/ /index.html; }   # SPA fallback
  location ~* \.(?:js|css|svg|png|jpg|jpeg|gif|webp|woff2?)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
  }
}
```

> 这是**容器内**的 nginx，和宿主机 nginx 是两层。SPA fallback 必须在这一层做，否则刷新 `/tasks` 等前端路由会 404。

#### 2.3 `.dockerignore`

```
node_modules
dist
.git
.github
*.log
.DS_Store
```

#### 2.4 `.github/workflows/deploy.yml`

两个 job：`build-push`（构建镜像推 ECR）→ `deploy`（SSH 触发服务器 `docker compose pull + up`）。完整内容见仓库内同名文件。关键变量：

```yaml
env:
  AWS_REGION: ap-northeast-2
  ECR_REGISTRY: 115908659860.dkr.ecr.ap-northeast-2.amazonaws.com
  ECR_REPOSITORY: know-your-attention/know-your-attention-fe
  IMAGE_TAG: latest
```

### Step 3 · GitHub Secrets 配置

**什么是 Secrets？** GitHub Actions 跑在 GitHub 提供的"runner"机器上，是一台干净的临时 Linux。它要登录 AWS、要 SSH 进你服务器，就需要密钥/密码。直接写在 workflow yaml 里会被公开（仓库公开则全世界可见），所以 GitHub 提供 "Secrets" 机制：你在网页上录入 → 存到 GitHub 的加密存储 → workflow 里通过 `${{ secrets.XXX }}` 读取 → 日志里自动打码成 `***`。

**录入位置**：仓库主页 → `Settings` → 左侧 `Secrets and variables` → `Actions` → `Repository secrets` → `New repository secret`

**6 个 Secret 详解**：

| Secret | 实际值长什么样 | 用在哪一步 | 作用 |
|---|---|---|---|
| `AWS_ACCESS_KEY_ID` | `AKIAIOSFODNN7EXAMPLE`（20 位字母数字） | `aws-actions/configure-aws-credentials` | AWS IAM 用户的"账号"，告诉 AWS 我是谁 |
| `AWS_SECRET_ACCESS_KEY` | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`（40 位） | 同上 | AWS IAM 用户的"密码"，证明我真的是我；和 ID 是一对，缺一不可 |
| `LIGHTSAIL_HOST` | `54.180.xxx.xxx` 或 `ec2-xxx.compute.amazonaws.com` | `appleboy/ssh-action` 的 `host` | SSH 目标服务器的 IP 或域名 |
| `LIGHTSAIL_USER` | `ubuntu`（Lightsail Ubuntu 镜像默认用户） | 同上 `username` | SSH 登录用的用户名，相当于本地 `ssh ubuntu@xxx` 里的 `ubuntu` |
| `LIGHTSAIL_SSH_KEY` | `-----BEGIN OPENSSH PRIVATE KEY-----\n...\n-----END OPENSSH PRIVATE KEY-----`（多行私钥全文） | 同上 `key` | SSH 私钥内容（不是文件路径！要把整个 .pem 文件内容复制进来）。runner 用它登录你的服务器 |
| `DEPLOY_DIR` | `/home/ubuntu/know-your-attention` | SSH 脚本里 `cd $DEPLOY_DIR` | 告诉远程脚本进哪个目录执行 `docker compose pull`。每个项目一个独立目录 |

**前 2 个怎么来**：在 AWS 控制台 → IAM → Users → 选一个有 ECR 推送权限的用户 → Security credentials → Create access key → 立刻复制下来或下载 csv（**只显示这一次**，关闭窗口后永远查不到）

**后 3 个 LIGHTSAIL_\*** 怎么来：
- `LIGHTSAIL_HOST`：AWS Lightsail 控制台实例详情页能看到 Public IP
- `LIGHTSAIL_USER`：Ubuntu 镜像默认是 `ubuntu`，Amazon Linux 是 `ec2-user`
- `LIGHTSAIL_SSH_KEY`：本地 SSH 进服务器用的那个 `.pem` 文件的**全部内容**。可以本地执行 `cat ~/.ssh/你的key.pem` 后整段复制（包括 BEGIN/END 行和中间所有换行）

**踩坑点：AWS Secret Key 不可回查**。AWS 控制台只在创建那一刻显示一次，之后任何地方都查不到。如果丢了 → IAM 里新建一对（一个 user 允许 2 个 active key，不影响旧的）。

**踩坑点：Credentials could not be loaded** → Secret 名字写错、设到了 Environment secrets / Variables（而不是 Repository secrets）、或私有仓库没继承 org secret。调试加一步打印长度（值会自动打码）：

```yaml
- name: Debug
  run: echo "len=${#AWS_ACCESS_KEY_ID}"
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
```

`AWS_ACCESS_KEY_ID` 长度应是 20，`AWS_SECRET_ACCESS_KEY` 长度应是 40。如果输出 0，说明根本没读到 → 回去检查名字和录入位置。

### Step 4 · AWS ECR 创建镜像仓库

控制台 → ECR → Create repository：
- Visibility: **Private**
- Repository name: `know-your-attention/know-your-attention-fe`（必须和 workflow 里的 `ECR_REPOSITORY` 完全一致）

**踩坑点：IAM 权限**。如果 IAM user 挂的是 managed policy `AmazonEC2ContainerRegistryFullAccess`，新仓库自动覆盖。如果是自定义 policy 且 `Resource` 写死了旧仓库 ARN，需要扩权。

### Step 5 · 服务器准备部署目录

**目的**：在服务器上为新项目开一个独立工作目录，里面放 `docker-compose.yml`（告诉 Docker 怎么跑容器）和 `.env`（环境变量），之后 GitHub Actions 通过 SSH 进来执行 `docker compose pull && docker compose up -d` 就能更新服务。

**逐条命令解释**：

```bash
sudo mkdir -p /home/ubuntu/know-your-attention
# sudo: 用 root 权限执行（创建系统级目录需要）
# mkdir -p: 创建目录，-p 表示如果父目录不存在也一起创建，且目录已存在不报错

sudo chown ubuntu:ubuntu /home/ubuntu/know-your-attention
# chown: change owner，把目录所有者改成 ubuntu 用户
# ubuntu:ubuntu = user:group
# 不改的话目录是 root 所有，后续 ubuntu 用户写文件会权限不足

cd /home/ubuntu/know-your-attention
# 进入这个目录，后面写文件就在这里
```

**接下来要在这个目录写两个文件，用到了 heredoc（here-document）语法**：

```bash
cat > docker-compose.yml <<'EOF'
name: know-your-attention
services:
  attention-fe:
    image: ${ECR_REGISTRY}/know-your-attention/know-your-attention-fe:latest
    container_name: know-your-attention-fe
    restart: unless-stopped
    ports:
      - "8090:80"
EOF
```

**heredoc 是什么？** Linux shell 的一种"多行输入"语法。等价于"把下面这段文本喂给左边的命令的标准输入"。逐部分拆解：

- `cat`：一个简单命令，把"标准输入"原样输出到"标准输出"
- `>`：**重定向**，把标准输出写到文件 `docker-compose.yml`（覆盖式，已存在的同名文件会被清空重写）
- `<< 'EOF'`：开始一个 heredoc，告诉 shell"从下一行开始接收输入，直到看见一行只有 `EOF` 三个字母时停止"
- `'EOF'`（**带单引号**）：表示**不要展开**输入内容里的 `$变量` 和反引号。如果写成 `<<EOF`（不带引号），那 `${ECR_REGISTRY}` 会被 shell 立即替换成空字符串（因为当前 shell 里没这个变量），写进文件的就是空的。**带单引号 = 原样保留**，让 Docker 自己后面解析
- 中间各行：文本内容，**前面不能有 Tab 之类的奇怪字符**
- `EOF`：**必须顶格、行首行尾无任何字符**，是 heredoc 的结束标志。这个名字是约定俗成，叫 `END` `BANANA` 都行，只要前后一致

整段连起来就是："执行 `cat`，标准输入从下方 heredoc 文本来，标准输出重定向写到 `docker-compose.yml`"，最终效果是**创建一个文件并写入下面那段内容**。

**.env 文件同理**：

```bash
cat > .env <<'EOF'
ECR_REGISTRY=115908659860.dkr.ecr.ap-northeast-2.amazonaws.com
EOF
```

写完后 `docker-compose.yml` 里的 `${ECR_REGISTRY}` 会被 Docker Compose 从 `.env` 自动读取替换。这样镜像仓库地址只配一处，多项目复用时也方便。

**验证两个文件写好了**：

```bash
ls -la                                # 应看到 docker-compose.yml 和 .env
cat docker-compose.yml                # 看一眼内容是否正确
```

**为什么不用 `nano` / `vim` 编辑？** 用编辑器也行，heredoc 是"一条命令一次性创建"，写脚本/复制粘贴更顺手，没有编辑器引入的 CRLF、缩进等问题。

### Step 6 · 服务器加 Nginx vhost + 申请证书

#### 6.1 先理解 vhost 是什么

**vhost** = virtual host（虚拟主机），是 Nginx 的术语。指一份**配置块**，告诉 Nginx："如果收到的 HTTP 请求里 `Host` 头是 XXX，就按这段配置处理"。

为什么需要它？因为一台服务器只有一个 IP，但可以挂多个域名（`groundedglow.cc`、`attention.groundedglow.cc`、未来还有更多）。所有域名的请求都打到同一个 IP 的同一个 80/443 端口，Nginx 必须根据请求里的 `Host` 头来区分"这个请求该转给谁"——区分逻辑就是不同的 vhost 配置。

**Ubuntu 上的 Nginx 配置组织方式**：

```
/etc/nginx/
├── nginx.conf                          # 主配置（一般不动）
├── sites-available/                    # 所有可用的 vhost 配置都放这里
│   ├── groundedglow.cc                 # 一个站点一个文件
│   └── attention.groundedglow.cc       # 新增的
└── sites-enabled/                      # 实际生效的 vhost（指向 sites-available 的软链）
    ├── groundedglow.cc -> ../sites-available/groundedglow.cc
    └── attention.groundedglow.cc -> ../sites-available/attention.groundedglow.cc
```

主配置 `nginx.conf` 末尾有一行 `include /etc/nginx/sites-enabled/*;`，意思是"自动加载 sites-enabled 下所有文件"。所以**启用一个站点 = 在 sites-enabled 里建一个软链；停用 = 删软链**。这种"配置库 + 启用开关"分离的设计让加/删站点零风险。

#### 6.2 这一步要做的两件事

1. **写 vhost**：在 `sites-available/` 新建一份配置，告诉 Nginx 把 `attention.groundedglow.cc` 的请求转给 `127.0.0.1:8090`（也就是新项目容器对外暴露的端口）
2. **签 HTTPS 证书**：用 certbot 申请 Let's Encrypt 免费证书。Cloudflare 是 Full strict 模式，要求源站必须有合法证书，否则 CF 到源站这段连接会失败

#### 6.3 第一步：写 vhost 配置文件

```bash
sudo tee /etc/nginx/sites-available/attention.groundedglow.cc > /dev/null <<'EOF'
server {
    listen 80;
    server_name attention.groundedglow.cc;
    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

**`tee` 是什么？** Linux 命令，把"标准输入"同时写到**文件**和**标准输出**。名字来自管道的 T 型分叉。语法：

```
某命令 | tee 文件名         # 内容同时进文件和屏幕
某命令 | tee 文件名 > /dev/null   # 进文件但不显示到屏幕（屏蔽屏幕回显）
```

**为什么用 `sudo tee` 而不是 Step 5 那种 `cat > 文件`？**
- `cat > /etc/nginx/...` 的写法中，**重定向 `>` 是由当前 shell 执行的**，但当前 shell 是普通用户身份，没权限写 `/etc/nginx/`，会报 `Permission denied`
- 即使前面加 `sudo`，写成 `sudo cat > 文件` 也没用——`sudo` 只对 `cat` 有效，重定向 `>` 仍由原 shell 处理
- 解决方法：让 `tee` 来处理写文件这件事，`tee` 本身可以被 `sudo`，从而有权限写 `/etc/nginx/`
- `> /dev/null` 是把 tee 同时往屏幕的输出丢到"黑洞" `/dev/null`，避免刷屏

**heredoc 部分（`<<'EOF' ... EOF`）和 Step 5 完全一样**：单引号 EOF 防止 shell 展开 `$host` `$scheme`（这些是 nginx 的内置变量，要留给 nginx 自己解释，不能被 shell 提前替换成空字符串）。

**配置内容逐行**：

| 行 | 含义 |
|---|---|
| `server { ... }` | 一个 server block = 一个 vhost |
| `listen 80;` | 监听 80 端口（HTTP）。证书签下来后 certbot 会自动加一个 listen 443 ssl 的 server block |
| `server_name attention.groundedglow.cc;` | 只处理 Host 头等于这个域名的请求 |
| `location / { ... }` | 匹配所有路径（`/` 是根路径前缀，匹配所有 URL） |
| `proxy_pass http://127.0.0.1:8090;` | **核心**：把请求转发到本机 8090 端口（也就是 docker compose 暴露的容器端口） |
| `proxy_http_version 1.1;` | 用 HTTP/1.1 与后端通信（支持 keep-alive） |
| `proxy_set_header Host $host;` | 把原始请求的 Host 头透传给后端（否则后端看到的 Host 是 127.0.0.1） |
| `proxy_set_header X-Real-IP $remote_addr;` | 把真实客户端 IP 传给后端（不传的话后端日志里所有请求都来自 127.0.0.1） |
| `proxy_set_header X-Forwarded-For ...;` | 同上，链路追踪用 |
| `proxy_set_header X-Forwarded-Proto $scheme;` | 告诉后端原始请求是 http 还是 https |

#### 6.4 第二步：启用并 reload

```bash
sudo ln -sf /etc/nginx/sites-available/attention.groundedglow.cc /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

**`ln` 是什么？** 创建链接（link）。`ln -s` 创建**软链接**（symbolic link，类似 Windows 的快捷方式）；`-f` 是 force，若目标已存在就覆盖。整句意思：在 `sites-enabled/` 里建一个软链，指向 `sites-available/` 里那份真实配置。这样 `include sites-enabled/*` 就会加载它。

`nginx -t` 是配置语法测试（不真正 reload）。**任何改完配置都要先 test 再 reload**，否则有语法错误会让 nginx 直接挂掉，连已有的站点都访问不了。

`systemctl reload nginx` 让 nginx 平滑重新加载配置，**不中断已有连接**（区别于 `restart` 会断开所有连接）。

#### 6.5 第三步：用 certbot 签 HTTPS 证书

```bash
sudo certbot --nginx -d attention.groundedglow.cc \
  --non-interactive --agree-tos --redirect \
  -m 你的邮箱@example.com
```

参数解释：

| 参数 | 含义 |
|---|---|
| `--nginx` | 用 nginx 插件，certbot 会自动找到对应 vhost 改写它 |
| `-d 域名` | domain，要为哪个域名签证书。可以多个 `-d` |
| `--non-interactive` | 不要交互式问问题，全部用命令行参数和默认值 |
| `--agree-tos` | 同意 Let's Encrypt 服务条款 |
| `--redirect` | 自动加 HTTP → HTTPS 跳转（80 端口 301 到 443） |
| `-m 邮箱` | 接收证书到期提醒和重要通知的邮箱 |

**certbot 内部做了什么？**

1. 找到 vhost 文件 `/etc/nginx/sites-enabled/attention.groundedglow.cc`
2. 在该域名的 webroot 下放一个临时文件 `/.well-known/acme-challenge/xxx`
3. 通知 Let's Encrypt："请来 http://attention.groundedglow.cc/.well-known/acme-challenge/xxx 取这个文件验证我"
4. Let's Encrypt 从公网访问验证（这就是为什么 Cloudflare 那条 A 记录要先开**灰云**——开橙云的话请求会被 CF 拦截，到不了源站）
5. 验证通过，签发证书到 `/etc/letsencrypt/live/attention.groundedglow.cc/`
6. **自动改写 vhost 文件**：加上 `listen 443 ssl`、证书路径、80→443 的 301 跳转 server block
7. `nginx -t && systemctl reload nginx`
8. 装好 systemd timer 自动续期（证书 90 天有效，到期前 30 天自动续）

完成后查看 vhost 文件，能看到 certbot 加了一堆 `# managed by Certbot` 注释——千万不要手动改这些行，否则下次续期可能出问题。

#### 6.6 常见踩坑

**踩坑点：heredoc 在脚本文件中失效**（`warning: here-document delimited by end-of-file`）。常见原因：结束符前后有空格、nano 保存时混入 CRLF（`cat -A file | grep -n EOF` 看到 `^M`）。解决：用 `dos2unix file` 转换；或干脆**不写脚本文件、直接命令行粘 heredoc**（交互式 shell 不会有解析问题）。

**踩坑点：certbot 验证失败**——99% 是 Cloudflare 那条 A 记录开了橙云代理，关掉改成灰云（DNS only）后重试。证书签好后再切回橙云。

**踩坑点：nginx -t 报错**——`grep -n "你怀疑的字段" /etc/nginx/sites-enabled/*` 看是不是别的 vhost 重复定义了同名 server_name。

### Step 7 · 推代码触发部署

```bash
git add Dockerfile nginx.conf .dockerignore .github/
git commit -m "chore: add docker build & deploy workflow"
git push origin main
```

去 GitHub → Actions 看 workflow 跑完。

### Step 8 · 验证

```bash
# 服务器
cd /home/ubuntu/know-your-attention
sudo docker compose ps                              # 看到 attention-fe Up
sudo docker compose logs attention-fe --tail 30

# 浏览器
# https://attention.groundedglow.cc  → 看到新项目
# https://groundedglow.cc/           → 原项目仍正常
```

证书拿到后，回 Cloudflare 把 `attention` A 记录从灰云切回橙云（与主域保持一致）。

## 3. 下次部署新项目的极简清单

> 把所有 `attention` / `know-your-attention` / `8090` 全部替换成新项目的值。

- [ ] **Cloudflare**：A 记录 `<新项目>` → 服务器 IP（灰云）
- [ ] **项目根目录**：加 `Dockerfile`（用 `node:20-alpine`）/ `nginx.conf` / `.dockerignore` / `.github/workflows/deploy.yml`（只改 `ECR_REPOSITORY` 和镜像名）
- [ ] **GitHub Secrets**：只需新增 `DEPLOY_DIR=/home/ubuntu/<新项目>`（其他 5 个 Secret 可复用）
- [ ] **AWS ECR**：建仓库 `<新项目>/<新项目>-fe`
- [ ] **服务器**：`mkdir /home/ubuntu/<新项目>` + 放 `docker-compose.yml` + `.env`（端口排到 8091/8092…）
- [ ] **服务器**：写 vhost + `certbot --nginx -d <新子域名>`
- [ ] **本地**：`git push` 触发 Actions
- [ ] **验证**：浏览器访问新子域名 + `docker compose ps` 看容器状态
- [ ] **Cloudflare**：A 记录切回橙云

## 4. 常用排错命令

```bash
# 容器层
sudo docker compose ps                              # 看服务状态
sudo docker compose logs <service> --tail 50 -f     # 实时日志
sudo docker compose pull && sudo docker compose up -d   # 手动拉新镜像重启

# 宿主机 Nginx
sudo nginx -t                                       # 测语法
sudo systemctl reload nginx                         # 热加载
sudo tail -f /var/log/nginx/error.log               # 错误日志
ls -la /etc/nginx/sites-enabled/                    # 看启用的 vhost

# 证书
sudo certbot certificates                           # 列出所有证书 + 到期日
sudo certbot renew --dry-run                        # 测试续期
sudo systemctl list-timers | grep certbot           # 续期定时任务

# AWS / ECR
aws ecr describe-repositories --region ap-northeast-2
aws ecr get-login-password --region ap-northeast-2 | sudo docker login --username AWS --password-stdin 115908659860.dkr.ecr.ap-northeast-2.amazonaws.com

# 网络
dig +short <域名>                                   # DNS 解析
curl -I https://<域名>                              # HTTPS 探测
curl -sI -H "Host: <域名>" http://127.0.0.1/        # 绕过 DNS 直测宿主机 nginx
```

## 5. 关键文件位置总结

| 文件 | 位置 | 改了之后要做什么 |
|---|---|---|
| 项目 `Dockerfile` | 项目根 | git push 触发重新构建 |
| 项目内层 `nginx.conf` | 项目根 | git push 触发重新构建 |
| 项目 GitHub workflow | `.github/workflows/deploy.yml` | git push 自动生效 |
| 部署目录 `docker-compose.yml` | `/home/ubuntu/<项目>/` | `docker compose up -d` |
| 部署目录 `.env` | `/home/ubuntu/<项目>/` | `docker compose up -d` |
| 宿主机 vhost | `/etc/nginx/sites-available/<子域名>` | `nginx -t && systemctl reload nginx` |
| 证书 | `/etc/letsencrypt/live/<子域名>/` | certbot 自动续期 |

**至此完成。这套架构可无限水平扩展子域名项目，互不影响。**
