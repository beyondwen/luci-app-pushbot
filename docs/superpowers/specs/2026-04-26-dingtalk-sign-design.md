# DingTalk 加签支持设计

## 背景
当前 `luci-app-pushbot` 的钉钉推送仅支持填写 `access_token`，通过固定 URL 模板向钉钉自定义机器人发送消息。对于开启“加签”安全设置的钉钉机器人，现有实现无法附加动态生成的 `timestamp` 与 `sign` 参数，因此推送会失败。

## 目标
为钉钉推送增加可选的 `secret` 配置项：

- 用户可在 LuCI 页面填写钉钉机器人的加签 `secret`
- 当 `secret` 非空时，发送前自动生成 `timestamp` 与 `sign`
- 当 `secret` 为空时，保持现有行为，继续仅使用 `access_token`
- 不影响其他推送渠道

## 非目标
- 不修改其他平台（企业微信、飞书、PushPlus 等）的推送逻辑
- 不改变现有 `dd_webhook` 的输入方式，仍然只填写 `access_token`
- 不引入新的强制依赖或破坏旧配置

## 设计

### 1. 配置层
在以下位置新增 `dd_secret`：

- `luasrc/model/cbi/pushbot/setting.lua`
- `root/etc/config/pushbot`

LuCI 页面表现：
- 仅在钉钉推送模式下显示
- 文案说明这是“钉钉机器人加签密钥（secret）”
- 允许留空

UCI 默认值：
- `option dd_secret ''`

### 2. 推送逻辑
在主脚本 `root/usr/bin/pushbot/pushbot` 中：

- 将 `dd_secret` 纳入 `read_config()` 的配置读取列表
- 在钉钉推送发送前判断：
  - 当前推送模式为 `/usr/bin/pushbot/api/dingding.json`
  - `dd_secret` 非空
- 满足条件时，基于钉钉官方规则生成：
  - `timestamp`：毫秒级 Unix 时间戳
  - `stringToSign`：`timestamp + "\n" + secret`
  - `sign`：`HMAC-SHA256(stringToSign, secret)` → Base64 → URL 编码
- 将生成的参数追加到原 webhook URL：
  - `&timestamp=<timestamp>&sign=<urlencoded_sign>`

### 3. 兼容性
- 若 `dd_secret` 未填写：使用现有 URL，不附加签名参数
- 现有用户无需修改配置即可继续工作
- 修改仅对钉钉推送路径生效

## 实现要点
- 尽量复用系统常见工具完成签名，如 `openssl`、`base64` 和基础 URL 编码
- 如 shell 中处理 Base64 换行或 URL 编码存在兼容性问题，优先以脚本内小范围辅助函数解决，不扩散到其他推送渠道
- 仍保持 `dingding.json` 作为 URL 模板来源，避免重构整个发送框架

## 测试
至少验证以下场景：

1. **无 secret 的钉钉配置**
   - 生成的请求 URL 与当前行为一致
2. **有 secret 的钉钉配置**
   - 请求 URL 包含 `timestamp` 和 `sign`
3. **非钉钉渠道**
   - 推送 URL 不受影响
4. **空 secret / 未配置 secret**
   - 不报错，保持旧逻辑

## 风险与应对
- **风险：** 目标 OpenWrt 环境中签名命令可用性不一致
  - **应对：** 采用最常见命令组合，保持实现局部化，必要时退回更兼容的 shell 写法
- **风险：** URL 编码处理不正确导致钉钉验签失败
  - **应对：** 对 `sign` 单独做编码，避免对整个 URL 二次编码

## 结论
采用“可选 secret + 自动加签”的最小兼容方案，在不破坏现有用户配置的前提下，为钉钉自定义机器人增加加签支持。
