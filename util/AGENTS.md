# util/ 工具模块

`main.py` 调用的三个独立子模块：加密 / 华米 API / 推送。彼此**不互相导入**，全部由 main 协调。

## 模块分工

| 文件 | 职责 | 关键常量 |
|------|------|----------|
| `aes_help.py` | AES-128-CBC + PKCS7 + base64（pycryptodome） | `HM_AES_KEY` / `HM_AES_IV`（华米固定，被 zepp_helper 导入） |
| `zepp_helper.py` | 华米 / Zepp 协议四端点调用 + token 检查 + 提交伪步数 | 伪装 User-Agent |
| `push_util.py` | pushplus / 企业微信 webhook / Telegram bot / 钉钉群机器人 四通道分发 | `PushConfig` dataclass |

## 任务导航

| 改什么 | 改哪里 |
|--------|--------|
| 协议端点 URL | `zepp_helper.py` 函数体内（每个函数独立） |
| 伪装 User-Agent | `zepp_helper.py` |
| 加密算法（用户 AES_KEY 部分） | `aes_help.py`（HM_* 不可改） |
| 增加推送渠道 | `push_util.py::push_results` + `PushConfig` 字段 |
| 推送时延判定 | `push_util.py::not_in_push_time_range`（读 cron_change_time） |

## 反模式（本目录专属）

- **HM_AES_KEY / HM_AES_IV 不可改**：华米协议固定，改了整个登录失败。
- **zepp_helper.check_app_token 第 169 行 `userid=1188760659`**：检测 token 有效性用的**占位 ID**，不是真实用户 ID，不要替换成真用户。
- **post_fake_brand_data 的 data_json 是硬编码 URL 编码模板**：用正则替换 date/step，改字段顺序需重新 URL 编码整个字符串。
- **函数返回 `(value, error_msg)` 二元组**：`zepp_helper.py` 全部函数返回 `(str | None, str | None)`，调用方必须同时处理成功和错误。
- **PushConfig 不是 dataclass**：是普通类带默认值，字段集合 = 推送配置全部集合。
- **push_plus_max 截断**：超过 30 条只发概要，调试时如果"为什么没收到详细内容"先查这个。
- **三模块之间无依赖**：aes_help 不导入 push_util，反之亦然；唯一跨模块导入是 zepp_helper 导入 aes_help 的 HM_* 常量。
