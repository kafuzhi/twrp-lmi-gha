# TWRP for Redmi K30 Pro (lmi) — GitHub Actions 构建

用 GitHub Actions 出 `recovery.img`，本机不落源码。设备树用 skkk 的
[sekaiacg/android_device_xiaomi_umi_TWRP](https://github.com/sekaiacg/android_device_xiaomi_umi_TWRP)
（`android-13` 分支，同树包含 umi/cmi/cas/apollo/**lmi** 五个 target），TWRP 源码用官方 minimal
manifest 的 **twrp-12.1**（= TWRP 3.7.1_12，skkk 自己发布包的基线；他的 TWRP fork 也只有
`android-11`/`android-12.1` 两个分支）。

上游没有 A15/A16/A17 分支（`TeamWin/android_bootable_recovery` 与 gerrit.twrp.me 都止于
`android-14.1`），所以这里不假装「A17 版 TWRP」。`twrp_branch` 输入留了口子，想试 `twrp-14.1`
可以自己填，但那棵设备树没为它适配，大概率编不过或跑不起来。

## 用法

1. 在 GitHub 建一个空仓库，把本目录推上去（`.github/workflows/build-twrp.yml` 必须在默认分支上）：
   ```bash
   git init -b main && git add -A && git commit -m "twrp lmi ci"
   git remote add origin git@github.com:<you>/<repo>.git && git push -u origin main
   ```
2. 仓库 → **Actions** → `build-twrp` → **Run workflow**，默认参数即可出 lmi 的包。
3. 跑完在 `Artifacts` 里下 `twrp-lmi-twrp-12.1-android-13`，解压得到 `recovery.img`。

## 参数

| 输入 | 默认 | 说明 |
|---|---|---|
| `device` | `lmi` | lmi=K30 Pro / POCO F2 Pro；umi=米10；cmi=米10 Pro；cas=米10 Ultra；apollo=K30S 至尊 |
| `twrp_branch` | `twrp-12.1` | minimal manifest 分支（上游只有 11 / 12.1 / 14 / 14.1） |
| `device_branch` | `android-13` | 设备树分支（`android-11/12/12.1/13`） |
| `decrypt_platform` | 空 | 导出 `DECRYPT_PLATFORM_VERSION`，空则用树里默认的 `PLATFORM_VERSION := 14`；旧 ROM 解密异常时试 `13`/`12` |
| `manifest_url` | 官方 minimal-manifest | 换成 fork 可试 `twrp-15`/`twrp-16`（官方只有 11 / 12.1 / 14 / 14.1） |

**一次 dispatch 出两个包**（矩阵 `variant: [miui, aosp]`，对应 skkk 每代发的两个文件）：

| variant | 导出 | 给谁用 |
|---|---|---|
| `miui` | `TW_USE_FBEV2=false`（`PRODUCT_SHIPPING_API_LEVEL := 29`） | MIUI / HyperOS，`..._A15-lmi-skkk.img` |
| `aosp` | `TW_USE_FBEV2=true`（`:= 30`） | AOSP / 类原生（FBEv2 解密），`..._A15_FBEv2-lmi-skkk.img` |
| `skkk_ver` | 空 | `TW_SKKK_VER_CODE`，只影响 `ro.twrp.version.skkk.code` |
| `min_free_gb` | `30` | 清理后可用空间低于此值直接失败，避免跑到 link 阶段才炸（runner 实测：`/` 73 GB，清理后 41 GB） |
| `repo_url` | gerrit.googlesource.com | `repo` 启动器来源；网络不通时换成镜像（如 `https://github.com/GerritCodeReview/git-repo`） |

## 资源与耗时（github-hosted `ubuntu-22.04`，4 vCPU / 16 GB / 约 65–70 GB 可用）

- `repo sync --depth=1`：manifest 有 **1061 个项目**，20–40 分钟；同步完立刻 `rm -rf .repo`，把 6–8 GB 还给 `out/`（一次性构建不需要 repo 元数据）。
- `mka recoveryimage`：4 核约 2–4 小时；job 上限 350 分钟（GitHub 硬上限 6 小时）。两个 variant 是两个并行 job，各自 sync 一份（免费额度内）。
- runner 磁盘实测有两种规格：`/` 73 GB（清理后 41 GB）和 146 GB（清理后 113 GB），闸门 30 GB 两种都过。
- 想要更快/更稳：换 larger runner（8 核以上），或自建 runner 挂一块大盘。
- 不要用 `ubuntu-24.04`（gcc-13 / glibc 2.39 与 AOSP 12.1 的 host 工具冲突）；`ubuntu-20.04` 已退役。

## 几个已知坑（已在 workflow 里处理）

- **产物目录永远是 `out/target/product/umi/`**：`twrp_lmi.mk` 里 `PRODUCT_DEVICE := umi`，五个 target 共用这个 device 名。workflow 用 `*/recovery.img` 兜住，不要按 lmi 去拼路径。
- **一次只能编一个设备**：切换 `device` 必须重新跑（job 是干净环境，天然满足）；本地编的话要 `make clean`，否则镜像串。
- **分区上限 128 MB**：`BOARD_RECOVERYIMAGE_PARTITION_SIZE := 134217728`，`Report image` 步骤会硬校验，超了就红。
- **ccache 故意关掉**：GH-hosted 的缓存 10 GB 上限装不下 AOSP，开着只会占盘。
- **`set -e` 与 `[ ... ] && export ...`**：条件导出全部写成 `if ... then ... fi`，否则空值会让整步退出。

## 刷机

```bash
fastboot flash recovery recovery.img     # 或者先 fastboot boot recovery.img 试跑
```

lmi 是 A-only + 动态分区（super 9.1 GB）、FBE + metadata wrappedkey 解密，设备树上
`BOARD_USES_QCOM_FBE_DECRYPTION` / `TW_INCLUDE_CRYPTO_FBE` / `TW_INCLUDE_FBE_METADATA_DECRYPT`
都开着。**这套 CI 只保证产出镜像，不保证在你的具体 ROM 上能解密/挂载** —— 解密靠的是树里
vendored 的 keymaster/gatekeeper blobs，跟 ROM 代次对不上就会失败。真机验证只能在你的机器上做。

## 已验证 / 未验证

已验证：`twrp-12.1` 分支存在且 manifest 正常（含 `twrp-default.xml`、1061 个项目）；设备树
`android-13` 分支含 `twrp_lmi.mk`；workflow YAML 可解析、每个 `run` 脚本 `bash -n` 通过。

未验证：完整 sync + 编译（本机只有 70 GB 剩余盘、且到 gerrit.googlesource.com 不通），所以
第一次在 CI 上跑请盯着 `Build recovery` 那一步的日志。
