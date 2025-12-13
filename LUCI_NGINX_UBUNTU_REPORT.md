# OpenWrt 代码框架速览与 LuCI 前后端交互机制（含 Ubuntu + Nginx 类似部署与 Demo）

## 1) 这份仓库的“代码框架”是什么

这个仓库主要是 **OpenWrt 的固件/包构建系统（buildroot）**，核心关注点是“如何把目标系统、内核、包、配置拼装成固件”，而不是一个单体 Web 应用。

常用目录定位：

- `package/`：包配方（`package/<分类>/<包名>/Makefile`）与补丁（`patches/0001-*.patch`）
- `target/`：各架构/板级支持、内核配置与镜像生成逻辑（如 `target/linux/<arch>/`）
- `include/`：构建系统 `.mk` 规则与公共逻辑（`rules.mk`、`package.mk` 等会被大量 include）
- `scripts/`：feeds 管理、配置/构建辅助脚本
- `toolchain/`、`tools/`：交叉工具链与主机工具构建

## 2) LuCI 在这棵树里从哪里来（Feeds）

LuCI 默认 **不直接放在主仓库源码里**，而是通过 feeds 拉取。你可以在 `feeds.conf.default` 看到：

- `src-git luci https://git.openwrt.org/project/luci.git`

典型流程（构建机上执行）：

```sh
./scripts/feeds update luci
./scripts/feeds install -a -p luci
```

执行后，LuCI 的包与源码会以“已安装 feed”的形式出现在 `package/feeds/luci/` 下，并参与 `make menuconfig`、`make` 的常规打包流程。

## 3) LuCI 的前后端交互：关键组件与请求路径

LuCI 的“前端”通常是浏览器中的 HTML/JS/CSS；“后端”是 OpenWrt 上的 RPC 与配置系统（ubus/UCI 等）。这套交互依赖以下基础设施（均可在本仓库的包定义中找到）：

### 3.1 HTTP 入口：uhttpd（以及 Lua/ubus 插件）

`package/network/services/uhttpd/` 定义了 OpenWrt 常用的轻量 Web 服务器 `uhttpd`，其 UCI 默认配置在：

- `package/network/services/uhttpd/files/uhttpd.config`

其中两条配置直接点出 LuCI 的典型入口：

- `option home /www`：静态资源根目录
- `list lua_prefix "/cgi-bin/luci=/usr/lib/lua/luci/sgi/uhttpd.lua"`：把 `/cgi-bin/luci/...` 交给 LuCI 的 Lua 入口处理（LuCI 来自 feed，路径在系统里由 LuCI 包安装提供）

此外，`uhttpd` 还有 `uhttpd-mod-ubus` 插件，会把 HTTP 请求桥接到 ubus。该插件的默认启用前缀通常是 `/ubus`（见 `package/network/services/uhttpd/files/ubus.default` 会把 `uhttpd.main.ubus_prefix` 设为 `/ubus`）。

### 3.2 系统消息总线：ubus/ubusd（进程间 RPC）

OpenWrt 用 `ubusd` 提供系统级消息总线（Unix socket），各种系统服务把能力以“ubus 对象/方法”暴露出来。

本仓库对应包：

- `package/system/ubus/`（包含 `ubusd`、`ubus` CLI、`libubus` 等）

socket 路径在默认配置中常见为：

- `/var/run/ubus/ubus.sock`（例如 `package/system/rpcd/files/rpcd.config`）

### 3.3 Web 可调用的 RPC：rpcd（鉴权/ACL + 插件）

`rpcd` 是 OpenWrt 常用的 RPC 后端之一，用于把若干系统能力以更适合前端消费的方式暴露出来，并配合 ACL 做权限控制。

本仓库对应包：

- `package/system/rpcd/`（含插件机制，例如 `rpcd-mod-file`、`rpcd-mod-rpcsys` 等，见 `package/system/rpcd/Makefile`）

### 3.4 典型交互链路（把它当作“LuCI 的数据通道”）

1) 浏览器加载页面与静态资源：

- 浏览器 `GET /`、`/luci-static/...`（通常位于 `/www` 下）
- uhttpd 直接返回静态文件

2) 浏览器发起“读取/修改系统状态”的请求：

- 前端 JS `POST /ubus`（JSON-RPC 风格）
- uhttpd 的 ubus 插件把请求转发到 `ubusd` socket
- `ubusd` 将调用分发给提供该方法的服务（包含 `rpcd` 及其他系统守护进程）
- 返回 JSON 给前端，前端渲染状态/提示结果

你可以把它类比为：**LuCI(前端) ⇄ /ubus(JSON-RPC) ⇄ ubus(系统 RPC 总线) ⇄ 各系统服务/插件**。

### 3.5 会话与权限控制（LuCI 为什么能“安全地”做管理操作）

- **会话（Session）**：`uhttpd-mod-ubus` 会在 ubus 上发布 `session.*` 能力（见 `uhttpd` 包描述），前端登录后获得一个会话 token，后续 RPC 调用携带该 token 才能执行敏感操作。
- **鉴权开关**：uhttpd 支持对 ubus-rpc 请求做会话校验，配置项在 `package/network/services/uhttpd/files/uhttpd.config` 中以 `no_ubusauth` 形式出现（并在 `uhttpd.init` 中映射为启动参数）。生产环境不要开启该选项。
- **ACL（权限）**：`rpcd` 会读取 `/usr/share/rpcd/acl.d/*.json` 来限制哪些 ubus 对象/方法可被调用；不同应用（例如 LuCI 的各 app）通常会带各自的 ACL 片段。

## 4) 在 Ubuntu 上复刻“类似 LuCI 的交互效果”的思路（Nginx 前端）

Ubuntu 上没有 ubus/rpcd 这套默认栈，但可以用同样的分层思想复刻：

- **Nginx**：负责静态前端（HTML/JS）与反向代理（把 `/ubus` 或 `/api/*` 转发到后端）
- **后端服务（Python/Go/Node）**：提供一个“可被前端调用”的 RPC/REST 接口；内部可再调用系统命令/数据库/其他服务（等价于 OpenWrt 的“各种 ubus 服务”）

为了贴近 LuCI，你可以选择：

- 接口风格用 JSON-RPC（更像 `/ubus`），或
- 用更常见的 REST（更易于与现代前端对接）

## 5) Ubuntu + Nginx 部署方案（最小可用）

先安装依赖（示例）：

```sh
sudo apt update
sudo apt install nginx python3
```

### 5.1 目录建议

- 前端静态文件：`/var/www/hello-ui/`
- 后端服务代码：`/opt/hello-api/`

### 5.2 Nginx 站点示例（静态 + 反代 `/ubus`）

保存为 `/etc/nginx/sites-available/hello-ui`：

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/hello-ui;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # 模拟 OpenWrt 的 /ubus：前端发 JSON-RPC 到这里
    location = /ubus {
        proxy_pass http://127.0.0.1:8080/ubus;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

启用并重载：

```sh
sudo ln -s /etc/nginx/sites-available/hello-ui /etc/nginx/sites-enabled/hello-ui
sudo nginx -t && sudo systemctl reload nginx
```

### 5.3 （可选）用 systemd 托管后端进程

创建 unit：`/etc/systemd/system/hello-api.service`

```ini
[Unit]
Description=Hello JSON-RPC API (demo)
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/hello-api
ExecStart=/usr/bin/python3 /opt/hello-api/server.py
Restart=on-failure

# 生产建议：创建专用用户并加固（这里给出最小示例）
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

启用并启动：

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now hello-api.service
```

## 6) Demo：后端实现 `hello ${USER}`，前端输入 USER 并查看结果

下面 demo 用 **JSON-RPC 2.0** 模拟 `/ubus` 的“前端调用后端”模式。

### 6.1 后端（Python，零依赖）

保存为 `/opt/hello-api/server.py`：

```python
#!/usr/bin/env python3
from __future__ import annotations

import json
import re
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

USER_RE = re.compile(r"^[A-Za-z0-9_.-]{1,64}$")


class Handler(BaseHTTPRequestHandler):
    server_version = "hello-api/0.1"

    def _send_json(self, status: int, payload: dict) -> None:
        body = json.dumps(payload, ensure_ascii=False).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Cache-Control", "no-store")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def do_POST(self) -> None:
        if self.path != "/ubus":
            self._send_json(404, {"error": "not found"})
            return

        length = int(self.headers.get("Content-Length", "0") or "0")
        if length <= 0 or length > 2048:
            self._send_json(400, {"error": "invalid content length"})
            return

        try:
            req = json.loads(self.rfile.read(length).decode("utf-8"))
        except Exception:
            self._send_json(400, {"error": "invalid json"})
            return

        rpc_id = req.get("id")
        if req.get("jsonrpc") != "2.0":
            self._send_json(400, {"jsonrpc": "2.0", "id": rpc_id, "error": {"message": "jsonrpc must be 2.0"}})
            return

        if req.get("method") != "hello":
            self._send_json(404, {"jsonrpc": "2.0", "id": rpc_id, "error": {"message": "unknown method"}})
            return

        params = req.get("params") or {}
        user = params.get("user", "")
        if not isinstance(user, str) or not USER_RE.match(user):
            self._send_json(400, {"jsonrpc": "2.0", "id": rpc_id, "error": {"message": "invalid user"}})
            return

        self._send_json(200, {"jsonrpc": "2.0", "id": rpc_id, "result": {"message": f"hello {user}"}})

    def log_message(self, fmt: str, *args) -> None:
        return


def main() -> None:
    httpd = ThreadingHTTPServer(("127.0.0.1", 8080), Handler)
    httpd.serve_forever()


if __name__ == "__main__":
    main()
```

运行：

```sh
python3 /opt/hello-api/server.py
```

### 6.2 前端（HTML + Fetch）

保存为 `/var/www/hello-ui/index.html`：

```html
<!doctype html>
<meta charset="utf-8" />
<title>Hello RPC Demo</title>
<style>
  body { font-family: system-ui, sans-serif; max-width: 720px; margin: 40px auto; }
  input { padding: 8px; width: 260px; }
  button { padding: 8px 12px; }
  pre { background: #f6f8fa; padding: 12px; }
</style>

<h1>hello ${USER}</h1>
<p>输入 USER（仅允许 A-Z a-z 0-9 _ . -，最长 64）</p>
<input id="user" placeholder="e.g. alice" />
<button id="run">Run</button>

<h2>Result</h2>
<pre id="out">(waiting)</pre>

<script>
  const $ = (id) => document.getElementById(id);
  $("run").addEventListener("click", async () => {
    const user = $("user").value.trim();
    $("out").textContent = "loading...";
    const res = await fetch("/ubus", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ jsonrpc: "2.0", id: Date.now(), method: "hello", params: { user } })
    });
    const data = await res.json();
    $("out").textContent = data?.result?.message || data?.error?.message || JSON.stringify(data);
  });
</script>
```

### 6.3 快速验证（不用浏览器）

```sh
# 走 Nginx（推荐）
curl -s http://localhost/ubus \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"hello","params":{"user":"alice"}}'

# 或直接打后端
curl -s http://127.0.0.1:8080/ubus \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"hello","params":{"user":"alice"}}'
```

## 7) 安全与工程化建议（从 LuCI 经验迁移）

- 不要把“执行任意命令”直接暴露给前端；像上面这样把输入限制为安全子集。
- 接口前置鉴权/会话：OpenWrt 依赖 `session.*` 与 ACL（`rpcd` 的 `acl.d`）；Ubuntu 侧可用 JWT/Session Cookie + RBAC。
- 通过 Nginx 限制请求体、速率，并在公网部署时启用 TLS（Let’s Encrypt）。
