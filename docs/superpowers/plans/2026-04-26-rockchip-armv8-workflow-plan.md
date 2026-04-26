# Rockchip ARMv8 Workflow 实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 将仓库中的 GitHub Actions OpenWrt 包构建从 x86/64 SDK 切换为适配用户路由器的 rockchip/armv8 SDK。

**架构：** 保持现有 workflow 结构、上传逻辑和运行器不变，只做最小改动：更新 workflow 名称以反映目标平台，并把 SDK 下载地址从 x86/64 改为 OpenWrt 24.10.5 的 rockchip/armv8 SDK。这样 Actions 仍在 GitHub 的 x86_64 runner 上执行，但生成的 ipk 面向用户的 ARM 设备目标。

**技术栈：** GitHub Actions、OpenWrt SDK 24.10.5、YAML

---

## 文件结构

- 修改：`.github/workflows/build-package-onx86.yml`
  - 职责：定义 GitHub Actions 中的 OpenWrt SDK 下载目标、工作流名称及包构建流程。
- 测试：手工静态验证 workflow 文本
  - 本仓库当前没有本地 CI 校验器，使用文件读取与 diff 验证目标字段变更。

### 任务 1：切换 workflow 名称与 SDK 目标

**文件：**
- 修改：`.github/workflows/build-package-onx86.yml`

- [ ] **步骤 1：将 workflow 名称从 x86 改为 rockchip/armv8**

把：

```yaml
name: Build luci-app-pushbot-x86
```

改成：

```yaml
name: Build luci-app-pushbot-rockchip-armv8
```

- [ ] **步骤 2：将 SDK_URL 改为 OpenWrt 24.10.5 rockchip/armv8 SDK**

把：

```yaml
SDK_URL: https://downloads.openwrt.org/releases/24.10.5/targets/x86/64/openwrt-sdk-24.10.5-x86-64_gcc-13.3.0_musl.Linux-x86_64.tar.zst
```

改成：

```yaml
SDK_URL: https://downloads.openwrt.org/releases/24.10.5/targets/rockchip/armv8/openwrt-sdk-24.10.5-rockchip-armv8_gcc-13.3.0_musl.Linux-x86_64.tar.zst
```

- [ ] **步骤 3：保留其余 workflow 逻辑不变**

核对以下项没有被误改：
- `runs-on: ubuntu-latest`
- feeds 更新与安装步骤
- `make package/$PackageName/compile V=s`
- artifact / release 上传逻辑

预期：仅目标平台相关字段发生变化，不引入额外行为变更。

- [ ] **步骤 4：Commit**

```bash
git -C "/home/wenha/project/luci-app-pushbot" add .github/workflows/build-package-onx86.yml
git -C "/home/wenha/project/luci-app-pushbot" commit -m "ci: switch package build to rockchip armv8"
```

### 任务 2：验证 workflow 变更内容

**文件：**
- 修改：无
- 测试：`.github/workflows/build-package-onx86.yml`

- [ ] **步骤 1：读取 workflow 关键字段**

运行：

```bash
grep -nE '^(name:|  SDK_URL:)' "/home/wenha/project/luci-app-pushbot/.github/workflows/build-package-onx86.yml"
```

预期：输出包含：
- `name: Build luci-app-pushbot-rockchip-armv8`
- `SDK_URL: https://downloads.openwrt.org/releases/24.10.5/targets/rockchip/armv8/...`

- [ ] **步骤 2：查看工作区 diff**

运行：

```bash
git -C "/home/wenha/project/luci-app-pushbot" --no-pager diff -- .github/workflows/build-package-onx86.yml
```

预期：diff 只展示 workflow 名称和 SDK_URL 的修改。

- [ ] **步骤 3：确认目标与用户设备匹配**

人工核对用户提供的设备信息：
- `release.target = rockchip/armv8`
- `arch aarch64_generic`
- `arch aarch64_cortex-a53`

预期：workflow 目标与设备 target/subtarget 一致。

- [ ] **步骤 4：Commit**

```bash
git -C "/home/wenha/project/luci-app-pushbot" status --short
```

预期：若已 commit，则这里只显示未提交的其他工作；否则应仅剩本次 workflow 改动。
