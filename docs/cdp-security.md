# CDP 本机安全边界 / Local CDP Security Boundary

Codex Dream Skin 通过 Chromium DevTools Protocol（CDP）向官方 Codex 渲染器注入主题。它不修改官方安装包、`app.asar` 或代码签名，但启用主题时会扩大本机攻击面。

Codex Dream Skin injects themes into the official Codex renderer through Chromium DevTools Protocol (CDP). It does not modify the official package, `app.asar`, or code signature, but an active themed session increases the local attack surface.

## 必须知道的边界 / What the boundary means

- 调试服务只绑定 `127.0.0.1`，因此其他局域网设备不能直接连接。
- CDP 本身没有针对本机进程的身份认证。能在这台电脑上运行并访问回环地址的其他程序仍可尝试连接。
- 成功连接的程序可能检查渲染内容、在渲染器中执行 JavaScript、调用 DevTools 命令或操作界面；应把该会话视为本机特权调试会话。
- 项目对端口归属、WebSocket 地址、Browser ID 和目标渲染器的校验用于防止 Dream Skin 自己连接错误目标，**不能**阻止其他本机程序主动连接 Codex 的 CDP 端口。

- The listener binds to `127.0.0.1`, preventing direct access from other LAN devices.
- CDP does not authenticate local processes. Other software that can run on the computer and reach loopback can still attempt to connect.
- A successful client may inspect rendered content, execute JavaScript in the renderer, issue DevTools commands, or operate the UI. Treat the session as a locally privileged debugging session.
- Dream Skin validates listener ownership, WebSocket shape, Browser ID, and renderer targets to avoid attaching to the wrong endpoint. Those checks do **not** stop another local program from attaching to Codex.

## 风险窗口 / Exposure window

风险从 Codex 带 `--remote-debugging-*` 参数启动时开始，直到该进程退出。停止或暂停 injector 只会停止 Dream Skin 的重注入逻辑，不会撤销 Codex 已开放的调试端口。

The window starts when Codex launches with `--remote-debugging-*` flags and ends when that process exits. Stopping or pausing only the injector stops Dream Skin reinjection, but it does not revoke the debugging listener already opened by Codex.

## 建议操作 / Recommended operation

1. 只在需要皮肤时启动主题会话，并只运行可信的本机软件。
2. 不要在主题运行期间执行来源不明的安装包、脚本、浏览器扩展或自动化工具。
3. 使用完成后执行平台的完整 Restore：
   - macOS：运行桌面的 `Codex Dream Skin - Restore.command`，或执行 `macos/scripts/restore-dream-skin-macos.sh --restart-codex`。不带 `--restart-codex` 只移除当前样式，不会结束已开放的 CDP listener。
   - Windows：执行 `windows/scripts/restore-dream-skin.ps1`。
4. Restore 应关闭带调试参数的 Codex，并以不带 CDP 参数的普通模式重新打开。若 Restore 报错，请先关闭 Codex，再按平台文档排查；不要把“injector 已停止”当作端口已关闭的证明。

1. Start a themed session only when needed, and run only trusted local software while it is active.
2. Avoid untrusted installers, scripts, browser extensions, or automation tools during the session.
3. Run the platform's full Restore when finished:
   - macOS: use `Codex Dream Skin - Restore.command`, or run `macos/scripts/restore-dream-skin-macos.sh --restart-codex`. Without `--restart-codex`, the script removes the live style but does not end the existing CDP listener.
   - Windows: run `windows/scripts/restore-dream-skin.ps1`.
4. Restore should close the debug-enabled Codex process and reopen it without CDP flags. If Restore fails, close Codex and follow the platform troubleshooting guide; an exited injector alone is not proof that the port is closed.

## 已评估的缓解方式 / Mitigations considered

- **独立 `--user-data-dir`**：可把主题会话与常用 Codex 配置隔离，但不会给 CDP 增加认证，也不能阻止本机程序连接端口；它只能降低部分配置复用风险。
- **注入后立即关闭端口**：Chromium 不能在不结束当前调试进程的情况下可靠撤销启动时创建的 CDP listener；同时，页面重载和路由变化需要重新注入。因此当前架构选择在主题会话期间保持端口，并要求用 Restore 明确结束风险窗口。
- **回环绑定**：这是必要的网络隔离，可防止 LAN 直接访问，但不是本机进程隔离。

- **Separate `--user-data-dir`**: this can isolate a themed profile from the usual Codex profile, but it does not add CDP authentication or prevent local attachment. It only reduces some profile-reuse exposure.
- **Close CDP immediately after injection**: Chromium does not provide a reliable way to revoke the startup listener without ending the debug-enabled process, and reloads or route changes require reinjection. The current design therefore keeps CDP active for the themed session and uses Restore to end the window explicitly.
- **Loopback binding**: this is necessary network isolation and blocks direct LAN access, but it is not local-process isolation.

## 不在范围内 / Out of scope

Dream Skin 不声称能防御已经在本机用户会话中运行的恶意软件。若你的威胁模型包含不可信本机进程，请不要启用 CDP 主题会话。

Dream Skin does not claim to defend against malware already running in the local user session. If your threat model includes untrusted local processes, do not enable a CDP-themed session.
