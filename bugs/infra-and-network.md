# 基础设施 / 网络

特点：代码层面没问题，是容器编排 / DNS / 端口管理 / 反向代理的坑。

---

## #1 端口 18888 / 18999 被占 — make rebuild 起不了容器

**症状**：`make rebuild` 报 `rootlessport listen tcp 0.0.0.0:18888: bind: address already in use`。
**触发**：从 host 进程模式（`./start.sh`）切到容器化模式时必现。
**根因**：host 上的 `./start.sh start` 还跑着 mall-api 进程占着 18888 / 18999，podman 起容器要绑同一端口。
**修复**：先 `./start.sh stop`，再 `make rebuild`。
**如何避免**：
1. `make rebuild` 脚本开头检测一下 host 进程，提示用户先 stop。
2. 团队约定：要么全用 host 模式开发，要么全用容器，避免混用。

---

## #2 nginx upstream 502 — mall-api 重启后前端永久挂

**症状**：每次 force-recreate mall-api 后浏览器调 /api/* 永远 502，要 `podman restart mall-fe` 才恢复。
**触发**：每次 mall-api / 任何被 nginx proxy 的服务重启后必现。
**根因**：nginx `proxy_pass http://mall-api:18888;` 启动时一次性 DNS 解析，podman rootless 网络下 mall-api 重启换 IP（10.89.0.61 → 别的）后 nginx 还在连旧 IP → unreachable → 502。
**修复**：commit `e741570` — nginx.conf 加 `resolver 10.89.0.1 valid=30s` + 变量形式 `proxy_pass http://$upstream_api`，每个请求重新走 resolver。
**如何避免**：
1. 任何 nginx upstream 用主机名（不是 IP）的场景，**必须**配 resolver + 变量 proxy_pass。
2. 同类问题在 envoy / haproxy / traefik 里有不同写法，新引入的反代要单独验证一次"upstream 重启后是否恢复"。
3. 长期：上 service mesh（如 envoy + service discovery）让重启对客户端透明。
