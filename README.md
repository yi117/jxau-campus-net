# 江农校园网守卫（jxau-campus-net）

江西农业大学校园网（锐捷 ePortal）的自动认证与断线自愈脚本。

开机不用再点认证页，掉线了自己连回来，会话到期前会按要求保活。

> **适用范围**：本校 SSID `JXAU`、认证服务器 `10.255.240.8` 的 ePortal 认证。
> 其他学校、其他厂商（华为 / 深澜 / Dr.COM 等）的认证本脚本不适用。

## 为什么需要它

学校的认证是 ePortal。认证成功后，服务器返回一个 `userIndex` 作为这条会话的编号，
会话空闲一段时间就会被认证设备踢掉。中间还有两个坑：

1. **认证设备的响应经常超时**，第一次登录拿不到 `userIndex`，会话只活两分钟；
2. **关机前的旧会话不会马上释放**，开机会被判 "Too many access"，得等它超时才能登进去。

脚本做的就是把这些事接管掉：登录、保活、掉线重连、清残留会话。

## 功能

- 开机自动登录，桌面不弹任何窗口
- 每 30 秒发一次保活（`keepalive`），防止空闲被踢
- 掉线自动重连；遇到 `Too many access` 先注销残留会话再试
- 登录响应超时也能救：按 `userIndex` 的生成规则自己拼一个，先保活接管会话
- 重连预算 5 分钟：连着试 5 分钟还不行就停手，等你手动登录，不再骚扰认证设备
- 可选：关掉 Windows "连上需要登录的网络就自动打开浏览器"
- 可选：关掉 Wi-Fi 网卡的省电断开

## 文件说明

| 文件 | 作用 |
|---|---|
| `install.cmd` / `install.ps1` | 录入账号、注册计划任务（UAC 一次） |
| `guard.ps1` | 守护进程本体 |
| `launch.vbs` | 无窗口启动守护进程（wscript 没有控制台） |
| `status.cmd` / `status.ps1` | 看当前状态、日志 |
| `selftest.cmd` | 主动注销会话，验证"能不能自己登回来" |
| `tune-wifi.cmd` / `tune-wifi.ps1` | 关网卡省电断开（可 `-Revert`） |
| `no-portal-popup.cmd` / `.ps1` | 关掉开机自动弹浏览器（可 `-Revert`） |
| `uninstall.cmd` / `uninstall.ps1` | 卸载并还原 |

运行时数据（凭据、日志、状态）都在 `%LOCALAPPDATA%\CampusNetGuard\`，不在仓库里。

## 快速开始

1. 双击 **`install.cmd`**，输入学号和密码（输两次确认），UAC 点允许
2. 双击 **`status.cmd`** 确认状态
3. 建议再跑一次 **`tune-wifi.cmd`**（关网卡省电）和 **`no-portal-popup.cmd`**（关开机弹浏览器）

装完可以把认证页从收藏夹删了。

## 看状态

双击 `status.cmd`，重点两行：

```
认证：已登录 userIndex=...
今日重登：N 次
```

`userIndex` 不为空，说明会话归它管了。

手动跑一轮：在本目录执行

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\guard.ps1 -Once
```

验证掉线自愈（会断 5~10 秒）：双击 `selftest.cmd`。

## 工作原理

```
每 30 秒：
  GET  http://www.msftconnecttest.com/connecttest.txt
       204/200  → 在线 → 到点发 keepalive
       302      → 认证页 → 取 Location 里的 queryString → 登录
       无响应   → 门户端口也连不上 → 判定链路问题，限频重连 Wi-Fi
```

登录请求与浏览器完全一致：

```
POST http://10.255.240.8/eportal/InterFace.do?method=login
Content-Type: application/x-www-form-urlencoded

userId=<双重URL编码的账号>&password=<双重URL编码的密码>&service=
&queryString=<双重URL编码的跳转参数>&operatorPwd=&operatorUserId=
&validcode=&passwordEncrypt=false
```

返回 `{"result":"success","userIndex":"...","keepaliveInterval":"..."}`。

`userIndex` 不是随机串，是 `<nasip>_<本机IP>_<账号>` 的十六进制，例如：

```
a1b2c3d4e5f60718_192.0.2.10_2026000000
```

所以登录响应超时时，脚本能自己拼一个去发保活，先把会话接管下来。

失败分类：

| 情况 | 处理 |
|---|---|
| `Too many access` / 已在线 | 按规则拼出 userIndex，调 `logout` 注销残留会话，隔 2 秒重登 |
| 密码/账号错误 | 停止重试，日志写原文 |
| 要验证码 | 停止重试，提示手动登录 |
| 其他 | 退避重试 5/15/30/60/120 秒；10 分钟内超过 6 次暂停 10 分钟 |

## 重连预算

从这一轮第一次发现认证页开始计时，总共 5 分钟：

- 用完还没连上 → 停止自动重连，只探测不登录，等你手动处理
- 手动登上去之后，它重新接管并继续保活
- 连续稳定在线满 5 分钟，才恢复自动重连能力

避免"断了就连、连上又断"无限循环。时长可在
`%LOCALAPPDATA%\CampusNetGuard\config.json` 里改 `reconnectWindowSec`。

## 卸载

双击 `uninstall.cmd`：删计划任务、还原网卡设置、清理运行数据。

只关掉自动弹浏览器的话：`no-portal-popup.cmd -Revert`。

## 安全与边界

- 账号密码用 Windows DPAPI 加密存在本机，明文不落盘，不上传任何地方
- 只把认证请求发给学校的认证服务器；换到手机热点、家用 Wi-Fi 会自动静默
- 只用**你自己的账号**，不提供共享账号、批量尝试之类的功能
- 校园网使用遵循学校网络管理规定，使用前请自行确认允许范围
- 软件按现状提供，不承诺任何效果；因使用产生的后果由使用者自负

## 许可

MIT
