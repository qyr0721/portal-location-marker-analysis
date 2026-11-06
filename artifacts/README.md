# 产物文件（Artifacts）

本目录存放本次研究的实际产出文件，均为针对 `moe.fuqiuluo.portaldev`（PortalDev 2.0.7，build r92，commit `587cfab`）的补丁产物。文件校验和（SHA-256）列于下表，供完整性校验。

| 文件 | 说明 | SHA-256 |
| --- | --- | --- |
| `PortalDev_patched_signed.apk` | **已签名可直接安装**的补丁包（RSA-2048，别名 `portalpatch`，v1+v2 签名） | `d12827a647ee85dfe85848946d51ac4e1b2759979dc5ffff83a54cc0e0c9c7a8` |
| `PortalDev_patched_fixed_unsigned.apk` | 未签名补丁包（校验和已修正），供使用自有密钥重新签名 | `b096994c752759095a75727b44bd6f88fc5d2968ab79a0485855a0d9cac5fcde` |
| `classes2_patched_fixed.dex` | 补丁后的 `classes2.dex`（含 2 字节分支补丁与重算后的头部校验和） | `44458beff65431423d6a7c19b31a4c230e3af79fd70ed9285c941730799f932b` |
| `PortalDev_patched_signed.apk.idsig` | `apksigner` 生成的 v4 签名附属文件，与签名版 APK 配对 | `ec2460741393ce9bfc7eb02ea8aeacdc2f202bd985d7f603c3ce7ac703c34b13` |

## 使用说明

### 直接安装（已签名版）

```bash
adb install --no-incremental PortalDev_patched_signed.apk
```

安装后需在 LSPosed 中启用 `portaldev` 模块并勾选 `system` 作用域，随后重启设备使钩子注入 `system_server`。

### 使用自有密钥重签名（未签名版）

```bash
zipalign -p -f 4 PortalDev_patched_fixed_unsigned.apk aligned.apk
apksigner sign --ks <your.keystore> --out PortalDev_patched_custom.apk aligned.apk
```

## 补丁要点回顾

- **补丁位置**：`classes2.dex` @ 文件偏移 `0x67b7fa`
- **补丁内容**：`39 0D 2D 00`（`if-nez v13, +0x2d`）→ `29 00 2D 00`（`goto/16 +0x2d`）
- **效果**：使 `portal.enable` / `allToggles` / `is_mock` 三个标记的写入指令成为不可达死代码

> **安全提示**：签名所用私钥 `portal_patch.keystore` 未随仓库上传。请勿将签名私钥提交至公开仓库；如需再次签名，请使用自行保管的密钥。
