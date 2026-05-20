# Cloudflare 代理模式（橙云 vs 灰云）速查

## 核心区别：流量走不走 Cloudflare

```
DNS only（灰云）                    Proxied（橙云）
─────────────                       ───────────────
浏览器                              浏览器
  │                                   │
  │ DNS 查询 → CF 返回真实 IP         │ DNS 查询 → CF 返回 CF 自己的 IP
  │                                   │
  └──────► 你的源站服务器             └──────► Cloudflare 边缘节点
                                                  │（处理 CDN/WAF/SSL）
                                                  └──────► 你的源站服务器
```

## 能力对比

| 维度 | DNS only（灰云） | Proxied（橙云） |
|---|---|---|
| DNS 查询返回的 IP | 你服务器的真实 IP | CF 边缘节点 IP（真实 IP 被隐藏） |
| HTTP/HTTPS 流量 | 直连源站 | 先到 CF，CF 再转发到源站 |
| CDN 缓存 | ❌ 没有 | ✅ 自动缓存静态资源 |
| DDoS 防护 | ❌ 没有 | ✅ 自动拦截 |
| WAF / Bot 防护 | ❌ 没有 | ✅ 自动启用 |
| 真实 IP 是否暴露 | ✅ 暴露 | ❌ 隐藏 |
| Page Rules / Workers | ❌ 不生效 | ✅ 生效 |
| CF 提供的 SSL | ❌ 不生效 | ✅ CF 证书 + 源站证书双向 |
| 响应日志/分析 | ❌ 看不到 | ✅ 完整 |
| 支持端口 | 任意（CF 不参与） | 只代理 80/443 等 HTTP 端口 |

## 适用场景

**灰云适合**：
- 首次用 certbot 签 Let's Encrypt 证书（HTTP-01 校验需直达源站，橙云会拦截）
- 非 HTTP 服务：SSH、SMTP、自定义 TCP 端口等
- 临时调试源站（绕过 CF 排查问题）

**橙云适合**：
- 正常生产环境的 HTTP/HTTPS 网站（99% 场景）
- 想隐藏服务器 IP 避免被直接攻击
- 想吃 CDN 缓存加速静态资源

## 常见坑

| 坑 | 原因 | 解决 |
|---|---|---|
| certbot 签证书失败 | 橙云拦截了 Let's Encrypt 的 HTTP-01 验证请求 | 切灰云签完证书再切回橙云 |
| 525 错误 | CF SSL 模式选 Full strict，但源站没合法证书 | 用 Let's Encrypt 给源站签证书；或临时改 SSL 模式为 Full（不验证证书） |
| 改了静态文件浏览器还是看到旧版 | CF 边缘节点缓存住了 | CF Dashboard → Caching → Purge Everything |
| 源站日志里所有 IP 都是 CF 的 | 橙云模式下源站直连方是 CF 节点 | nginx 从 `X-Forwarded-For` 取真实客户端 IP（本项目的 vhost 配置已加） |
| 子域名不能 SSH | 橙云只代理 HTTP 端口 | SSH 类用途的子域名设灰云 |

## SSL 模式选择（CF Dashboard → SSL/TLS → Overview）

| 模式 | CF→源站走 HTTP/HTTPS | 是否验证源站证书 | 何时用 |
|---|---|---|---|
| Off | HTTP | - | 完全关闭 SSL，不推荐 |
| Flexible | HTTP | - | 浏览器→CF 是 HTTPS，CF→源站是 HTTP。**源站可以没证书**但容易混合内容报错 |
| Full | HTTPS | 不验证 | 源站需要装证书（自签都行），中等安全 |
| **Full (strict)** | HTTPS | 严格验证 | **推荐**。源站必须装合法证书（Let's Encrypt 即可），端到端加密 |

本项目使用 **Full (strict)** + 源站 Let's Encrypt 证书。

## 操作位置

Cloudflare Dashboard → 选域名 → DNS → Records → 每条记录右侧有云朵图标，点击切换橙/灰

切换后**立即生效**（DNS 缓存可能让客户端继续看到旧值几分钟）。

## 一句话总结

- **灰云** = Cloudflare 只当 DNS 解析器用
- **橙云** = Cloudflare 当反代 / CDN / 防火墙用

正常生产用橙云；遇到要直连源站的场景（签证书、调试、非 HTTP 端口）才切灰云。
