# DingTalk 加签支持实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 为 luci-app-pushbot 的钉钉推送增加可选的 `secret` 配置，并在填写后自动生成 `timestamp` 与 `sign`。

**架构：** 在 LuCI 配置页和 UCI 默认配置中新增 `dd_secret` 字段；在主推送脚本中新增局部签名逻辑，仅当当前推送模式为钉钉且 `dd_secret` 非空时，对原 webhook URL 追加动态签名参数。其余推送渠道和未填写 secret 的旧配置保持不变。

**技术栈：** LuCI CBI（Lua）、OpenWrt shell、UCI、curl、openssl/base64

---

## 文件结构

- 修改：`luasrc/model/cbi/pushbot/setting.lua`
  - 职责：定义 LuCI 配置页面中的钉钉 `dd_secret` 输入项。
- 修改：`root/etc/config/pushbot`
  - 职责：为 UCI 提供 `dd_secret` 默认配置项。
- 修改：`root/usr/bin/pushbot/pushbot`
  - 职责：读取 `dd_secret`，生成钉钉签名，并仅在钉钉模式下拼接到发送 URL。
- 测试：手工验证脚本 URL 生成行为
  - 仓库内没有自动化测试基础设施，本次使用最小可验证的手工命令检查。

### 任务 1：新增 LuCI 与 UCI 配置项

**文件：**
- 修改：`luasrc/model/cbi/pushbot/setting.lua`
- 修改：`root/etc/config/pushbot`

- [ ] **步骤 1：在 UCI 默认配置中加入空的 `dd_secret`**

```conf
config pushbot 'pushbot'
	option pushbot_enable '0'
	option sleeptime '60'
	option pushbot_ipv6 '0'
	option pushbot_up '1'
	option pushbot_down '1'
	option cpuload_enable '1'
	option cpuload '2'
	option temperature_enable '0'
	option dd_secret ''
```

- [ ] **步骤 2：在钉钉 webhook 输入框后新增 `dd_secret` 输入框**

```lua
a=s:taboption("basic", Value,"dd_secret",translate('加签 Secret'), translate("钉钉机器人加签密钥（secret）").."<br>开启钉钉机器人加签后填写，留空则按旧方式发送<br><br>")
a.rmempty = true
a.password = true
a:depends("jsonpath","/usr/bin/pushbot/api/dingding.json")
```

- [ ] **步骤 3：检查显示依赖关系**

运行后确认：
- 仅在 `jsonpath=/usr/bin/pushbot/api/dingding.json` 时显示
- 切换到其他推送渠道时不显示该字段

预期：LuCI 钉钉配置区域同时出现 `dd_webhook` 和 `dd_secret` 两个字段。

- [ ] **步骤 4：Commit**

```bash
git -C "/home/wenha/project/luci-app-pushbot" add luasrc/model/cbi/pushbot/setting.lua root/etc/config/pushbot
git -C "/home/wenha/project/luci-app-pushbot" commit -m "feat: add dingtalk secret config"
```

### 任务 2：实现钉钉签名 URL 生成

**文件：**
- 修改：`root/usr/bin/pushbot/pushbot`

- [ ] **步骤 1：将 `dd_secret` 纳入配置读取列表**

在 `read_config()` 的 `get_config` 参数中，把：

```sh
"jsonpath" "dd_webhook" "we_webhook"
```

改成：

```sh
"jsonpath" "dd_webhook" "dd_secret" "we_webhook"
```

- [ ] **步骤 2：新增钉钉签名辅助函数**

在 `diy_send()` 之前加入局部函数，负责：
- 生成毫秒级时间戳
- 用 `secret` 计算 `HMAC-SHA256`
- Base64 编码
- 将 `+`、`/`、`=` 等字符做 URL 编码

```sh
function urlencode_sign(){
	echo -n "$1" | sed \
		-e 's/%/%25/g' \
		-e 's/+/\%2B/g' \
		-e 's#/#%2F#g' \
		-e 's/=/%3D/g'
}

function build_dingtalk_signed_url(){
	local base_url="$1"
	[ -z "$dd_secret" ] && echo "$base_url" && return
	local timestamp=$(date +%s%3N 2>/dev/null)
	[ -z "$timestamp" ] && timestamp="$(($(date +%s) * 1000))"
	local string_to_sign="${timestamp}\n${dd_secret}"
	local sign=$(printf '%b' "$string_to_sign" | openssl dgst -sha256 -hmac "$dd_secret" -binary | openssl base64 -A)
	sign=$(urlencode_sign "$sign")
	echo "${base_url}&timestamp=${timestamp}&sign=${sign}"
}
```

- [ ] **步骤 3：仅在钉钉模式下改写发送 URL**

在 `diy_send()` 中，读取 `diyurl` 后立即加入条件分支：

```sh
local diyurl=`/usr/bin/jq -r .url ${3}` && local diyurl=`eval echo ${diyurl}`
[ "$jsonpath" = "/usr/bin/pushbot/api/dingding.json" ] && [ -n "$dd_secret" ] && diyurl=$(build_dingtalk_signed_url "$diyurl")
```

预期：
- 非钉钉模式：URL 完全不变
- 钉钉 + 空 secret：URL 完全不变
- 钉钉 + 非空 secret：URL 末尾新增 `timestamp` 与 `sign`

- [ ] **步骤 4：检查命令依赖是否局部失败可见**

运行：

```bash
grep -n "build_dingtalk_signed_url\|urlencode_sign\|dd_secret" "/home/wenha/project/luci-app-pushbot/root/usr/bin/pushbot/pushbot"
```

预期：能看到新增函数、配置读取项、以及 `diy_send()` 中的条件调用。

- [ ] **步骤 5：Commit**

```bash
git -C "/home/wenha/project/luci-app-pushbot" add root/usr/bin/pushbot/pushbot
git -C "/home/wenha/project/luci-app-pushbot" commit -m "feat: add dingtalk signed webhook support"
```

### 任务 3：验证兼容性与输出

**文件：**
- 修改：无
- 测试：`root/usr/bin/pushbot/pushbot`

- [ ] **步骤 1：检查工作区改动**

运行：

```bash
git -C "/home/wenha/project/luci-app-pushbot" diff -- luasrc/model/cbi/pushbot/setting.lua root/etc/config/pushbot root/usr/bin/pushbot/pushbot
```

预期：仅包含 `dd_secret` 配置和钉钉签名逻辑相关变更。

- [ ] **步骤 2：验证未填写 secret 的兼容性**

运行：

```bash
grep -n 'dd_secret' "/home/wenha/project/luci-app-pushbot/root/etc/config/pushbot" && grep -n 'dd_secret' "/home/wenha/project/luci-app-pushbot/luasrc/model/cbi/pushbot/setting.lua"
```

预期：配置项已存在，且允许留空；逻辑层只有在非空时才启用签名。

- [ ] **步骤 3：语法检查 shell 脚本**

运行：

```bash
sh -n "/home/wenha/project/luci-app-pushbot/root/usr/bin/pushbot/pushbot"
```

预期：无输出，退出码 0。

- [ ] **步骤 4：确认计划目标已覆盖**

逐项检查：
- LuCI 已新增 `dd_secret`
- UCI 默认配置已新增 `dd_secret`
- 钉钉模式 + secret 非空时会附加 `timestamp` 与 `sign`
- secret 为空时保持旧行为
- 其他渠道不受影响

预期：全部满足。

- [ ] **步骤 5：Commit**

```bash
git -C "/home/wenha/project/luci-app-pushbot" status --short
```

预期：若前两次 commit 已完成，这里应只剩计划文档或无额外实现改动。
