# LenoWorkBuddy — WorkBuddy 自动签到（Leno 版）

用 **Leno** 每日签到工具：自动领取每日签到积分，外加成长中心全套
（领旅行礼物 / 派 Buddy / 任务接单与领奖 / 断登补登 / 连登兑换 / 开盲盒 / 能量开 Buddy）。

- 本实现：`Leno` 多模块工程，CLI 与将来的 GUI 共用同一份核心逻辑
- 只读取**你自己本机**的登录态，不内置任何密钥，可安全分享

---

## 前置条件

- ✅ 已安装并**登录过 WorkBuddy 桌面端**（登录后自动写出凭据文件，脚本靠它鉴权）
  - Linux 没有桌面端，登录过 CodeBuddy CLI 即可（脚本会自动探测它的凭据）
- ✅ Leno 解释器（本仓库按 `D:\CLeno\Leno\build\leno.exe` 开发验证；其它版本请以能跑通为准）
- ⬜ 可选：打包成单文件 exe 后，目标机器**不需要**装 Leno

---

## 命令

```bat
leno wb_sign.leno                :: 签到 + 成长中心（默认动作，等同 auto）
leno wb_sign.leno auto           :: 同上
leno wb_sign.leno silent         :: 同 auto，但结果写日志文件而非 stdout（配合无窗口定时任务）
leno wb_sign.leno growth         :: 仅成长中心（不签到）
leno wb_sign.leno silent-poll    :: 轮询：补签（未签才签）+ 成长中心，空跑不写日志
leno wb_sign.leno silent-growth  :: silent-poll 的旧名，行为完全相同
leno wb_sign.leno status         :: 仅查签到状态（调试，原始返回）
leno wb_sign.leno claim          :: 仅领取签到（调试，幂等）
leno wb_sign.leno all            :: 查签到状态 + 领取（调试）
leno wb_sign.leno doctor         :: 离线检查凭据格式与运行时能力，不解密、不联网
```

> 解释器自己的旗标（`-p` / `-c` / `--no-cache` …）要写在**脚本路径之前**，
> 脚本路径之后的一切参数都原样交给脚本。

退出码：`0` 成功（含"今日已签"与"非签到季"）／`1` 认证·网络·需要关注的失败／`2` 用法与环境错误。

### 图形界面（窗口 + 托盘）

```bat
leno wb_gui.leno            :: 开窗（右上角托盘也一起起来）
leno wb_gui.leno --tray     :: 启动即只留托盘（开机自启写的就是这条）
```

界面三张卡片 + 底部状态行：

| 区域 | 内容 |
|---|---|
| 今日 | 今日状态徽标（已签到 / 未签到 / 非签到季 / 网络不可达…）、连签天数、累计积分、能量 |
| 操作 | 立即签到（签到 + 成长中心）· 查状态 · 领成长奖励 · 检查凭据（离线自检） |
| 自动化 | 开机自启动 · **定时签到**（HH:MM 可编辑）· **轮询成长中心**（分钟可编辑）· 关窗收进托盘 |
| 底部 | 最近一次动作的汇报文案 + 定时/轮询的"下次什么时候"提示 |

托盘：**右键 = 菜单**（显示/隐藏窗口 · 立即签到 · 开机自启动 ✓ · 退出），**左键 = 显示/隐藏窗口**；
悬停提示跟随今日状态。关窗行为由「关窗收进托盘」决定（开 ⇒ 收进托盘继续跑定时；关 ⇒ 直接退出）。

**定时为什么放在进程里**：GUI 常驻托盘 ⇒ 它自己就是定时器。这样不必去建系统计划任务
（也就没有"任务找不到解释器 / 工作目录不对 / 无窗口看不到输出"那一串坑），到点必触发。
代价是**程序没在跑就不会自动签**——所以配了「开机自启动」，且启动时若已过点会立刻补一次。
它与 CLI 的 `silent` / `silent-poll` **不冲突**：签到接口幂等，两边都跑只会多一次 ALREADY 查询。

配置与状态落在 `%APPDATA%\LenoWorkBuddy\`（`wb_gui.json` / `wb_state.json`；用 `Sys.configDir()`，跨平台 ✓）。
界面上改的每一项都会即时落盘；坏掉的配置文件一律回落默认值，不会把程序带崩。

输出是**一行 JSON**，`report` 字段是给人看的话：

```json
{"result":"CLAIMED","report":"成功领取 100 积分（连续 7 天，累计 700 积分）","growth":"派 Buddy 去咖啡馆（2 小时后回）；…","needs_attention":false}
```

---

## 定时（两种模式）

### 模式 A · AI 自动化（跨平台，消耗模型调用）

让 WorkBuddy 的自动化每天 00:05 跑一次命令，模型只负责"跑命令 + 汇报"：

```text
运行 leno <绝对路径>\wb_sign.leno auto，
把命令输出 JSON 里 report 字段的内容一句话汇报给我；
若 needs_attention 为 true 或退出码非 0，额外提醒我处理。
```

### 模式 B · 系统级静默（推荐：零模型调用、零窗口）

先打一个**无窗口**的单文件 exe：

```bat
leno -p wb_sign.leno -o dist --no-console
```

再建计划任务（Windows）：

```powershell
powershell -ExecutionPolicy Bypass -File .\install-windows.ps1
```

它会建两个任务：

| 任务 | 频率 | 干什么 |
|---|---|---|
| `WorkBuddyAutoSignin` | 每天 00:05 | 签到 + 成长中心，静默写 `signin.log` |
| `WorkBuddyGrowthPoll` | 每 4 小时 | 补签（未签才签）+ 成长中心全套 |

> 脚本丢了也不怕，核心就这几句（`$Exe` 换成 `dist\wb_sign.exe` 或 `leno.exe`）：
>
> ```powershell
> $act = New-ScheduledTaskAction -Execute "D:\path\dist\wb_sign.exe" -Argument "silent"
> $trg = New-ScheduledTaskTrigger -Daily -At "00:05"
> $set = New-ScheduledTaskSettingsSet -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Minutes 10)
> Register-ScheduledTask -TaskName "WorkBuddyAutoSignin" -Action $act -Trigger $trg -Settings $set -Force
>
> $act2 = New-ScheduledTaskAction -Execute "D:\path\dist\wb_sign.exe" -Argument "silent-poll"
> $trg2 = New-ScheduledTaskTrigger -Once -At "01:00" -RepetitionInterval (New-TimeSpan -Hours 4) `
>           -RepetitionDuration (New-TimeSpan -Days 3650)
> $set2 = New-ScheduledTaskSettingsSet -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Minutes 5)
> Register-ScheduledTask -TaskName "WorkBuddyGrowthPoll" -Action $act2 -Trigger $trg2 -Settings $set2 -Force
> ```

两个任务都零模型调用、无窗口、关机错过后下次开机自动补跑（`StartWhenAvailable`）。

**为什么要两个任务**：签到一天一次就够，成长中心却不是 —— Buddy 旅行 1~4 小时就带礼物回来，
礼物得手动领；只靠 00:05 那一次，礼物会压到第二天，当天唯一的派出名额也会白浪费。
轮询同时兼任**签到兜底**：每轮先查一次状态，未签就补（接口幂等，已签只是一个查询请求）。
空跑（已签 / Buddy 在路上 / 名额已用完）默认**不写日志**，免得把有价值的记录淹没；
想逐轮查看就设 `WORKBUDDY_GROWTH_LOG_EMPTY=1`。

---

## 工作原理

```
① 定位凭据   %LOCALAPPDATA%\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info
             （macOS / Linux 路径见 wb_auth.find_auth_file；可用 WORKBUDDY_AUTH_FILE 覆盖）
② 解密 token 明文直接用；新版 `$wbEncrypted` 信封 → 拉起客户端自己的运行时解密（见下）
③ 查状态     POST {endpoint}/v2/billing/meter/checkin-activity-status
④ 领取       POST {endpoint}/v2/billing/meter/daily-checkin   （幂等：先查，未签才领）
⑤ 成长中心   /v2/activity/growth/…  旅行 → 任务 → 补登 → 连登兑换 → 盲盒 → Buddy 盲盒 → 展示值
⑥ 输出       一行 JSON；silent 模式追加写 signin.log
```

### 加密凭据是怎么解的（两条路，默认走 ②）

凭据文件里的 `accessToken` 是加密信封（`{$wbEncrypted:1, envelope:"<base64>"}`），
内层是 `{suite:1, keyId, nonce, authTag, ciphertext}`：

```
密钥   = sha256( atRestSecretKey 这个字符串的 UTF-8 字节 )      ← 哈希的是"字符串本身"
keyId  = sha256(密钥).hex 前 16 字符
AAD    = WB-AAD\0 | 0x01 | lp('WBEV1') | lp('sym-v1') | 0,0,0,1 | lp(keyId) | 2,0,0
算法   = AES-256-GCM，nonce 12 字节，标签 16 字节
```

| 路 | 条件 | 说明 |
|---|---|---|
| ① **纯 Leno** | 有密钥（`WORKBUDDY_AT_REST_KEY`） | 全流程 Leno：`LenoCrypto` 的 AES-256-GCM + 本项目 `wb_envelope.leno`。**不启任何子进程**，也不依赖客户端运行时 |
| ② 客户端运行时 | 没有密钥（默认） | 把内嵌助手交给 `WorkBuddy.exe`（`ELECTRON_RUN_AS_NODE=1`）跑一次，借它的原生存储接口拿密钥、顺手解开 |

**为什么默认不是 ①**：密钥（44 字符 base64 / 32 字节）只由客户端的原生绑定
`electron_browser_workbuddy_storage.loggerGet()` 提供，实现编译在 `WorkBuddy.exe` 里
（`electron_api_workbuddy_storage.cc`），**不在任何可读落点**上 —— 已逐条实测排除：
客户端 exe/asar 内无该值（每机生成）、`~/.workbuddy` 44,960 个文件（<256KB）与
`%APPDATA%\CodeBuddy CN` 里无明文字符串/原始 32 字节/DPAPI 块、不是 `Local State` 的
os_crypt 密钥、凭据管理器与 `HKCU\Software\Tencent` 里也没有。
⇒ 想彻底不碰客户端，还得把那个原生存储的位置与格式逆出来（未做）。

**①路已用真实凭据验证过**：同一份信封下，纯 Leno 解出的 token 与客户端 JS 解出的
**逐字节一致（1365 字节）**，`doctor` 报 `credential_mode: pure-leno`，`status` 拿到 HTTP 200 ——
且全程没有启动过客户端。

性能：纯 Leno 解一次 1365 字节的信封约 **27 ms**（其中 GHASH 约 10 ms），对"一次运行解一次"完全够用。

### 加密凭据的机制差异

密钥（`atRestSecretKey`）只存在于客户端的**原生存储接口**里，所以必须让客户端自己来解。
管道方案（JS 喂 stdin、结果从 stdout 出）在 Leno 里行不通：`_exec` 只有"合并 stdout+stderr"
一个出口、**没有 stdin**，于是这里改成：

```
助手源码**内嵌**在 wb_auth.leno 里 + 环境变量协议
（WB_HELPER_JS 传代码本体 / WB_HELPER_AUTH 凭据路径 / WB_HELPER_OUT 结果文件 / WB_HELPER_OP 探针或解密）
                     ↓  _spawn  (CreateProcessW，不经 cmd)
        WorkBuddy.exe 以 ELECTRON_RUN_AS_NODE=1 运行 → 结果落盘 → 读完即删
```

> 助手不再是独立文件：源码内嵌成 `HELPER_JS` 原始字符串，命令行只传一句
> `-e "eval(process.env.WB_HELPER_JS)"`（代码本体走环境变量）。这样打包时
> **不存在"资源没带上"这个失败面** —— 单文件 exe 与脚本模式是同一份代码。

顺带避开了"路径带空格时 `cmd` 引号被吞"的坑（`_spawn` 会把 exe 与参数分开传）。
密钥只在那个短命子进程的内存里出现，随即置零；不落盘、不缓存、不回传。

### 行为口径

- **结果码逐字一致**：`CLAIMED / ALREADY / INACTIVE / NETWORK / TIMEOUT / UNKNOWN / ERROR /
  AUTH_REJECTED / FORBIDDEN / NO_AUTH / NO_SESSION / AUTH_ERROR`，以及 `report` 文案
  （`auto` / `growth` 的输出字段完全一致）。
- **重试口径一致**：只重试"本机网络没就绪"与 5xx（网络 5/15/30/60/90 秒、5xx 3/10 秒，
  按失败类型**分别**记退避进度）；写操作（抽奖/领奖/补登/兑换）一律不重试，
  只有两个签到接口例外（状态只读、领取幂等）。
- **时间预算一致**：签到类默认 420 秒（上限 540）、轮询类 180 秒（上限 240），
  非法值一律夹到安全区间并附 `config_warning`，绝不因为一个坏环境变量把当天日志丢掉。
- **"常态"不记失败**：旅行名额已用完、日期无需补登、连登档位未解锁、没有抽奖次数
  —— 这些按正常业务状态处理，否则轮询会每轮强制落盘。
- **"成功"必须由服务端确认**：领取接口返回什么都**不算**签上了 —— 正文是 `null`、
  或响应里带 `credit` 字段（哪怕是 0），都可能是服务端压根没发积分。唯一判据是
  **再查一次状态**：`today_checked_in == true` 才算数（报 `CLAIMED` / `ALREADY`）。
  复核不了、或复核说没签 ⇒ 如实报 `UNKNOWN` 并置 `needs_attention`，**绝不编成功**。
  （例外：服务端明确回 `code 10001`"今天已签到"时采信它，但复核对不上会额外标
  `⚠ 复核显示今日未签到，请到客户端核实`。）
- **跨天不沿用旧值**：GUI 落盘的"今日状态"带日期（`today_status_date`）。日期不是
  本地今天就一律显示 `今日：待查询（上次 2026-10-14：已签到）`，徽标回 `待查询`
  —— 不再拿昨天的"已签到"冒充今天。

差异（有意为之）：

| 项 | 本实现 | 原因 |
|---|---|---|
| JSON 输出 | 紧凑一行 | 机器可读，体积更小 |
| 凭据助手 I/O | 环境变量 + 结果落盘 | Leno 无 stdin 通道 |
| 端点点位 | 可用 `WORKBUDDY_API_BASE` 覆盖 | 离线 stub 端到端回归（对标 LenoTrae 的 `TRAE_API_BASE`） |

---

## 配置（环境变量）

| 变量 | 作用 |
|---|---|
| `WORKBUDDY_AUTH_FILE` | 自动探测失败时，手动指定凭据文件路径 |
| `WORKBUDDY_AT_REST_KEY` | **at-rest 密钥**（44 字符 base64）。设了就走**纯 Leno** 解密、完全不碰客户端运行时 |
| `WORKBUDDY_EXE` | 指定与加密凭据对应的客户端**可执行文件**（路径错误直接报告，不静默回退） |
| `WORKBUDDY_SIGNIN_LOG` | silent 模式日志路径（默认脚本/exe 同目录的 `signin.log`，不可写时自动回退到同目录） |
| `WORKBUDDY_BUDGET_SECONDS` | 单次运行的网络时间预算。签到类默认 420、上限 540；轮询类默认 180、上限 240 |
| `WORKBUDDY_GROWTH_LOG_EMPTY` | 设为 `1`/`true`/`yes`/`on` 时，轮询连空跑也写日志 |
| `WORKBUDDY_API_BASE` | **测试钩子**：覆盖端点（跳过 https 校验），配合 `test/wb_stub.leno` 做离线回归 |

---

## 排错

| 现象 | 处理 |
|---|---|
| `NO_AUTH / 未找到登录凭据` | 先登录一次 WorkBuddy 桌面端；或设置 `WORKBUDDY_AUTH_FILE`（输出里的 `looked_in` 是实际检查过的路径） |
| `NO_SESSION` | 本地缺少登录会话或必要字段，先登录客户端 |
| `AUTH_ERROR / INVALID_FORMAT` | 凭据结构或令牌格式无效，检查凭据来源与客户端版本 |
| `AUTH_ERROR / UNSUPPORTED_ENVELOPE` | 加密格式尚不支持：**重新登录不会解决**，要更新脚本 |
| `AUTH_ERROR / RUNTIME_NOT_FOUND`、`INVALID_RUNTIME_PATH` | 用 `WORKBUDDY_EXE` 指定客户端可执行文件 |
| `AUTH_ERROR / KEY_MISMATCH`、`DECRYPT_FAILED` | 凭据与所选客户端不是一对（多版本安装时用 `WORKBUDDY_EXE` 明确选） |
| `AUTH_ERROR / HELPER_TIMEOUT`、`HELPER_PROTOCOL` | 凭据助手超时/返回无效结果。**先看包是不是比源码旧**：助手已内嵌进 `wb_auth.leno`，早于该改动打的 exe 还会去找独立的 `wb_helper.js` ⇒ 重新打包即可。脚本模式下可用 `doctor` 区分（它同样走助手链路，能过就说明代码没问题） |
| `AUTH_REJECTED / HTTP 401` | 服务端拒绝认证：检查客户端登录状态、凭据对应的服务地址 |
| `FORBIDDEN / HTTP 403` | 服务端拒绝操作，检查账号权限或活动条件（**不能**据此判断登录过期） |
| `INACTIVE / 签到活动未开启` | 非签到季，正常 |
| `NETWORK / 网络不可达` | 断网或服务端不可用，**非**登录问题；已按分钟级退避重试，仍失败等下一次运行 |
| `TIMEOUT / 已达本次运行时间预算` | 已领到部分照常记录，剩余项下次再领 |
| `UNKNOWN / 未确认`（退出码 1） | 领取请求发出去了，但服务端**没确认**（复核说未签，或复核本身失败）。这不等于签好了：去客户端看一眼，必要时再跑一次（接口幂等，多跑只是多一次查询） |
| GUI 顶部显示`今日：待查询（上次 …）` | 正常：那行是**上次**查询的结论，跨天不再沿用。点「查状态」或等自动拍子刷新成今天 |
| 服务端说已签、但签到记录里没有本地今天 | 文案会提示核实并置 `needs_attention`（多半是服务端还没跨天重置）。**以客户端显示为准**，别只看脚本 |
| 00:05 那次失败但连签没断 | 正常：轮询任务会先查状态、未签就补（日志里该轮带 `"trigger":"poll"`） |
| `signin.log` 里查不到轮询记录 | 正常：空跑不落盘，`WORKBUDDY_GROWTH_LOG_EMPTY=1` 可打开 |
| 调试原始返回 | `leno wb_sign.leno status` |

---

## 安全与隐私

- 只**读**本机登录态文件，不改写；不含、不内嵌、不传输任何第三方密钥
- **永不打印 token**：token 一进 `core.add_sensitive` 就会被 `[REDACTED]` 替换掉
- 加密路径的密钥只在短生命周期的客户端子进程内存里使用，用完即置零；不写临时密钥文件
- 纯 Leno 路（`WORKBUDDY_AT_REST_KEY`）**不落盘、不派生子进程**：密钥只在本进程内存里用到解密结束；
  密钥由使用者自己保管，本工具不生成、不持久化、不上传
- Fork 与分享都只作用于**你自己**的登录态

---

## 开发者

### 目录结构（单一事实来源：CLI / 将来的 GUI 共用同一份核心）

| 文件 | 职责 |
|---|---|
| `wb_core.leno` | 基础设施：时间预算、HTTP 重试、`dig` 信封取值、标量转换、输出与脱敏 |
| `wb_auth.leno` | 凭据：定位、校验、两条解密路（纯 Leno / 客户端运行时）、请求头、失败原因人话化、doctor；**内嵌凭据助手源码**（`HELPER_JS`，经 `WB_HELPER_JS` 交给客户端运行时） |
| `wb_envelope.leno` | `sym-v1` 信封的**纯 Leno** 解析与解密（密钥派生、AAD 布局、AES-256-GCM），依赖 `LenoCrypto` |
| `wb_checkin.leno` | 签到：查状态 → 未签才领 → 复取状态 → 汇报（幂等口径都在这里） |
| `wb_growth.leno` | 成长中心七段；写操作不重试、"常态"不记失败的口径都在这里 |
| `wb_sign.leno` | CLI 入口：参数、退出码、silent 落盘、轮询空跑判据 |
| `wb_gui.leno` | 图形界面：窗口、三张卡片、开关/输入框接线、主循环 |
| `wb_tray.leno` | 托盘 + 自绘菜单 + 开机自启动（系统集成层） |
| `wb_auto.leno` | 进程内定时判定 + 配置/状态落盘（纯函数，可离线断言） |
| `images/` | 托盘与菜单图标（随包资源，见 `resource.toml`） |
| `test/` | 离线回归（见下） |

### 测试

```bat
leno test\test_wb_units.leno      :: 纯函数自测（预算夹取 / token 字符集 / 判据 / 日历 / ETA）
leno test\test_wb_flow.leno       :: 端到端离线回归（自起 stub 服务端，真 HTTP）
leno test\test_wb_envelope.leno   :: 纯 Leno 解 sym-v1 信封（密钥派生 / AAD 布局 / 篡改拒绝）
leno test\test_wb_auto.leno       :: 定时判定与配置/状态落盘（时间口径 / 跨天 / 间隔下限 / 坏配置）
leno test\test_wb_gui.leno        :: GUI 无头自检（下面那个包装：建窗口 → 逐项断言 → 退出）

:: 密码学侧（Leno 仓库）：FIPS-197 单块 + NIST GCM 用例 + 应用形态向量
leno <Leno>\build\leno_module\LenoCrypto\test\test_crypto_vectors.leno
```

GUI 的两个无头模式（都不需要人看屏幕，可直接进 CI）：

```bat
:: ① 冒烟：建窗口 + 跑 N 帧后按退出码退出（验证控件构造、主循环、渲染回调；不再打印摘要）
set SDL_VIDEODRIVER=dummy & set WORKBUDDY_GUI_AUTO=60 & set WORKBUDDY_GUI_NO_NET=1
leno wb_gui.leno

:: ② 自检：逐项跑动作并断言状态变化（默认只用临时目录，不碰真实配置）
set WORKBUDDY_GUI_SELFTEST=1 & leno wb_gui.leno
```

`test\wb_stub.leno` 是一个只监听 127.0.0.1 的迷你 HTTP 服务端，用 `WB_STUB_MODE`
切场景（`normal` / `already` / `stale_dates` / `inactive` / `claim400` / `null_body` /
`null_body_signed` / `claim_zerocredit` / `claim_unrecorded` / `unauthorized` / `flaky` /
`server_error`），把"未签→领取""服务端判已签""**看起来成功但服务端没记上**""401"
"5xx 重试""低预算不空等"这些**平时撞不上**的路径全跑穿。
当前：单测 86 项 + 流程 56 项 + 信封 9 项 + 定时/落盘 42 项 + GUI 自检 11 项，全绿。

### 打包

```bat
leno -p wb_sign.leno -o dist            :: 单文件（内嵌 libcurl + images/；助手源码本就在代码里）
leno -p wb_sign.leno -o dist --no-console  :: 再免掉控制台窗口（定时任务用）
```

---

## 免责声明

本项目为**非官方**工具，与腾讯或 WorkBuddy 无任何隶属关系。签到接口系从桌面端逆向所得，
服务器改一版就可能失效。使用风险自负；请遵守相关服务条款。
