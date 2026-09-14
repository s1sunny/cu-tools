# cu

Python 异步脚本，仅供编程学习与技术交流。示例涵盖 `httpx` 并发请求、Cookie 会话管理、多账号调度，以及常见加解密与签名流程的实现方式。

> 请勿用于商业用途或任何违反服务条款的场景；使用风险由使用者自行承担。

## 项目结构

```text
cu.py
├── 工具函数 ........................ 响应解析、异常格式化、Cookie 处理
├── Unicom .......................... 核心业务类（请求 / 登录 / 各任务模块）
└── main ............................ 多账号并发入口
```

## 任务模块

登录后按顺序执行（单任务失败不影响其余任务）：

| 标签 | 模块 | 备注 |
|------|------|------|
| 权益超市 | `market_task` | |
| 签到区 | `sign_task` | |
| 新疆联通 | `xj_task` | 按归属地跳过 |
| 联通祝福 | `ltzf_task` | |
| 安全管家 | `sec_task` | |
| 通通乡村 | `farm_task` | |
| 商都福利 | `shangdu_task` | 按归属地跳过 |
| 云手机积分 | `uphone_task` | 见下方说明 |
| 沃阅读积分 | `woread_task` | 每账号自动换票 |
| 校园季 | `campus_task` | 上传 / 转存 / AI 对话 / 抽奖，需云盘 token |
| 上传大比拼 | `cloud_battle_task` | 上传冲榜 / 每日上传得抽奖，需云盘 token |

### 云手机（`uphone_task`）

在手厅 `ecs_token` 基础上换票进入云手机活动体系，顺序为：

1. **SSO**：`getTicketByNative` → `getTokenByTicket` 得到 `cpToken`
2. **打卡挑战赛**（CLD13 / `h5forphone`）：复用 `cpToken` 作 `accesstoken`；`querySignInList` → `signIn` → 有订单则 `raffleSignIn`；补领 `signInRightList` 中 `state=1` 奖品（需 QQ/微信的 type=2 跳过）
3. **活动登录** + **积分签到**（`Points_Sign_2507`）
4. **赚积分任务**（`Points_Obtain_2507`）：可 API 完成的任务自动上报并领取；讨论区、看广告等需真机行为的任务跳过
5. **积分十连**：余额 ≥ 阈值且当日仍有次数时执行（默认满 100 分）
6. **夏日刮一刮**（`HD2026062200218`）：仅领 `2508-01` 换 1 次次数，有次数则抽完

### 校园季（`campus_task`）

1. **激活**后先查任务状态：已满的任务自动跳过，避免重复副作用
2. **上传**占位文件 + **AI 学习助手对话**（SSE 流式，独立新会话）
3. **转存教育/娱乐内容**：依次尝试校园 tab 候选 → 芒果TV 内容源
   （需芒果会员，无权益秒退）→ mbh 影视/教育频道（不限会员），
   内容池耗尽自动切换，已转存过的内容自动跳过
4. **自动抽奖**（次数本地硬上限保护）

### 上传大比拼（`cloud_battle_task`）

与校园季同源的 panservice 签名 / upload2C 上传协议，但页面、活动 ID、上传域名独立：

1. 进入活动页经 `openPlatLineNew.htm` 跳转拿带 ticket 的 Referer（lottery-times 必需）
2. 校验 `activity-status` 与冲榜开启状态；未开启则用归属省自动开启
3. 无抽奖次数时上传 1 个占位文件，轮询次数到账
4. **自动抽奖**（每日 1 次，首次必中）；榜单查询失败不阻断抽奖

`activityId` 每期更换（30 → 38），下期换活动只需改 `UNICOM_BATTLE_ACTIVITY_ID`。

## 快速开始

### 依赖

```bash
pip install httpx pycryptodome gmssl
```

`gmssl` 仅部分接口解密需要，可按需安装。

### 配置

通过环境变量 `chinaUnicomCookie` 传入 `token_online`（多账号用 `@` 分隔）：

```bash
export chinaUnicomCookie="token1@token2"
```

可用仓库内的 `login.py` 获取 token，或从抓包中提取 `onLine.htm` 请求体内的 `token_online` 字段。

### 运行

```bash
python cu.py
```

### 可选环境变量

| 变量 | 默认 | 说明 |
|------|------|------|
| `UNICOM_TTXC_GARBAGE_WAIT` | `28` | 农场垃圾任务等待秒数 |
| `UNICOM_TTXC_GROW_MAX_CHARGE_PER_LAND` | `20` | 单地块最大充能次数 |
| `UNICOM_TTXC_HARVEST_WAIT` | `3` | 收获等待秒数 |
| `UNICOM_CAMPUS_EDU_ID` | 见源码 | 校园季教育转存兑底内容 ID |
| `UNICOM_CAMPUS_ENT_ID` | 见源码 | 校园季娱乐转存兑底内容 ID |
| `UNICOM_BATTLE_ACTIVITY_ID` | `Mzg=` | 上传大比拼活动 ID（每期更换） |
| `UNICOM_BATTLE_UPLOAD_URL` | 见源码 | 上传大比拼上传域名（逗号分隔多候选） |
| `UNICOM_CLOUD_BATTLE_FILE` | `文本.txt` | 上传大比拼占位文件名 |
| `UNICOM_CLOUD_BATTLE_CONTENT` | `1` | 上传大比拼占位文件内容 |
| `SHANGDU_LOTTERY_MAX` | 剩余次数 | 单次最多抽奖次数 |
| `UNICOM_UPHONE_LOTTERY_COST` | `100` | 云手机积分十连触发余额阈值 |

## 说明

- 默认关闭 HTTP/2，使用 HTTP/1.1
- 日志中对手机号、token、ticket 等敏感信息做脱敏处理
- 请勿将 token、密钥等敏感信息提交到公开仓库

## 免责声明

本项目仅供学习交流，不得用于违法用途。开发者不对使用后果承担责任。
