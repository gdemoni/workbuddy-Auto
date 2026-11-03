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

先在电脑上安装并登录 WorkBuddy，进入主界面后再运行对应命令。按提示填写**当前登录账号的手机号**。

以下四段命令均可**独立运行**，自行读取并处理本机凭据，不依赖其他命令或剪贴板已有内容。支持明文及受支持的 `$wbEncrypted / suite 1` 加密字段；其他客户端版本可能需要进一步适配。

命令只读取认证文件，不修改客户端登录状态、不发送网络请求。加密字段通过本机 WorkBuddy 客户端运行时处理。提取成功不代表已通过服务端有效性验证。

#### 首次配置：获取 手机号:AT:RT

仅在首次配置 GitHub Secret 时，按系统选择下面一段命令。输出格式为 `手机号:AT:RT`，复制完整一行到第 3 步的 `WORKBUDDY_REFRESH_TOKEN`。

<details>
<summary><strong>🍎 macOS 获取命令（首次配置）</strong></summary>

打开 Terminal，确认有 Python 3。完整复制代码块，包含第一行 `python3 - <<'PY'` 和最后一行 `PY`。

```bash
python3 - <<'PY'
import json
import os
import plistlib
import re
import subprocess
import sys
from pathlib import Path

def fail(msg):
    sys.exit("提取失败：" + msg)

try:
    with open("/dev/tty", "w") as tty:
        tty.write("请输入当前 WorkBuddy 账号的手机号：")
        tty.flush()
    with open("/dev/tty", "r") as tty:
        phone = tty.readline().strip()
except OSError:
    fail("无法读取手机号，请在 Mac 终端运行。")
if not re.fullmatch(r"\+?[0-9]{5,20}", phone):
    fail("手机号为空或格式不正确。")

path = Path.home() / "Library/Application Support/CodeBuddyExtension/Data/Public/auth/workbuddy-desktop.info"
try:
    data = json.loads(path.read_text(encoding="utf-8"))
except Exception:
    fail("无法读取认证文件，请先登录 WorkBuddy。")
auth = data.get("auth") or {}
values = {}
for field in ("accessToken", "refreshToken"):
    top = data.get(field)
    values[field] = top if isinstance(top, str) and top else auth.get(field)
    if values[field] is None:
        fail("认证文件缺少 " + field)

js = r"""
const crypto = require('crypto');
let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', s => input += s);
process.stdin.on('end', () => {
  let key;
  try {
    function decrypt(v) {
      if (typeof v === 'string') return v;
      if (!v || v.$wbEncrypted !== 1) throw Error('UNSUPPORTED_FORMAT');
      const e = JSON.parse(Buffer.from(v.envelope,'base64').toString('utf8'));
      if (e.suite !== 1) throw Error('UNSUPPORTED_SUITE');
      if (!key) {
        const p = JSON.parse(process._linkedBinding('electron_browser_workbuddy_storage').loggerGet());
        if (p.version !== 1 || typeof p.atRestSecretKey !== 'string')
          throw Error('RUNTIME_UNAVAILABLE');
        key = crypto.createHash('sha256').update(p.atRestSecretKey,'utf8').digest();
      }
      if (crypto.createHash('sha256').update(key).digest('hex').slice(0,16) !== e.keyId)
        throw Error('KEY_MISMATCH');
      function lp(s) {
        const b = Buffer.from(s,'utf8'), n = Buffer.alloc(4);
        n.writeUInt32BE(b.length);
        return Buffer.concat([n,b]);
      }
      const aad = Buffer.concat([
        Buffer.from('WB-AAD\0','ascii'), Buffer.from([1]),
        lp('WBEV1'), lp('sym-v1'), Buffer.from([0,0,0,1]),
        lp(e.keyId), Buffer.from([2,0,0])
      ]);
      const c = crypto.createDecipheriv('aes-256-gcm',key,Buffer.from(e.nonce,'base64'),{authTagLength:16});
      c.setAAD(aad);
      c.setAuthTag(Buffer.from(e.authTag,'base64'));
      const plain = Buffer.concat([c.update(Buffer.from(e.ciphertext,'base64')),c.final()]);
      try { return plain.toString('utf8'); }
      finally { plain.fill(0); }
    }
    const result = {};
    for (const [k,v] of Object.entries(JSON.parse(input))) result[k] = decrypt(v);
    process.stdout.write(JSON.stringify({ok:true,result}));
  } catch (e) {
    const allowed = ['UNSUPPORTED_FORMAT','UNSUPPORTED_SUITE','RUNTIME_UNAVAILABLE','KEY_MISMATCH'];
    process.stdout.write(JSON.stringify({
      ok:false,error:allowed.includes(e.message) ? e.message : 'DECRYPT_FAILED'
    }));
  } finally {
    if (key) key.fill(0);
  }
});
"""

if any(not isinstance(v, str) for v in values.values()):
    executables = []
    override = os.environ.get("WORKBUDDY_EXE")
    if override:
        executables.append(Path(override).expanduser())
    else:
        for root in (Path("/Applications"), Path.home() / "Applications"):
            for bundle in root.glob("*.app"):
                if "workbuddy" not in bundle.name.lower():
                    continue
                try:
                    with (bundle / "Contents/Info.plist").open("rb") as f:
                        name = plistlib.load(f)["CFBundleExecutable"]
                    if Path(name).name == name:
                        executables.append(bundle / "Contents/MacOS" / name)
                except Exception:
                    pass
    executables = [p for p in executables if p.is_file()]
    if not executables:
        fail("未找到 WorkBuddy 客户端。")
    env = os.environ.copy()
    env["ELECTRON_RUN_AS_NODE"] = "1"
    errors = []
    for exe in executables:
        try:
            proc = subprocess.run(
                [str(exe), "-e", js], input=json.dumps(values),
                text=True, capture_output=True, env=env, timeout=20
            )
            result = json.loads(proc.stdout)
            if result.get("ok"):
                values = result["result"]
                break
            errors.append(result.get("error", "UNKNOWN"))
        except subprocess.TimeoutExpired:
            errors.append("TIMEOUT")
        except Exception:
            errors.append("RUNTIME_UNAVAILABLE")
    else:
        fail("解密未成功：" + ", ".join(errors))

def normalize(v):
    if not isinstance(v, str):
        return ""
    v = v.strip()
    for _ in range(2):
        try:
            decoded = json.loads(v)
        except (ValueError, TypeError):
            break
        if not isinstance(decoded, str):
            break
        v = decoded.strip()
    return v

at = normalize(values["accessToken"])
rt = normalize(values["refreshToken"])
for name, value in (("AT", at), ("RT", rt)):
    if not re.fullmatch(r"[A-Za-z0-9._~+/-]+=*", value):
        fail(name + " 为空或格式异常，未输出凭据。")

print(f"{phone}:{at}:{rt}")
PY
```

</details>

<details>
<summary><strong>🪟 Windows 获取命令（首次配置）</strong></summary>

打开 **PowerShell（不是 CMD）**，完整复制代码块运行。不需要另装 Python 或 Node.js。成功后显示完整一行并尝试复制到剪贴板。

```powershell
& {
    $ErrorActionPreference = "Stop"
    try {
        $phone = (Read-Host "请输入当前 WorkBuddy 账号的手机号").Trim()
        if ($phone -notmatch '^\+?[0-9]{5,20}$') {
            throw "手机号格式不正确"
        }
        $path = Join-Path $env:LOCALAPPDATA "CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info"
        if (-not (Test-Path -LiteralPath $path)) {
            throw "找不到认证文件，请先登录 WorkBuddy"
        }
        $data = Get-Content -LiteralPath $path -Raw -Encoding UTF8 | ConvertFrom-Json
        $tokens = @{}
        foreach ($name in @("accessToken", "refreshToken")) {
            if ($data.$name -is [string] -and $data.$name) {
                $tokens[$name] = $data.$name
            } else {
                $tokens[$name] = $data.auth.$name
            }
            if ($null -eq $tokens[$name]) { throw "认证文件缺少 $name" }
        }

        $js = @'
const crypto = require('crypto');
let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', s => input += s);
process.stdin.on('end', () => {
  let key;
  try {
    function decrypt(v) {
      if (typeof v === 'string') return v;
      if (!v || v.$wbEncrypted !== 1) throw Error('UNSUPPORTED_FORMAT');
      const e = JSON.parse(Buffer.from(v.envelope,'base64').toString('utf8'));
      if (e.suite !== 1) throw Error('UNSUPPORTED_SUITE');
      if (!key) {
        const p = JSON.parse(process._linkedBinding('electron_browser_workbuddy_storage').loggerGet());
        if (p.version !== 1 || typeof p.atRestSecretKey !== 'string')
          throw Error('RUNTIME_UNAVAILABLE');
        key = crypto.createHash('sha256').update(p.atRestSecretKey,'utf8').digest();
      }
      if (crypto.createHash('sha256').update(key).digest('hex').slice(0,16) !== e.keyId)
        throw Error('KEY_MISMATCH');
      function lp(s) {
        const b = Buffer.from(s,'utf8'), n = Buffer.alloc(4);
        n.writeUInt32BE(b.length);
        return Buffer.concat([n,b]);
      }
      const aad = Buffer.concat([
        Buffer.from('WB-AAD\0','ascii'), Buffer.from([1]),
        lp('WBEV1'), lp('sym-v1'), Buffer.from([0,0,0,1]),
        lp(e.keyId), Buffer.from([2,0,0])
      ]);
      const c = crypto.createDecipheriv('aes-256-gcm',key,Buffer.from(e.nonce,'base64'),{authTagLength:16});
      c.setAAD(aad);
      c.setAuthTag(Buffer.from(e.authTag,'base64'));
      const plain = Buffer.concat([c.update(Buffer.from(e.ciphertext,'base64')),c.final()]);
      try { return plain.toString('utf8'); }
      finally { plain.fill(0); }
    }
    const result = {};
    for (const [k,v] of Object.entries(JSON.parse(input))) result[k] = decrypt(v);
    process.stdout.write(JSON.stringify({ok:true,result}));
  } catch (e) {
    const allowed = ['UNSUPPORTED_FORMAT','UNSUPPORTED_SUITE','RUNTIME_UNAVAILABLE','KEY_MISMATCH'];
    process.stdout.write(JSON.stringify({
      ok:false,error:allowed.includes(e.message) ? e.message : 'DECRYPT_FAILED'
    }));
  } finally {
    if (key) key.fill(0);
  }
});
'@

        $encrypted = @($tokens.Values | Where-Object { $_ -isnot [string] }).Count -gt 0
        if ($encrypted) {
            $paths = @()
            if ($env:WORKBUDDY_EXE) {
                $paths += $env:WORKBUDDY_EXE
            } else {
                $paths += @(
                    "$env:LOCALAPPDATA\Programs\WorkBuddy\WorkBuddy.exe"
                    "$env:LOCALAPPDATA\WorkBuddy\WorkBuddy.exe"
                    "$env:ProgramFiles\WorkBuddy\WorkBuddy.exe"
                    "${env:ProgramFiles(x86)}\WorkBuddy\WorkBuddy.exe"
                )
                foreach ($root in @(
                    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*"
                    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*"
                    "HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*"
                )) {
                    foreach ($app in (Get-ItemProperty $root -ErrorAction SilentlyContinue)) {
                        if ($app.DisplayName -notlike "*WorkBuddy*") { continue }
                        if ($app.InstallLocation) {
                            $paths += Join-Path $app.InstallLocation "WorkBuddy.exe"
                        }
                        if ($app.DisplayIcon) {
                            $icon = ([string]$app.DisplayIcon -replace ',\s*-?\d+\s*$', '').Trim().Trim('"')
                            if ([IO.Path]::GetFileName($icon) -ieq "WorkBuddy.exe") {
                                $paths += $icon
                            }
                        }
                    }
                }
            }
            $paths = @($paths | Where-Object {
                Test-Path -LiteralPath $_ -PathType Leaf
            } | Select-Object -Unique)
            if ($paths.Count -eq 0) {
                throw '找不到客户端。先运行 $env:WORKBUDDY_EXE="D:\实际路径\WorkBuddy.exe"，再重新运行整段代码'
            }
            $encoded = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($js))
            $payload = $tokens | ConvertTo-Json -Depth 10 -Compress
            $success = $false
            $errors = @()
            foreach ($exe in $paths) {
                $process = New-Object System.Diagnostics.Process
                try {
                    $info = New-Object System.Diagnostics.ProcessStartInfo
                    $info.FileName = $exe
                    $info.Arguments = '-e "eval(Buffer.from(''' + $encoded + ''',''base64'').toString(''utf8''))"'
                    $info.UseShellExecute = $false
                    $info.CreateNoWindow = $true
                    $info.RedirectStandardInput = $true
                    $info.RedirectStandardOutput = $true
                    $info.RedirectStandardError = $true
                    $info.EnvironmentVariables["ELECTRON_RUN_AS_NODE"] = "1"
                    $process.StartInfo = $info
                    [void]$process.Start()
                    $outTask = $process.StandardOutput.ReadToEndAsync()
                    $errTask = $process.StandardError.ReadToEndAsync()
                    $process.StandardInput.Write($payload)
                    $process.StandardInput.Close()
                    if (-not $process.WaitForExit(20000)) {
                        $process.Kill()
                        $errors += "TIMEOUT"
                        continue
                    }
                    $reply = $outTask.GetAwaiter().GetResult() | ConvertFrom-Json
                    [void]$errTask.GetAwaiter().GetResult()
                    if ($reply.ok) {
                        $tokens = @{
                            accessToken = $reply.result.accessToken
                            refreshToken = $reply.result.refreshToken
                        }
                        $success = $true
                        break
                    }
                    $errors += [string]$reply.error
                } catch {
                    $errors += "RUNTIME_UNAVAILABLE"
                } finally {
                    $process.Dispose()
                }
            }
            if (-not $success) { throw ("解密失败：" + ($errors -join ", ")) }
        }

        foreach ($name in @("accessToken", "refreshToken")) {
            $v = $tokens[$name]
            if ($v -isnot [string]) { throw "$name 类型异常" }
            $v = $v.Trim()
            for ($i = 0; $i -lt 2; $i++) {
                try {
                    $decoded = ConvertFrom-Json -InputObject $v -ErrorAction Stop
                    if ($decoded -isnot [string]) { break }
                    $v = $decoded.Trim()
                } catch { break }
            }
            if (-not $v -or $v -cnotmatch '^[A-Za-z0-9._~+/-]+=*$') {
                throw "$name 为空或格式异常"
            }
            $tokens[$name] = $v
        }

        $text = "{0}:{1}:{2}" -f $phone, $tokens["accessToken"], $tokens["refreshToken"]
        Write-Output $text
        try {
            Set-Clipboard -Value $text -ErrorAction Stop
            Write-Host "已复制到剪贴板，可以直接粘贴到 GitHub Secret。" -ForegroundColor Green
        } catch {
            Write-Host "自动复制失败，请手动复制上面的输出。" -ForegroundColor Yellow
        }
    } catch {
        Write-Host ("提取失败：" + $_.Exception.Message) -ForegroundColor Red
    }
}
```

</details>

#### 后续维护：生成 JSON 文件内容

添加或更新账号时，再按系统选择下面一段命令。输出为完整 JSON，用于编辑私有仓库中的 `wb_refresh_tokens.json`，不要填入首次配置的 Secret。

<details>
<summary><strong>🍎 macOS 生成 JSON 命令（后续维护）</strong></summary>

打开 Terminal，完整复制代码块运行。这段命令可独立执行，不需要先运行首次配置命令。

```bash
python3 - <<'PY'
import json
import os
import plistlib
import re
import subprocess
import sys
from pathlib import Path

def fail(msg):
    sys.exit("提取失败：" + msg)

try:
    with open("/dev/tty", "w") as tty:
        tty.write("请输入当前 WorkBuddy 账号的手机号：")
        tty.flush()
    with open("/dev/tty", "r") as tty:
        phone = tty.readline().strip()
except OSError:
    fail("无法读取手机号，请在 Mac 终端运行。")
if not re.fullmatch(r"\+?[0-9]{5,20}", phone):
    fail("手机号为空或格式不正确。")

path = Path.home() / "Library/Application Support/CodeBuddyExtension/Data/Public/auth/workbuddy-desktop.info"
try:
    data = json.loads(path.read_text(encoding="utf-8"))
except Exception:
    fail("无法读取认证文件，请先登录 WorkBuddy。")
auth = data.get("auth") or {}
values = {}
for field in ("accessToken", "refreshToken"):
    top = data.get(field)
    values[field] = top if isinstance(top, str) and top else auth.get(field)
    if values[field] is None:
        fail("认证文件缺少 " + field)

js = r"""
const crypto = require('crypto');
let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', s => input += s);
process.stdin.on('end', () => {
  let key;
  try {
    function decrypt(v) {
      if (typeof v === 'string') return v;
      if (!v || v.$wbEncrypted !== 1) throw Error('UNSUPPORTED_FORMAT');
      const e = JSON.parse(Buffer.from(v.envelope,'base64').toString('utf8'));
      if (e.suite !== 1) throw Error('UNSUPPORTED_SUITE');
      if (!key) {
        const p = JSON.parse(process._linkedBinding('electron_browser_workbuddy_storage').loggerGet());
        if (p.version !== 1 || typeof p.atRestSecretKey !== 'string')
          throw Error('RUNTIME_UNAVAILABLE');
        key = crypto.createHash('sha256').update(p.atRestSecretKey,'utf8').digest();
      }
      if (crypto.createHash('sha256').update(key).digest('hex').slice(0,16) !== e.keyId)
        throw Error('KEY_MISMATCH');
      function lp(s) {
        const b = Buffer.from(s,'utf8'), n = Buffer.alloc(4);
        n.writeUInt32BE(b.length);
        return Buffer.concat([n,b]);
      }
      const aad = Buffer.concat([
        Buffer.from('WB-AAD\0','ascii'), Buffer.from([1]),
        lp('WBEV1'), lp('sym-v1'), Buffer.from([0,0,0,1]),
        lp(e.keyId), Buffer.from([2,0,0])
      ]);
      const c = crypto.createDecipheriv('aes-256-gcm',key,Buffer.from(e.nonce,'base64'),{authTagLength:16});
      c.setAAD(aad);
      c.setAuthTag(Buffer.from(e.authTag,'base64'));
      const plain = Buffer.concat([c.update(Buffer.from(e.ciphertext,'base64')),c.final()]);
      try { return plain.toString('utf8'); }
      finally { plain.fill(0); }
    }
    const result = {};
    for (const [k,v] of Object.entries(JSON.parse(input))) result[k] = decrypt(v);
    process.stdout.write(JSON.stringify({ok:true,result}));
  } catch (e) {
    const allowed = ['UNSUPPORTED_FORMAT','UNSUPPORTED_SUITE','RUNTIME_UNAVAILABLE','KEY_MISMATCH'];
    process.stdout.write(JSON.stringify({
      ok:false,error:allowed.includes(e.message) ? e.message : 'DECRYPT_FAILED'
    }));
  } finally {
    if (key) key.fill(0);
  }
});
"""

if any(not isinstance(v, str) for v in values.values()):
    executables = []
    override = os.environ.get("WORKBUDDY_EXE")
    if override:
        executables.append(Path(override).expanduser())
    else:
        for root in (Path("/Applications"), Path.home() / "Applications"):
            for bundle in root.glob("*.app"):
                if "workbuddy" not in bundle.name.lower():
                    continue
                try:
                    with (bundle / "Contents/Info.plist").open("rb") as f:
                        name = plistlib.load(f)["CFBundleExecutable"]
                    if Path(name).name == name:
                        executables.append(bundle / "Contents/MacOS" / name)
                except Exception:
                    pass
    executables = [p for p in executables if p.is_file()]
    if not executables:
        fail("未找到 WorkBuddy 客户端。")
    env = os.environ.copy()
    env["ELECTRON_RUN_AS_NODE"] = "1"
    errors = []
    for exe in executables:
        try:
            proc = subprocess.run(
                [str(exe), "-e", js], input=json.dumps(values),
                text=True, capture_output=True, env=env, timeout=20
            )
            result = json.loads(proc.stdout)
            if result.get("ok"):
                values = result["result"]
                break
            errors.append(result.get("error", "UNKNOWN"))
        except subprocess.TimeoutExpired:
            errors.append("TIMEOUT")
        except Exception:
            errors.append("RUNTIME_UNAVAILABLE")
    else:
        fail("解密未成功：" + ", ".join(errors))

def normalize(v):
    if not isinstance(v, str):
        return ""
    v = v.strip()
    for _ in range(2):
        try:
            decoded = json.loads(v)
        except (ValueError, TypeError):
            break
        if not isinstance(decoded, str):
            break
        v = decoded.strip()
    return v

at = normalize(values["accessToken"])
rt = normalize(values["refreshToken"])
for name, value in (("AT", at), ("RT", rt)):
    if not re.fullmatch(r"[A-Za-z0-9._~+/-]+=*", value):
        fail(name + " 为空或格式异常，未输出凭据。")

result = {
    phone: {
        "refresh_token": rt,
        "access_token": at
    }
}
print(json.dumps(result, ensure_ascii=False, indent=2))
PY
```

</details>

<details>
<summary><strong>🪟 Windows 生成 JSON 命令（后续维护）</strong></summary>

打开 PowerShell，完整复制代码块运行。这段命令可独立执行，不读取剪贴板；成功后显示 JSON 并尝试复制到剪贴板。

```powershell
& {
    $ErrorActionPreference = "Stop"
    try {
        $phone = (Read-Host "请输入当前 WorkBuddy 账号的手机号").Trim()
        if ($phone -notmatch '^\+?[0-9]{5,20}$') {
            throw "手机号格式不正确"
        }
        $path = Join-Path $env:LOCALAPPDATA "CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info"
        if (-not (Test-Path -LiteralPath $path)) {
            throw "找不到认证文件，请先登录 WorkBuddy"
        }
        $data = Get-Content -LiteralPath $path -Raw -Encoding UTF8 | ConvertFrom-Json
        $tokens = @{}
        foreach ($name in @("accessToken", "refreshToken")) {
            if ($data.$name -is [string] -and $data.$name) {
                $tokens[$name] = $data.$name
            } else {
                $tokens[$name] = $data.auth.$name
            }
            if ($null -eq $tokens[$name]) { throw "认证文件缺少 $name" }
        }

        $js = @'
const crypto = require('crypto');
let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', s => input += s);
process.stdin.on('end', () => {
  let key;
  try {
    function decrypt(v) {
      if (typeof v === 'string') return v;
      if (!v || v.$wbEncrypted !== 1) throw Error('UNSUPPORTED_FORMAT');
      const e = JSON.parse(Buffer.from(v.envelope,'base64').toString('utf8'));
      if (e.suite !== 1) throw Error('UNSUPPORTED_SUITE');
      if (!key) {
        const p = JSON.parse(process._linkedBinding('electron_browser_workbuddy_storage').loggerGet());
        if (p.version !== 1 || typeof p.atRestSecretKey !== 'string')
          throw Error('RUNTIME_UNAVAILABLE');
        key = crypto.createHash('sha256').update(p.atRestSecretKey,'utf8').digest();
      }
      if (crypto.createHash('sha256').update(key).digest('hex').slice(0,16) !== e.keyId)
        throw Error('KEY_MISMATCH');
      function lp(s) {
        const b = Buffer.from(s,'utf8'), n = Buffer.alloc(4);
        n.writeUInt32BE(b.length);
        return Buffer.concat([n,b]);
      }
      const aad = Buffer.concat([
        Buffer.from('WB-AAD\0','ascii'), Buffer.from([1]),
        lp('WBEV1'), lp('sym-v1'), Buffer.from([0,0,0,1]),
        lp(e.keyId), Buffer.from([2,0,0])
      ]);
      const c = crypto.createDecipheriv('aes-256-gcm',key,Buffer.from(e.nonce,'base64'),{authTagLength:16});
      c.setAAD(aad);
      c.setAuthTag(Buffer.from(e.authTag,'base64'));
      const plain = Buffer.concat([c.update(Buffer.from(e.ciphertext,'base64')),c.final()]);
      try { return plain.toString('utf8'); }
      finally { plain.fill(0); }
    }
    const result = {};
    for (const [k,v] of Object.entries(JSON.parse(input))) result[k] = decrypt(v);
    process.stdout.write(JSON.stringify({ok:true,result}));
  } catch (e) {
    const allowed = ['UNSUPPORTED_FORMAT','UNSUPPORTED_SUITE','RUNTIME_UNAVAILABLE','KEY_MISMATCH'];
    process.stdout.write(JSON.stringify({
      ok:false,error:allowed.includes(e.message) ? e.message : 'DECRYPT_FAILED'
    }));
  } finally {
    if (key) key.fill(0);
  }
});
'@

        $encrypted = @($tokens.Values | Where-Object { $_ -isnot [string] }).Count -gt 0
        if ($encrypted) {
            $paths = @()
            if ($env:WORKBUDDY_EXE) {
                $paths += $env:WORKBUDDY_EXE
            } else {
                $paths += @(
                    "$env:LOCALAPPDATA\Programs\WorkBuddy\WorkBuddy.exe"
                    "$env:LOCALAPPDATA\WorkBuddy\WorkBuddy.exe"
                    "$env:ProgramFiles\WorkBuddy\WorkBuddy.exe"
                    "${env:ProgramFiles(x86)}\WorkBuddy\WorkBuddy.exe"
                )
                foreach ($root in @(
                    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*"
                    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*"
                    "HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*"
                )) {
                    foreach ($app in (Get-ItemProperty $root -ErrorAction SilentlyContinue)) {
                        if ($app.DisplayName -notlike "*WorkBuddy*") { continue }
                        if ($app.InstallLocation) {
                            $paths += Join-Path $app.InstallLocation "WorkBuddy.exe"
                        }
                        if ($app.DisplayIcon) {
                            $icon = ([string]$app.DisplayIcon -replace ',\s*-?\d+\s*$', '').Trim().Trim('"')
                            if ([IO.Path]::GetFileName($icon) -ieq "WorkBuddy.exe") {
                                $paths += $icon
                            }
                        }
                    }
                }
            }
            $paths = @($paths | Where-Object {
                Test-Path -LiteralPath $_ -PathType Leaf
            } | Select-Object -Unique)
            if ($paths.Count -eq 0) {
                throw '找不到客户端。先运行 $env:WORKBUDDY_EXE="D:\实际路径\WorkBuddy.exe"，再重新运行整段代码'
            }
            $encoded = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($js))
            $payload = $tokens | ConvertTo-Json -Depth 10 -Compress
            $success = $false
            $errors = @()
            foreach ($exe in $paths) {
                $process = New-Object System.Diagnostics.Process
                try {
                    $info = New-Object System.Diagnostics.ProcessStartInfo
                    $info.FileName = $exe
                    $info.Arguments = '-e "eval(Buffer.from(''' + $encoded + ''',''base64'').toString(''utf8''))"'
                    $info.UseShellExecute = $false
                    $info.CreateNoWindow = $true
                    $info.RedirectStandardInput = $true
                    $info.RedirectStandardOutput = $true
                    $info.RedirectStandardError = $true
                    $info.EnvironmentVariables["ELECTRON_RUN_AS_NODE"] = "1"
                    $process.StartInfo = $info
                    [void]$process.Start()
                    $outTask = $process.StandardOutput.ReadToEndAsync()
                    $errTask = $process.StandardError.ReadToEndAsync()
                    $process.StandardInput.Write($payload)
                    $process.StandardInput.Close()
                    if (-not $process.WaitForExit(20000)) {
                        $process.Kill()
                        $errors += "TIMEOUT"
                        continue
                    }
                    $reply = $outTask.GetAwaiter().GetResult() | ConvertFrom-Json
                    [void]$errTask.GetAwaiter().GetResult()
                    if ($reply.ok) {
                        $tokens = @{
                            accessToken = $reply.result.accessToken
                            refreshToken = $reply.result.refreshToken
                        }
                        $success = $true
                        break
                    }
                    $errors += [string]$reply.error
                } catch {
                    $errors += "RUNTIME_UNAVAILABLE"
                } finally {
                    $process.Dispose()
                }
            }
            if (-not $success) { throw ("解密失败：" + ($errors -join ", ")) }
        }

        foreach ($name in @("accessToken", "refreshToken")) {
            $v = $tokens[$name]
            if ($v -isnot [string]) { throw "$name 类型异常" }
            $v = $v.Trim()
            for ($i = 0; $i -lt 2; $i++) {
                try {
                    $decoded = ConvertFrom-Json -InputObject $v -ErrorAction Stop
                    if ($decoded -isnot [string]) { break }
                    $v = $decoded.Trim()
                } catch { break }
            }
            if (-not $v -or $v -cnotmatch '^[A-Za-z0-9._~+/-]+=*$') {
                throw "$name 为空或格式异常"
            }
            $tokens[$name] = $v
        }

        $result = [ordered]@{}
        $result[$phone] = [ordered]@{
            refresh_token = $tokens["refreshToken"]
            access_token = $tokens["accessToken"]
        }
        $text = $result | ConvertTo-Json -Depth 5
        Write-Output $text
        try {
            Set-Clipboard -Value $text -ErrorAction Stop
            Write-Host "JSON 已复制到剪贴板，可以直接粘贴。" -ForegroundColor Green
        } catch {
            Write-Host "自动复制失败，请手动复制上面的输出。" -ForegroundColor Yellow
        }
    } catch {
        Write-Host ("提取失败：" + $_.Exception.Message) -ForegroundColor Red
    }
}
```

</details>

JSON 输出示例：

```json
{
  "你的手机号": {
    "refresh_token": "你的 RT",
    "access_token": "你的 AT"
  }
}
```

> 编辑多账号文件时，只添加或替换对应手机号节点，保留其他账号。`updated` 由签到脚本刷新成功后维护。
>
> 找不到认证文件时，请先确认桌面端已登录。`KEY_MISMATCH` 表示客户端与凭据不匹配；`RUNTIME_UNAVAILABLE` 或 `DECRYPT_FAILED` 时需检查客户端路径、版本及格式支持。反馈错误时不要公开真实 Token。

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

运行第 2 步“首次配置”中的 macOS 或 Windows 命令，复制输出的完整一行；多个账号每行一个：

```text
手机号1:AT1:RT1
手机号2:AT2:RT2
```

第 2 步“后续维护”输出的 JSON 用于编辑 `wb_refresh_tokens.json`，与首次配置命令相互独立，不需要改动 `daily.py`。

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

完成首次环境变量配置并成功运行后，后续不需要再修改 `WORKBUDDY_REFRESH_TOKEN`。登录需要添加或更新的 WorkBuddy 桌面账号，重新运行第 2 步对应的 macOS 或 Windows 命令；终端会直接输出可复制的完整 JSON。

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
