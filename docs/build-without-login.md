# 免登录构建补丁固件

原版构建流程需要登录小米账号获取 OTA 固件下载链接，但小米账号经常因为安全验证（`securityStatus: 128`）导致登录失败。

本教程通过手动构造 OTA 链接，完全绕过小米账号登录。

## 原理

构建脚本 `src/ota.ts` 支持通过 `OTA` 环境变量直接传入设备信息，跳过登录：

```javascript
if (process.env.OTA) {
  ota = JSON.parse(process.env.OTA);
}
```

而小米 OTA 接口本身不需要鉴权，只需要 `model`、`version` 和一个简单的 MD5 签名。

## 使用方法

### 1. 克隆代码

```bash
git clone https://github.com/idootop/open-xiaoai.git
cd open-xiaoai/packages/client-patch
```

### 2. 生成 OTA 链接

运行以下 Python 脚本生成 OTA JSON（修改 `model` 和 `version` 为你的设备信息）：

```bash
python3 -c '
import hashlib, base64, time, json

# ===== 修改这两个参数 =====
model = "OH2P"       # OH2P 或 LX06
version = "1.60.2"   # 你的当前固件版本
# =========================

channel = "release"
t = int(time.time() * 1000)
s = f"channel={channel}&filterID=&locale=zh_CN&model={model}&time={t}&version={version}&8007236f-a2d6-4847-ac83-c49395ad6d65"
code = hashlib.md5(base64.b64encode(s.encode())).hexdigest()
url = f"http://api.miwifi.com/rs/grayupgrade/v2/{model}?model={model}&version={version}&channel={channel}&filterID=&locale=zh_CN&time={t}&s={code}"
print(json.dumps({"model": model, "version": version, "url": url}))
'
```

输出类似：

```json
{"model": "OH2P", "version": "1.60.2", "url": "http://api.miwifi.com/rs/grayupgrade/v2/OH2P?..."}
```

### 3. 构建固件

将上一步的输出作为 `OTA` 环境变量传入 Docker：

```bash
OTA=$(python3 -c '
import hashlib, base64, time, json
model = "OH2P"
version = "1.60.2"
channel = "release"
t = int(time.time() * 1000)
s = f"channel={channel}&filterID=&locale=zh_CN&model={model}&time={t}&version={version}&8007236f-a2d6-4847-ac83-c49395ad6d65"
code = hashlib.md5(base64.b64encode(s.encode())).hexdigest()
url = f"http://api.miwifi.com/rs/grayupgrade/v2/{model}?model={model}&version={version}&channel={channel}&filterID=&locale=zh_CN&time={t}&s={code}"
print(json.dumps({"model": model, "version": version, "url": url}))
')

docker run --rm \
    --platform linux/amd64 \
    -e "OTA=$OTA" \
    -e SSH_PASSWORD=open-xiaoai \
    -v $(pwd)/assets:/app/assets \
    -v $(pwd)/patches:/app/patches \
    idootop/open-xiaoai:latest
```

> **Apple Silicon 用户注意**：需要在 Docker Desktop 设置中开启 `Use Rosetta for x86_64/amd64 emulation on Apple Silicon`。

### 4. 获取固件

构建成功后，补丁固件在 `assets/` 目录下：

```
assets/mico_all_xxx_版本号/root-patched.squashfs
```

## 一键脚本

将以下内容保存为 `build-no-login.sh`，修改 `MODEL` 和 `VERSION` 后直接运行：

```bash
#!/usr/bin/env bash
set -e

# ===== 修改这两个参数 =====
MODEL="OH2P"        # OH2P 或 LX06
VERSION="1.60.2"    # 你的当前固件版本
PASSWORD="open-xiaoai"  # SSH 登录密码
# =========================

cd "$(dirname "$0")/packages/client-patch"

OTA=$(python3 -c "
import hashlib, base64, time, json
model, version, channel = '$MODEL', '$VERSION', 'release'
t = int(time.time() * 1000)
s = f'channel={channel}&filterID=&locale=zh_CN&model={model}&time={t}&version={version}&8007236f-a2d6-4847-ac83-c49395ad6d65'
code = hashlib.md5(base64.b64encode(s.encode())).hexdigest()
url = f'http://api.miwifi.com/rs/grayupgrade/v2/{model}?model={model}&version={version}&channel={channel}&filterID=&locale=zh_CN&time={t}&s={code}'
print(json.dumps({'model': model, 'version': version, 'url': url}))
")

echo "OTA: $OTA"

docker run --rm \
    --platform linux/amd64 \
    -e "OTA=$OTA" \
    -e "SSH_PASSWORD=$PASSWORD" \
    -v "$(pwd)/assets:/app/assets" \
    -v "$(pwd)/patches:/app/patches" \
    idootop/open-xiaoai:latest

echo ""
echo "✅ 补丁固件:"
find assets -name "root-patched.squashfs" -type f
```

## 查看固件版本

如果不确定当前固件版本，可以在米家 App 中查看，或者直接用 Python 查询最新可用版本：

```bash
python3 -c '
import hashlib, base64, time, urllib.request, json

model = "OH2P"  # OH2P 或 LX06
version = "0.0.0"  # 用一个旧版本号来获取最新 OTA
channel = "release"
t = int(time.time() * 1000)
s = f"channel={channel}&filterID=&locale=zh_CN&model={model}&time={t}&version={version}&8007236f-a2d6-4847-ac83-c49395ad6d65"
code = hashlib.md5(base64.b64encode(s.encode())).hexdigest()
url = f"http://api.miwifi.com/rs/grayupgrade/v2/{model}?model={model}&version={version}&channel={channel}&filterID=&locale=zh_CN&time={t}&s={code}"

data = json.loads(urllib.request.urlopen(url).read())
if data.get("code") == "0" and data.get("data", {}).get("currentInfo"):
    info = data["data"]["currentInfo"]
    print(f"最新版本: {info.get(\"toVersion\", \"未知\")}")
    print(f"文件大小: {info[\"size\"] / 1024 / 1024:.1f} MB")
    print(f"下载链接: {info[\"link\"]}")
'
```
