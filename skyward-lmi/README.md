# Skyward for Redmi K30 Pro (lmi) — A17 / OS4

把这个补丁打在 **skyward** 上游（Rust recovery，github.com/kirklin/skyward）上，用底包里的
内核和 ramdisk 打包，就得到本仓库 Release 里的镜像。

## 这个补丁做了什么

| 部分 | 内容 |
|---|---|
| `skyward-pack/src/image.rs` | boot header v1/v2 读写（`Legacy{header_page,dtbo,dtb,dtb_gap}`）、`adopt_kernel`、dtb 长度回写 |
| `skyward-pack/src/ramdisk.rs` | gzip + LZ4 两种 ramdisk 压缩 |
| `skyward-pack/src/main.rs` | `--kernel` / `--kernel-blob` / `--custom-ramdisk` / `--keep-cmdline` / `--tools`；把 TWRP `init.recovery.usb.rc` 里的 `mtp.gs0` 换成 `ffs.mtp` |
| `skyward-vab` | `Slot::Only`：单槽位（A-only）设备的槽位表示 |
| `skyward-install/src/image.rs` | 稀疏镜像（super.img）识别 + 展开写入；分区候选改为宽表并按存在性过滤 |
| `skyward-crypto/src/data.rs` | `/data` 一次 `rw` 挂载（f2fs 才能回放 checkpoint）；失败后 `fsck.f2fs -a` 再试；明文挂载后按 TWRP 的做法探测 `/data/unencrypted/key/version` 判断是否 FBE |
| `skyward-crypto/src/keymaster_hidl.rs` | Keymaster 4.0 **HIDL** 客户端（hwbinder），目前只有 `getHardwareInfo` |
| `skyward/src/avb.rs` | 生成 flags=3 的空 vbmeta 并写入 `vbmeta / vbmeta_system / vbmeta_vendor` |
| `skyward-ui` | A-only 上隐藏槽位相关界面与文案、备份页不再提 init_boot/vendor_boot、高级页去掉系统更新、新增「禁用 AVB 2.0」、截屏按 TWRP 的目录与命名 |
| `devices/lmi.toml` | lmi 的机型描述：1080x2400/440、A-only、aw8697 振动、qseecomd + keymaster@4.0 + gatekeeper@1.0 三个服务、触控黑名单 hbtp_vm |

## 打包命令

```bash
skyward-pack recovery \
  --stock  recovery.img                      \  # 底包 recovery.img（取 dtbo/dtb/cmdline）
  --custom-ramdisk                           \  # 用 --stock 的 ramdisk 作为底座
  --kernel /path/to/boot.img                 \  # 或 --kernel-blob Image
  --keep-cmdline --kernel-blob kernels/miui/Image \
  --binary target/aarch64-linux-android/release/skyward \
  --config devices/lmi.toml \
  --font resources/fonts/MiSansVF.ttf \
  --mono resources/fonts/KirkMono-Regular.ttf \
  --mono-license resources/fonts/KirkMono-OFL.txt \
  --out skyward-lmi.img
```

## 已知限制

- `/data` 必须是**未加密**的（用「解密包」的 ROM），否则只能提示“这是加密设备”。lmi 是 keymaster 4.0/4.1 **HIDL**，skyward 上游只实现了 KeyMint AIDL，真解密需要补齐 keymaster HIDL 的密钥链（`generateKey` / `begin` / `update` / `finish`）。
- WLAN 未启用：vendor 里的 `qca_cld3_*.ko` 与内核 vermagic 绑定，且还需要 `cnss-daemon`。`devices/lmi.toml` 里保留了配置样例。
