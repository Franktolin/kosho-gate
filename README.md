# kosho-gate

DSH Web GUI 插件：远程访问控制 + frpc 内网穿透 + 面板密码门禁 + 手机端 UI 适配。

在设置面板里统一管理：

- 远程访问：一键切换 Web GUI 绑定 0.0.0.0 / 127.0.0.1，让局域网设备可访问；关闭时自动还原所有修改。
- frpc 内网穿透：自动下载并拉起 frpc，把面板映射到公网 frps 服务器，可随时开/停。
- 面板密码门禁：非本机访问必须先通过密码验证（scrypt 加盐哈希 + 会话 cookie + 登录限速）。
- 手机端 UI：注入移动端适配 CSS。

---

## ⚠️ 安全警告（务必先读）

开启「远程访问」会把 Web GUI 绑定到 0.0.0.0（所有网卡），局域网内任何设备都能访问并操作你的 Harness（会话、文件、终端、凭据、代码执行等）。

开启「frpc 内网穿透」会把面板暴露到公网。强烈建议同时设置「面板密码」——本插件的密码门禁会对所有非本机访问（包括走 frp 隧道的访问）要求先登录。

- 仅在你信任的网络使用。
- 公网 / 不可信网络开启，等同于把完整控制权暴露给他人，请务必设置强密码。

---

## 功能特性

- 远程访问开关（默认关）：切换 0.0.0.0 与 127.0.0.1 绑定；关闭时还原全部修改。
- frpc 内网穿透（默认关，实时开关）：启动时自动从 GitHub Releases 下载 frpc（带 SHA256 校验，失败回退手动路径），生成 frpc.toml 并拉起；取消勾选立即停止。配置项：服务器地址/端口、token 验证、本机端口、远程端口。
- 面板密码门禁：非 127.0.0.1 来源必须先通过密码验证；密码只存加盐哈希，带登录限速（5 次/分/IP）与 7 天会话。
- 独立密码网关：frpc 隧道不直连 DSH，而是先经过一个本地密码网关（127.0.0.1 的「本机端口」，默认 3081），验证通过后才转发到 DSH 主端口 3080——这样公网隧道也需要密码，而本机 127.0.0.1 免密访问不受影响。
- 手机端 UI 调整（默认开）：注入移动端适配 CSS。
- 首次使用安全确认：阅读并确认风险后才解锁开关。
- crypto.randomUUID 补丁：修复局域网 HTTP 下 randomUUID 缺失。

## 截图

<table>
  <tr>
    <td align="center"><img src="assets/settings.png" alt="设置页" width="220"><br>设置页</td>
    <td align="center"><img src="assets/login-gate.png" alt="密码登录页" width="300"><br>密码登录页</td>
    <td align="center"><img src="assets/mobile-ui-1.png" alt="手机端 1" width="130"><br>手机端 1</td>
  </tr>
  <tr>
    <td align="center"><img src="assets/mobile-ui-2.png" alt="手机端 2" width="130"><br>手机端 2</td>
    <td align="center"><img src="assets/mobile-ui-3.png" alt="手机端 3" width="130"><br>手机端 3</td>
    <td></td>
  </tr>
</table>

---

## 安装

### 其他电脑（GitHub 发布版）

    dsh plugin --profile web add github:Franktolin/kosho-gate

### 本机开发（file: 依赖 + junction）

本地独立副本（自 chicheng-gate fork），通过 file: 依赖安装：

    "dependencies": {
      "kosho-gate": "file:../../../my-plugins/kosho-gate"
    }

在 profile 目录（`.dsh-data/profiles/web`）执行：

    CI=true pnpm install --no-frozen-lockfile

改代码：编辑 `my-plugins/kosho-gate/lib/index.js`（host）或 `lib/client.js`（前端），重启 dsh web 生效。

---

## 使用

1. 打开 Web GUI → 设置 → 侧栏选「kosho网关」。
2. 首次进入弹出安全确认，勾选「我已阅读并了解上述安全风险」→ 点「同意」。
3. 设置面板密码（推荐）：在「面板密码」卡片输入至少 8 位密码并保存。
4. 配置 frpc（如需公网访问）：在「内网穿透 (frpc)」卡片填服务器地址、端口、token、本机端口（默认 3081，不可用 3080）、远程端口。
5. 打开「远程访问」开关（局域网访问）和/或「启用 frpc」（公网访问）。
6. 重启 dsh web（远程访问和 frpc 配置改动需重启生效；frpc 的启用/停用开关本身是实时的）。
7. 访问：
   - 局域网：http://<本机IP>:3080
   - 公网（frpc 隧道）：http://<frps公网IP>:<远程端口> —— 首次会弹登录页，输入面板密码。

---

## 工作原理

- 主机侧（lib/index.js）：
  - 启动早期读取设置，提供 remoteAccess 服务（决定 webserver.host 与 connection.trustedHosts），并给 web-runtime 注入同一 trust 列表（供 /api 与 dsh-better-sidebar 等 fence 使用）。
  - 按开关应用/还原 4 处官方源码补丁（打补丁前自动备份 .kosho-gate.bak，关闭时还原）。
  - 面板密码门禁：包住 HTTP server 的 request/upgrade，非本机访问要求会话 cookie；提供 /kosho-gate/login、/kosho-gate/logout、/kosho-gate/password、/kosho-gate/status、/kosho-gate/restart 路由。
  - frpc 管理：自动下载/启动/停止 frpc（存放于 $DSH_HOME/frpc/，PID 记在 frpc.pid），按设置实时开关。
  - 独立密码网关：监听 127.0.0.1 的「本机端口」，反向代理（HTTP + WebSocket + SSE）到 DSH 主端口，复用同一套密码/会话。
- 客户端（lib/client.js）：在设置侧栏注册「kosho网关」分区，卡片式渲染远程访问 / 面板密码 / frpc / 手机端 UI，通过 settingsScope 读写设置。

设置命名空间：kosho-gate（写入 settings.yaml）：

    kosho-gate:
      consented: false
      remoteEnabled: false
      mobileUi: true
      frpcEnabled: false
      frpcServerAddr: ""
      frpcServerPort: 7000
      frpcAuthMethod: token
      frpcToken: ""
      frpcLocalPort: 3081
      frpcRemotePort: 3080
      frpcPath: ""
      panelPasswordHash: ""

（panelPasswordHash 为 secret，settings 里只存哈希，不存明文。）

---

## 常见问题

- 改了远程访问 / frpc 配置没生效：需要重启 dsh web（frpc 的「启用」开关本身是实时的，无需重启）。
- frps 上不显示端口 / 公网打不开：检查 frpc 状态（设置页有实时状态），常见原因是「远程端口」在 frps 上被占用，或 frps 的 vhostHTTPPort 占用了同一个端口。换一个空闲端口即可。
- 公网能打开但弹登录页：正常，走隧道必须输入面板密码；本机 127.0.0.1 不需要。
- 资源管理器 / 设置 403：确认远程访问已开、trustedHosts 已包含访问来源，并已重启。

---

## 维护备忘

### junction 软链接（改源码实时同步）

`node_modules/kosho-gate` 是一个目录软链接（junction），指向本源码目录 `my-plugins/kosho-gate`。所以直接改本目录的 `lib/*.js` 后重启 dsh web 即生效，无需手动同步。

**注意**：在 profile 目录跑 `pnpm install`（装新插件 / 更新依赖）会把 junction 替换回 pnpm 管理的硬链接副本，导致"改源码不生效"。恢复方法（在 profile 目录或任意位置执行）：

```powershell
Remove-Item "E:\code\dsh\.dsh-data\profiles\web\node_modules\kosho-gate" -Recurse -Force
cmd /c mklink /J "E:\code\dsh\.dsh-data\profiles\web\node_modules\kosho-gate" "E:\code\dsh\my-plugins\kosho-gate"
```

### 补丁与 dsh 升级

本插件启动时会对多个官方文件打源码补丁（放开远程设置、内测声明确认持久化、设置导航图标、手机端设置面板适配），首次打补丁前自动备份为 `.kosho-gate.bak`，关闭「远程访问」时还原（REMOTE/MOBILE 系列）。

- dsh 升级（重装 profile 依赖）会把这些官方文件覆盖回原版，补丁失效 → 远程设置重新锁死。重启 dsh web 会自动重打补丁，前提是官方文件结构没变。
- 若官方升级改了文件结构导致补丁字符串失配，补丁会静默失效（打不上），需对照新版文件更新 `lib/index.js` 里的 `REMOTE_PATCHES` / `MOBILE_PATCHES` / `ALWAYS_PATCHES` 的匹配串。
- **dsh 升级后**：先删除所有 `.kosho-gate.bak` 文件（`profiles/node_modules` 下 `dsh-client-connection/lib/index.js`、`dsh-client-ui-settings/lib/client.js`、`dsh-client-ui-settings-models/lib/client.js`、`dsh-client-ui-settings-general/lib/client.js` 等），让插件用新版文件重新备份，避免"关闭远程访问"时用旧备份降级官方文件。

### 相对原版 chicheng-gate 的改动

1. frpc 认证支持 `user` 模式（新增 `frpcAuthMethod: user` + `frpcUser`），适配 user 认证的服务商。
2. 新增自定义代理名 `frpcProxyName`（服务商指定的 proxy name）。
3. 新增传输协议选择 `frpcTransportProtocol`（tcp / websocket / kcp / quic），限速时切 websocket。
4. 隧道网关对文本响应加 gzip 压缩，大幅加速公网加载（前端资源 6.59MB → 约 1.4MB）。
5. 修复 `checkoutRoot()`：补丁目标从 `process.argv[1]` 推断的源码根，改为 `DSH_HOME/profiles`，否则补丁永远打不上、远程设置锁死。
6. 包名 chicheng-gate → kosho-gate；UI 名「赤橙网关」→「kosho网关」。
7. 隧道网关转发时把请求 **Host 改写为 `127.0.0.1:<DSH端口>` 并删除 Origin**：密码门禁验证通过后，上游 DSH 一律按本机信任处理，一次性放行 dsh 官方 RPC（settings/credentials）与 @linxin666 全家桶（git-graph / dsh-ssh / task-board）的 loopback-only 检查。**安全模型：过了面板密码 = 本机权限**。
8. 内测声明确认持久化：patch `dsh-client-ui-settings-models` 客户端 bundle，把欢迎步骤的持久化从 `connection.isLoopback ? "host" : "memory"` 强制为 `"host"`——公网（隧道）浏览器地址栏是公网 IP 被判定为非 loopback，原本确认只存在于进程内、每次打开都重弹「内测声明」，patch 后确认真正写入 Host 设置 `ui-onboarding.welcomeNoticeVersion`（只放开这一个字段）。
9. 手机端输入栏折叠：触屏窄容器（`@container (max-width: 560px)`）下，权限选择与模型选择收起为固定短文本「权限」「模型」，点击展开后菜单仍显示完整详情（aria-label/title 保留完整名称，无障碍信息不丢）。
10. dsh rc.6 及更早兼容：rc.6 的 `dsh-host-apiproxy` 用静态白名单 `WEB_SETTINGS_NAMESPACES` 决定哪些命名空间对 Web 客户端可见/可写（插件自注册暴露是 rc.7 才有的能力），kosho-gate 不在白名单 → 客户端 `settings.describe` 看不到它 → 设置页全部控件因 `consented` 读不到而禁用。新增 ALWAYS 补丁把 `kosho-gate` 注入白名单数组（rc.7+ 数组已删除 → 补丁自动 no-op）。

### 发布更新（改代码 → 推 GitHub）

1. 改 `my-plugins/kosho-gate/lib/*.js`，重启 dsh web 测试。
2. 测试通过后提交并推送：

    cd E:\code\dsh\my-plugins\kosho-gate
    git add -A
    git commit -m "描述改动"
    git push

3. 其他电脑更新：重新 `dsh plugin --profile web add github:Franktolin/kosho-gate`（或升级依赖）。

> git 已配置全局代理 `127.0.0.1:7892`，push 需代理在线；若代理离线，先 `git config --global --unset http.proxy` 和 `git config --global --unset https.proxy`。

---

## License

MIT
