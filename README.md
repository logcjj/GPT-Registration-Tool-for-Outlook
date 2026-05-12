# GPT Registration Tool for Outlook

基于 Outlook 邮箱账号池的 ChatGPT / OpenAI 账号注册 CLI 工具。项目会串联邮箱领取、OTP 收取、OAuth 回调、登录态拉取、可选 2FA 设置和结果归档，适合在受控环境里做批量注册流程测试。

> 请只在合法、合规、获得授权的场景中使用本项目，并自行确认符合相关服务条款、当地法律和账号来源规范。

## 功能概览

- Outlook 邮箱账号池自动导入与领取
- 自动等待并读取邮箱 OTP
- 支持单账号、批量注册和多线程并发注册
- 支持代理池随机选择
- 支持注册成功后拉取 ChatGPT `accessToken`
- 支持可选 TOTP 2FA 设置
- 支持注册结果按批次归档到本地文件
- 支持注册成功后按配置触发外部 Flow

## 目录结构

```text
.
├── main.py                    # CLI 入口
├── requirements.txt           # Python 依赖
├── config/                    # 注册、邮箱、代理、浏览器指纹、2FA、Flow 配置
├── core/                      # 注册流程、会话、邮箱池、结果归档等核心逻辑
├── sentinel/                  # Sentinel runner 与 sdk 资源
└── 用于注册的邮箱.txt           # 本地邮箱素材文件，默认不提交
```

## 环境要求

- Python 3.10+
- Node.js 18+
- 可访问 OpenAI / ChatGPT / Outlook 取信服务的网络环境
- Outlook 邮箱素材，格式见下文

安装依赖：

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Windows PowerShell：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## 配置说明

### 邮箱账号池

默认开启 Outlook 自动取件，配置在 `config/email.py`：

```python
USE_EMAIL_SERVICE = True
OUTLOOK_ACCOUNTS_FILE = "用于注册的邮箱.txt"
```

在项目根目录创建 `用于注册的邮箱.txt`，每行一个 Outlook 账号素材：

```text
email@example.com----password----clientId----refreshToken
```

程序启动时会自动导入新增邮箱，并在注册成功或失败后更新本地账号池状态。

### 注册默认信息

配置在 `config/register.py`：

```python
REGISTER_EMAIL = ""
REGISTER_NAME = ""
REGISTER_BIRTHDAY = "2000-01-01"
```

- `REGISTER_EMAIL` 留空时，从 Outlook 邮箱池自动领取邮箱
- `REGISTER_NAME` 留空时，自动生成英文显示名
- `REGISTER_BIRTHDAY` 使用 `YYYY-MM-DD` 格式

### 代理池

配置在 `config/proxy.py`：

```python
PROXY_POOL = [
    # "socks5h://user:pass@host:port",
]
```

每次注册会从代理池随机选择一个代理；代理池为空时不使用代理。

### 2FA

配置在 `config/twofa.py`：

```python
ENABLE_2FA = False
```

开启后，注册成功会继续完成 TOTP enroll / activate，并把 `totp_secret` 写入本地归档。

### Flow 触发

配置在 `config/flow_trigger.py`：

```python
ENABLE_FLOW_TRIGGER = False
```

开启后，注册成功会用本次账号的 `accessToken` 调用配置好的外部 Flow 接口。请不要把真实 Bearer、Cookie 或内部接口配置提交到公开仓库。

## 使用方法

单次注册：

```bash
python main.py
```

连续注册 10 个账号：

```bash
python main.py -n 10 --continue-on-fail
```

并发注册：

```bash
python main.py -n 10 --workers 3 --continue-on-fail
```

显示详细日志：

```bash
python main.py --verbose
```

常用参数：

| 参数 | 说明 |
| --- | --- |
| `-n, --count` | 注册数量，默认 `1` |
| `--workers` | 并发线程数，默认 `1` |
| `--delay` | 每次提交或注册后的间隔秒数 |
| `--continue-on-fail` | 单个账号失败后继续执行 |
| `--verbose` | 输出详细步骤日志和错误堆栈 |

## 输出文件

注册运行后会在本地生成或更新以下文件：

```text
accounts/                         # 每批注册的归档目录
用于注册的邮箱.json                 # Outlook 邮箱池状态
用于注册的邮箱.txt                  # 可继续注册的邮箱素材
注册成功的邮箱.json                 # 注册成功账号完整状态
注册成功的邮箱.txt                  # 注册成功邮箱素材
注册成功的token.txt                 # 注册成功账号 accessToken
注册日志/                           # 本地运行日志
accounts_viewer.html               # 静态账号查看页
```

这些文件通常包含邮箱、token、refresh token、TOTP secret 或运行日志，默认不应提交到 Git。

## 安全注意事项

- 不要提交真实邮箱素材、access token、refresh token、代理账号密码、Bearer、Cookie 和批量归档文件
- 推送前先运行 `git status --short` 检查待提交内容
- 推荐把真实配置迁移到环境变量或本地私有配置文件中
- 若仓库公开，请确认 `config/` 下没有硬编码敏感信息

## 排障

- `未找到 Node 可执行文件`：确认已安装 Node.js，或通过 `NODE_EXECUTABLE` 指定绝对路径
- `sentinel-runner.js` 或 `sdk.js` 缺失：确认 `sentinel/` 目录完整
- 长时间未收到 OTP：检查 Outlook 账号素材是否有效、取信服务是否可用、`OTP_MAX_WAIT` 是否过短
- 并发注册失败率高：降低 `--workers`，增加 `--delay`，并检查代理池质量
