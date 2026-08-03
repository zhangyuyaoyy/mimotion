# 项目知识库

**生成时间**: 2026-08-03
**Commit**: 3d408d8
**分支**: master

## 概览

小米运动 / Zepp Life 自动刷步数 GitHub Actions 项目。Python 3.10 单脚本架构，通过模拟华米 / Zepp APP 协议提交伪造步数，配合 cron 随机化规避检测，结果通过 pushplus / 企业微信 / Telegram 多渠道推送。

## 目录结构

```
mimotion/
├── main.py                # 主入口：刷步数完整流程（MiMotionRunner）
├── inspect_configs.py     # 配置提取工具（加密打印/推到微信/TG）
├── cron_convert.sh        # cron 转换 + 随机化 + 日志持久化（bash）
├── encrypted_tokens.data  # 运行时生成：加密 token 缓存（git 提交回 master）
├── cron_change_time       # 运行时生成：cron 执行日志（UTC/北京/next exec）
├── .github/workflows/
│   ├── run.yml            # "刷步数"：schedule cron → main.py
│   ├── cron.yml           # "Random Cron"：workflow_run → cron_convert.sh + git commit
│   ├── inspect_configs.yml# "提取配置信息"：workflow_dispatch → inspect_configs.py
│   └── star.yml           # "star watcher"：被 star 时打印时间
├── util/                  # 工具模块（aes_help / zepp_helper / push_util）
└── local/
    └── decrypt_data.py    # 本地小工具：解密 encrypted_tokens.data
```

## 任务导航

| 任务 | 入口 | 说明 |
|------|------|------|
| 改登录/刷步逻辑 | `main.py::MiMotionRunner` | 三层 token 缓存降级链 |
| 改华米 API 协议 | `util/zepp_helper.py` | 4 个端点：login/grant/check/post |
| 改加密方式 | `util/aes_help.py` | AES-128-CBC + PKCS7 |
| 改推送渠道 | `util/push_util.py` | pushplus / 企业微信 / Telegram |
| 改 cron 随机化 | `cron_convert.sh` + `.github/workflows/cron.yml` | bash 跨平台 |
| 配置提取工作流 | `inspect_configs.py` + `.github/workflows/inspect_configs.yml` | 单独触发 |
| 部署/Secrets | `README.md` | 完整用户文档 |
| 本地解密 token | `local/decrypt_data.py` | 一次性脚本 |

## 代码地图

| 符号 | 类型 | 位置 | 角色 |
|------|------|------|------|
| `MiMotionRunner` | class | `main.py` | 刷步数主流程；持有三层 token 缓存与降级刷新 |
| `execute` | func | `main.py` | 多账号遍历（# 分割）；串行/并发；调 push_results |
| `prepare_user_tokens` / `persist_user_tokens` | func | `main.py` | 读写 encrypted_tokens.data（AES 加密） |
| `get_min_max_by_time` | func | `main.py` | 按北京时间线性插值步数范围（22 点达 MAX） |
| `desensitize_user_name` | func | `main.py` | 日志脱敏（前3后4） |
| `login_access_token` | func | `util/zepp_helper.py` | POST registrations/tokens → access_token |
| `grant_login_tokens` | func | `util/zepp_helper.py` | POST /v2/client/login → login_token |
| `grant_app_token` | func | `util/zepp_helper.py` | GET /v1/client/app_tokens → app_token |
| `check_app_token` | func | `util/zepp_helper.py` | GET getUserInfo.json 验证 app_token |
| `post_fake_brand_data` | func | `util/zepp_helper.py` | POST band_data.json 提交伪步数 |
| `encrypt_data` / `decrypt_data` | func | `util/aes_help.py` | AES-128-CBC + 随机/固定 IV |
| `PushConfig` | class | `util/push_util.py` | 推送配置 dataclass |
| `push_results` | func | `util/push_util.py` | 多渠道分发（pushplus/企微/TG） |
| `not_in_push_time_range` | func | `util/push_util.py` | 读 cron_change_time 判定真实执行小时 |
| `inspect_hours` / `convert_utc_to_shanghai` / `persist_execute_log` | func | `cron_convert.sh` | cron 解析/转换/日志写入 |

## 约定

### 配置（CONFIG env，JSON）

`USER`/`PWD`（多账号用 `#` 分割，数量必须匹配）/`MIN_STEP`/`MAX_STEP`/`PUSH_PLUS_TOKEN`/`PUSH_PLUS_HOUR`/`PUSH_PLUS_MAX`（默认 30）/`PUSH_WECHAT_WEBHOOK_KEY`/`TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID`/`SLEEP_GAP`（默认 5 秒）/`USE_CONCURRENT`（字符串 `'True'` 启用并发，默认 False）。

### GitHub Secrets

- `PAT`（必需）：fine-grained token，cron.yml / inspect_configs.yml 用于 git push 回仓库
- `AES_KEY`（必需 16 字节）：加密 encrypted_tokens.data
- `CONFIG`（必需 JSON）：上述配置
- `INSPECT_AES_KEY` / `INSPECT_WECHAT_HOOK_KEY` / `INSPECT_TELEGRAM_*`（可选）：配置提取工作流

### GitHub Variables

- `CRON_HOURS`（可选）：逗号分隔 UTC 小时如 `"0,2,4,6,8,14"`，覆盖 run.yml 的 cron 小时部分并剔除当前小时

### 时区

全部硬编码 `Asia/Shanghai`（北京时间）；cron 表达式用 UTC（北京 -8）。

### 加密约定

- `HM_AES_KEY=b'xeNtBVqzDc6tuNTh'` / `HM_AES_IV=b'MAAAYAAAAAAAAABg'`（16 字节固定，华米协议要求，定义在 `util/aes_help.py`，`zepp_helper.py` 导入）
- 用户 `AES_KEY` 用于 encrypted_tokens.data 持久化（IV 随机）
- 首次更换 AES_KEY 时日志出现"密钥不正确或者加密内容损坏 放弃token"属**正常**（旧文件用旧密钥，下次刷新会重新生成）

### 多账号

`#` 分割；账号/密码数量不匹配 → `exit(1)`；同 IP 多账号可能触发 429。

### 登录降级链

```
check_app_token(app_token)  ──失效──►  grant_app_token(login_token)
                                              │
                                              ▼ 失效
                                   grant_login_tokens(access_token)
                                              │
                                              ▼ 失效
                                   login_access_token(user, password)
```

每次刷新都更新对应 `*_time` 字段；全部失效才走完整登录。

### 用户名判定

自动加 `+86` 前缀后判定是否手机号（`is_phone`）；含 `@` 视为邮箱 → 影响 `grant_login_tokens` 的 `third_name`（`huami_phone` / `huami_email`）。

## 反模式（本项目专属）

- **无 requirements.txt**：依赖在 workflow 内联 `pip3 install requests pytz pycryptodome`。改依赖必须同步改所有 workflow。
- **运行时文件提交 git**：`encrypted_tokens.data` 和 `cron_change_time` 每次 workflow 运行后被 cron.yml commit 回 master。fork sync 会**覆盖**这两个文件，README 警告用户备份。
- **不支持 PR**：README 明确说同步请用 Sync fork，不要提交 PR。
- **硬编码华米密钥**：HM_AES_KEY / HM_AES_IV 是协议固定值，不可改。
- **硬编码 dummy userid**：`zepp_helper.check_app_token` 第 169 行 `userid=1188760659` 是检测 token 用的占位 ID，**不是**真实用户。
- **硬编码 User-Agent**：`MiFit6.14.0 (M2007J1SC; Android 12; Density/2.75)` 等伪装字符串。
- **裸 `except:`**：main.py 多处仅记录到 log_str，吞掉异常类型信息。
- **fake_ip 已禁用**：main.py 第 107-108 行注释掉的伪 IP 生成（223.64-117.x.x），改用需谨慎。
- **push_plus_max 概要截断**：默认 30 条，超过只推送概要避免内容过长。

## 独特风格

- **三层 token 缓存**：app_token / login_token / access_token 持久化在 git，规避重复登录触发风控。
- **cron 自举随机化**：每次刷步数成功后由 `cron.yml` 通过 `workflow_run` 触发 `cron_convert.sh`，随机化下次执行的分钟数并 commit。
- **跨平台 bash**：`cron_convert.sh` 用 `uname -s` 判断 Darwin / Linux，`sed -i ''` vs `sed -i`。
- **推送时延补偿**：`not_in_push_time_range` 读 `cron_change_time` 最后一行的北京时间小时，绕过 GitHub Actions schedule 延迟。
- **配置提取安全通道**：`inspect_configs.py` 故意不用 pushplus（README 注"不安全"），改用加密 base64 打印 / 企微 / TG。
- **日志脱敏**：`desensitize_user_name` 保留前 3 后 4 字符。

## 命令

```bash
# 本地运行（需先 export CONFIG / AES_KEY 等 env）
python3 main.py

# 本地解密 token 缓存（需 export AES_KEY）
python3 local/decrypt_data.py

# 手动触发工作流（在 GitHub Actions UI）
# - 刷步数：workflow_dispatch on run.yml
# - 提取配置：workflow_dispatch on inspect_configs.yml
```

依赖：`requests` `pytz` `pycryptodome`（无 requirements.txt，在 workflow 内联安装）。

## 注意事项

- **新部署首跑**：encrypted_tokens.data 不存在或密钥不匹配属正常，首次会全量登录刷新。
- **token 失效诊断**：看 main.py 日志的"放弃xxx 换取新token"链路，按降级链顺序排查。
- **429 限流**：同账号高频刷新或同 IP 多账号会触发；调大 SLEEP_GAP 或减少账号。
- **cron 不触发**：检查 `cron_change_time` 文件是否被 fork sync 覆盖、GitHub Variables CRON_HOURS 格式。
- **推送不到**：分别核查 pushplus token / 企微 webhook key / TG bot token+chat_id；`not_in_push_time_range` 可能因 cron_change_time 缺失而误判。
- **改协议端点**：所有 URL 集中在 `util/zepp_helper.py` 顶部函数体内，User-Agent 也在该文件。
