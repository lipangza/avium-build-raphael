# avium-build-raphael

只负责构建 **AviumUI 16.2（Android 16 / lineage-23.2 基线）** 的 **raphael**：
Redmi K20 Pro / Mi 9T Pro（LineageOS 设备代号 `raphael`）。

仓库内不含任何 ROM 源码，只有 CI 与设备清单；源码在 runner 上由 `repo sync` 拉到
`$HOME/android/avium-raphael`。

```
.
├── .github/workflows/build.yml   # 单机型 workflow，机型写死 raphael
├── local_manifests/raphael.xml   # 设备树/内核/vendor 清单
└── README.md
```

## 1. 在 GitHub 上先建空仓库

New repository → 名称 `avium-build-raphael` → 可见性自选 → **不要**勾选
Add README / .gitignore / license（否则首次 push 会冲突）。

## 2. 放入文件并推送

方式 A（先 clone 空仓库）：

```bash
git clone https://github.com/<你的账号>/avium-build-raphael.git
cd avium-build-raphael
cp -r <本目录>/.github ./
cp -r <本目录>/local_manifests ./
cp <本目录>/README.md ./
git add .github local_manifests README.md
git commit -m "init: AviumUI 16.2 raphael self-hosted build"
git push origin main
```

方式 B（本地直接 init）：

```bash
cd <本目录>                      # 即 outputs/avium-build-raphael
git init -b main
git add .
git commit -m "init: AviumUI 16.2 raphael self-hosted build"
git remote add origin https://github.com/<你的账号>/avium-build-raphael.git
git push -u origin main
```

`<本目录>` 指 `outputs/avium-build-raphael`。push 需要**你本人**的凭据
（HTTPS 用带 `repo` 权限的 PAT，或配置 SSH key）。

## 3. runner 前提

| 项 | 要求 |
|---|---|
| 系统 | Ubuntu 22.04 / 24.04 x86_64 |
| 磁盘 | ≥350 GB 空闲（建议 500 GB；源码树 300–400 GB） |
| 内存 | ≥32 GB |
| CPU | ≥16 核 |
| 网络 | github.com、android.googlesource.com |
| 权限 | 可 sudo（装依赖） |
| JDK | 不用手装，AOSP 16 自带 `prebuilts/jdk` |

注册：本仓库 Settings → Actions → Runners → New self-hosted runner → Linux/x64，然后

```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/download/v<版本>/actions-runner-linux-x64-<版本>.tar.gz
tar xzf actions-runner.tar.gz
./config.sh --url https://github.com/<你的账号>/avium-build-raphael --token <页面给出的一次性 token>
sudo ./svc.sh install && sudo ./svc.sh start
```

默认 `runs-on: self-hosted`，任何带该标签的机器都能接单。若想本仓库只用指定机器，
把 `build.yml` **第 34 行**改成 `runs-on: [self-hosted, raphael]`，并在该机器上给
runner 加 `raphael` 标签。

## 4. 触发与产物

Actions → Build AviumUI (raphael) → Run workflow（可选 `clean_build` 删除源码后全量重编）。

- 产物：`AviumUI-raphael-<run_number>`，保留 7 天
- 内容：`*.zip` 刷机包、`boot.img`、`dtbo.img`、`vbmeta.img`、`build-info.txt`
- 同一仓库的两次运行由 `concurrency: avium-raphael` 串行化，不会抢同一源码目录
- 编译日志：job 里 `build.log`（第 10 步 tee），失败时直接看该步骤输出

## 5. 本机型说明

- 设备树来自 **LineageOS 官方 `lineage-23.2`**：`device/xiaomi/raphael` +
  `device/xiaomi/sm8150-common` + `kernel/xiaomi/sm8150` + `hardware/xiaomi`。
- 机型级 vendor `TheMuppets/proprietary_vendor_xiaomi_raphael` 有 `lineage-23.2`；
  **公共级** `proprietary_vendor_xiaomi_sm8150-common` 官方只到 `lineage-22.2`，
  当前清单里用社区镜像 `BayerischeMotorenWerke/...@lineage-23.2`（`raphael.xml` 第 6 个
  project，需你确认；备选 `raphael-resources/...@16.2`）。
- 变体为 `userdebug` + test-keys；**GMS 默认开启**，见下节。
- equuleus（原 `minghao680/avium-build` 的机型）不在本仓库，见该仓库说明。

## 6. GMS（默认开启，`WITH_GMS=true`）

- **来源**：`MindTheGapps/vendor_gapps`，分支 **`baklava`**（Android 16），落到 `vendor/gapps`。
  `arm64/arm64-vendor.mk` 会接着 inherit `common/common-vendor.mk`，与本仓库清单里的路径完全一致。
- **体积**：git 仓约 1.25 GB，checkout 约 1.7 GB（其中 APK 约 440 MB）。
  产物 zip 约 **+250–400 MB**，system 占用约 **+0.8–1.2 GB**。
- **打开方式**：workflow 顶层 `env: WITH_GMS: "true"`（环境变量直接作为 make 变量传给 product 配置）。
  `breakfast` 之后用 `get_build_var WITH_GMS` 校验，不是 `true` 就直接 fail，避免"以为开了其实没开"。
- **raphael 特殊处理**：设备树没有 gapps 行，workflow 步骤 **3b** 在 sync 后幂等追加
  `$(call inherit-product, vendor/gapps/arm64/arm64-vendor.mk)` 到
  `device/xiaomi/raphael/lineage_raphael.mk`。
- **维护代价**：该追加只存在于 runner 的工作树、不入 git；`repo sync --force-sync` 会还原，
  所以每次构建都重新追加一次。若上游把 `lineage_raphael.mk` 改名，步骤 3b 会显式报错
  （不会静默产出无 GMS 的包）。
- **编译时间**：首次约 +10–20 分钟（Soong 处理 GApps APK + preopt），增量约 +3–8 分钟。
- **首次开机**：多出 Google 账号登录 / SetupWizard，需要联网。

## 7. 风险

- **未认证**：开机可能提示 "This device isn't Play Protect certified"，Play 商店部分应用不可安装/更新。
  需要把设备 GSF ID 在 Google 的 device registration 页面注册后才会消失。
- **签名**：MindTheGapps 的内置 APK 自带签名，构建阶段**不需要**你自己的密钥；ROM 本身仍是
  test-keys（不影响 GMS，只是不能接受官方 OTA）。
- **上游不支持**：AviumUI 清单默认不带 GApps，属"可自行加入"而非官方组合；若首次开机卡在开机动画，
  先看 `logcat` 的 `privapp-permissions` 报错。
- **分区**：若 `brunch` 报 `not enough space`，需要调设备树的
  `BOARD_SUPER_PARTITION_SIZE`（这是唯一需要改设备树的高风险点）。
