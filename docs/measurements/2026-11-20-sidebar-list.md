# 实测记录：侧栏 bridge 的 `list`（DSH NEXT 2.0.24-next，2026-11-20）

环境：macOS，DSH Desktop `dsh-desktop-next` 2.0.24-next（解包布局 `Resources/app/`，
`Resources/app.asar.before-bridge` 存在），插件 0.5.8 + 本 fork，`browser.channel = "sidebar"`，
bridge 协议 3。

## A 组：上游 0.5.8 的 bridge（未修改）

```
{"op":"version"}                                  -> {"ok":true,"loadedAt":"2026-11-20T05:08:45.765Z","protocol":3}
{"op":"list"}                                      -> {"ok":false,"error":"Script failed to execute, this normally means an error was thrown. ..."}
{"op":"list","owner":"session-probe"}              -> {"ok":false, 同上}
{"op":"list","owner":"aa_47343d90216aef14_sess_..."} -> {"ok":false, 同上}
```

三个不同的 owner（空、带 `session-` 前缀、`aa_…_sess_…`）全部失败，说明失败与 owner 的**形状**无关：
对外壳窗口的 `executeJavaScript` 本身返回 "Script failed to execute"，`list` 的两处渲染器询问
（身份、面板根）把整个报告拖成错误。

## B 组：本 fork（同一台机器、同一个外壳、同一个会话 id）

```
{"op":"version"}                                  -> {"ok":true,"loadedAt":"2026-11-20T06:25:13.156Z","protocol":3}
{"op":"list"}                                      -> {"ok":true,"protocol":3,"guests":[外壳窗口],"sidebar":[],"frame":"none"}
{"op":"list","owner":"session-probe"}              -> {"ok":true, ...}
{"op":"list","owner":"aa_47343d90216aef14_sess_..."} -> {"ok":true, ...}
```

`sidebar` 为空是因为宿主的 `tool-browser` 层未挂载：没有任何会话去认领页面，
账本里因此没有页面可列——这与“渲染器答不上来”是两回事，也正是修复要区分的东西。
`frame:"none"` 表示侧栏此刻没有浏览器面板（面板是打开时才创建的）。

## 修复位置

- 提交 `8e48e35`：身份询问与面板探测合并为一次脚本；答不上来时 `list` 退回所有权账本，
  `frame` 退回 debugger 探测；驱动/关闭仍要求渲染器证明。
- 提交 `73b60ce`：会话 ID 形状由外壳决定（`namesConversation`），bridge 6 处 / client 3 处。
- 提交 `0d9ffc6`：安装器多份安装的取舍（本机 `/Applications` 同时有 `DeepSeek Harness.app`
  与 `DSH NEXT.app`，实测解析到正在运行的 `DSH NEXT.app`）。

## 未覆盖

- 侧栏**页面**级实测（`dsh_browser_open` 落进侧栏、不弹独立窗口）需要宿主 `tool-browser`
  挂载且不与已有浏览器工具重名；本机没有 `~/.dsh/logs/dsh-builtin-browser-host.log`，
  宿主进程从未在此运行，故无法取得该证据。
