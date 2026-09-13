# 🌱 WorkBuddy Auto

> 把每天重复点击的工作交给 GitHub。  
> 配置一次，云端自动签到、做任务、领奖励，不需要服务器，也不用一直开着电脑。

![Python](https://img.shields.io/badge/Python-3.11-blue)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-Automation-2088FF)
![License](https://img.shields.io/badge/License-MIT-green)

## ✨ 它能做什么

| 功能 | 说明 |
| --- | --- |
| 🌞 每日签到 | 自动完成签到并领取奖励 |
| 🌱 成长任务 | 自动执行成长中心的云端任务 |
| 🎮 互动玩法 | 支持抽奖、盲盒、Buddy、旅行、连签兑换、补签等玩法 |
| 🔄 Token 续期 | 自动刷新 Token，并保存最新状态 |
| 👥 多账号 | 一个 JSON 文件可保存多个账号 |
| 📣 消息通知 | 可选 PushPlus 微信通知 |

GitHub Actions 默认在北京时间每天 **07:00、12:00 和 23:30** 自动运行。

## 🚀 五分钟部署

### 1. 用模板创建自己的仓库

点击页面右上角：

```text
Use this template
→ Create a new repository
```

+ 仓库名称可以随意填写
+ 可见性必须选择 `Private`
+ 最后点击 `Create repository`

> [!IMPORTANT]
> 使用中的仓库必须保持私有。任务运行后会保存用于自动续期的 Token，不要把自己的运行仓库公开。

### 2. 获取 Token

先在电脑上安装并登录 WorkBuddy，进入主界面后再运行对应命令。

<details>
<summary><strong>🍎 macOS 获取命令</strong></summary>

打开 Terminal，完整复制并运行：

```bash
python3 - <<'PY'
import json
import os

path = os.path.expanduser(
    "~/Library/Application Support/CodeBuddyExtension/Data/Public/auth/workbuddy-desktop.info"
)
with open(path, encoding="utf-8") as file:
    data = json.load(file)

print(
    f"{data['account']['phoneNumber']}:"
    f"{data['auth']['accessToken']}:"
    f"{data['auth']['refreshToken']}"
)
PY
```

</details>

<details>
<summary><strong>🪟 Windows 获取命令</strong></summary>

打开 PowerShell，完整复制并运行：

```powershell
$path = "$env:LOCALAPPDATA\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info"
$data = Get-Content -Raw -Encoding UTF8 $path | ConvertFrom-Json
"$($data.account.phoneNumber):$($data.auth.accessToken):$($data.auth.refreshToken)"
```

</details>

成功后，终端会输出一整行 `手机号:AT:RT`。复制完整一行即可；首次配置多个账号时，每个账号占一行。

> 如果提示文件不存在，请先确认 WorkBuddy 桌面端已经登录，并且已经进入主界面。

### 3. 把 Token 放进 GitHub Secret

进入自己刚创建的私有仓库：

```text
Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

填写：

```text
Name: WORKBUDDY_REFRESH_TOKEN
Secret: 手机号:AT:RT
```

多账号示例：

```text
手机号1:AT1:RT1
手机号2:AT2:RT2
```

需要微信通知时，可以再添加一个可选 Secret：

```text
Name: PUSHPLUS_TOKEN
Secret: 你的 PushPlus Token
```

### 4. 允许 Actions 保存新 Token

依次进入：

```text
Settings
→ Actions
→ General
→ Workflow permissions
```

选择 `Read and write permissions`，然后点击 `Save`。

### 5. 点一下，让它跑起来

打开仓库的 `Actions` 页面：

```text
Daily
→ Run workflow
→ Run workflow
```

看到绿色对勾就说明部署成功 🎉

之后它会按照设定时间自动运行，你不需要重复添加 Token，也不需要保持电脑开机。

## 🔄 Token 为什么不用天天换

第一次运行时，工作流会从 `WORKBUDDY_REFRESH_TOKEN` 读取账号信息。

运行成功后，它会：

```text
读取初始 Token
→ 自动刷新
→ 生成 wb_refresh_tokens.json
→ 保存最新 Token
→ 下次继续使用
```

所以正常情况下环境变量只需要配置一次。首次成功运行后，工作流会优先读取私有仓库中的 `wb_refresh_tokens.json`，并持续更新它。

### 后续添加或更换账号：直接生成 JSON

完成首次环境变量配置并成功运行后，后续不需要再修改 `WORKBUDDY_REFRESH_TOKEN`。登录需要添加或更新的 WorkBuddy 桌面账号，然后运行下面的命令；终端会输出可复制的完整 JSON。

<details>
<summary><strong>🍎 macOS 输出 JSON 命令</strong></summary>

```bash
python3 - <<'PY'
import json
import os

path = os.path.expanduser(
    "~/Library/Application Support/CodeBuddyExtension/Data/Public/auth/workbuddy-desktop.info"
)
with open(path, encoding="utf-8") as file:
    data = json.load(file)

phone = str(data["account"]["phoneNumber"])
result = {
    phone: {
        "refresh_token": data["auth"]["refreshToken"],
        "access_token": data["auth"]["accessToken"],
    }
}
print(json.dumps(result, ensure_ascii=False, indent=2))
PY
```

</details>

<details>
<summary><strong>🪟 Windows 输出 JSON 命令</strong></summary>

```powershell
$ErrorActionPreference = "Stop"
$path = "$env:LOCALAPPDATA\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info"
$data = Get-Content -Raw -Encoding UTF8 $path | ConvertFrom-Json
$phone = [string]$data.account.phoneNumber
$result = [ordered]@{}
$result[$phone] = [ordered]@{
    refresh_token = [string]$data.auth.refreshToken
    access_token  = [string]$data.auth.accessToken
}
$result | ConvertTo-Json -Depth 3
```

</details>

从终端输出开头的 `{` 到结尾的 `}` 全部复制，然后在私有仓库中编辑 `wb_refresh_tokens.json`。

单账号模板：

```json
{
  "手机号1": {
    "refresh_token": "RT1",
    "access_token": "AT1"
  }
}
```

多账号模板：

```json
{
  "手机号1": {
    "refresh_token": "RT1",
    "access_token": "AT1"
  },
  "手机号2": {
    "refresh_token": "RT2",
    "access_token": "AT2"
  }
}
```

> [!IMPORTANT]
> 多个账号必须写在同一个 JSON 最外层对象中。除最后一个账号外，每个账号结束的 `}` 后都要加英文逗号 `,`。不要把两个带外层 `{}` 的完整 JSON 直接首尾拼接。

添加新账号时保留原有节点并插入新节点；更新同一账号时，只替换对应手机号节点。提交后手动运行一次 `Daily` 验证，日志应显示正确的账号数量。

可参考仓库中的 [`wb_refresh_tokens.example.jsonc`](wb_refresh_tokens.example.jsonc)。它带有说明注释，仅供阅读；实际运行的 `wb_refresh_tokens.json` 必须保持标准 JSON，不能包含 `//` 或 `#` 注释。

## 📁 仓库里有什么

```text
.github/workflows/daily.yml  GitHub Actions 工作流
daily.py                     主程序
requirements.txt             Python 依赖
wb_refresh_tokens.example.jsonc  多账号填写示例（带注释，不参与运行）
LICENSE                      开源许可证
```

## 🔐 请认真看这几条

- 从模板创建的运行仓库必须保持 `Private`
- 不要公开手机号、AT、RT 或 `wb_refresh_tokens.json`
- 真实 Token 只能放在私有仓库的 `wb_refresh_tokens.json`，不要写进 README、脚本或工作流文件
- 不要让多个仓库同时运行同一个账号，否则 Token 可能互相覆盖
- 更换仓库前，先停用旧仓库的 Actions

## ❓ 常见问题

<details>
<summary><strong>为什么有些桌面任务没有执行？</strong></summary>

GitHub Actions 使用 Linux。只能在 Windows 桌面端完成的任务会自动跳过，其他云端任务仍会正常运行。

</details>

<details>
<summary><strong>为什么提示找不到认证文件？</strong></summary>

请先登录 WorkBuddy 桌面端并进入主界面，然后重新运行获取 Token 的命令。

</details>

<details>
<summary><strong>以后还需要重新添加 Token 吗？</strong></summary>

通常不需要。工作流会自动刷新并保存最新 Token；只有认证彻底失效时才需要重新获取。

</details>

## 📄 许可证

本项目采用 [MIT License](LICENSE)。

如果这个项目帮你省下了一点时间，欢迎点个 ⭐。
