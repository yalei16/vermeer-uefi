# vermeer-uefi

Redmi K70 / POCO F6 Pro（**vermeer**, SM8550）的 Project Mu（Mu-Silicium）UEFI 移植骨架。

## 这是什么

主线的 `sm8550-xiaomi-vermeer.dtb` 塞进 `vendor_boot` 会被这台机的 ABL 拒绝（实测：
换 board-id、拼 8 条 FDT 表、换 ramdisk 都不行，只有原厂 DTB 能过）。UEFI 路线绕开这一层：
BootShim 顶在 `boot` 分区里，ABL 只看到一个"内核"，UEFI 自己建 ACPI/DT 再把控制权交给 Linux。

本仓库不包含 Mu-Silicium 源码，只包含**设备相关的覆盖文件**：

```
overlay/Platforms/Xiaomi/vermeerPkg/   ← 从 XEC fork 的 nuwaPkg（小米 13 Pro，同为 SM8550）派生
overlay/Resources/Configs/vermeer.toml
overlay/Resources/DTBs/vermeer.dtb     ← 我们编译的 sm8550-xiaomi-vermeer.dtb（138,605 B）
.github/workflows/build.yml            ← CI：拉 Mu-Silicium + 覆盖 + 编译 + 出 artefact
```

## 已改的

| 项目 | 值 |
|---|---|
| PLATFORM_NAME / GUID | `vermeer` / `0B87A5F7-F1D2-52EF-A1AD-AD6F23B229DA` |
| SMBIOS | Xiaomi / Redmi K70 / vermeer / K70_vermeer |
| 帧缓冲 | 1440 × 3200 × 32（K70 与 13 Pro 同分辨率） |
| FdtBlob | `sm8550-xiaomi-vermeer.dtb` |
| DXE/APRIORI/RAW 路径 | `Binaries/nuwa` → `Binaries/vermeer` |

## 已知缺口（重要）

1. **`Binaries/vermeer` 还不存在。** Mu-Silicium 的 `Binaries/` 是子模块
   （`Project-Silicium/Device-Binaries`），里面是**从设备原厂 UEFI（abl.img）里提取出来的
   DXE 驱动**，每个机型一份，79 个机型里没有 vermeer。
   - 现阶段 CI 默认用 **nuwa 的二进制顶替**（同 SoC、同厂商），只为先把镜像编出来、看能不能点亮；
   - 要真正可用，需要从 **vermeer 的 `abl_a` 分区**提取 DXE 驱动（UEFIExtract），替换 `Binaries/vermeer`。
2. **内存映射与 ConfigurationMap** 仍沿用 nuwa 的表（`MemoryMapLib.c` / `ConfigurationMapLib.c`），
   其中 `PcdSchedulerInterfaceAddr=0xA703A948`、`PcdDtbExtensionAddr=0xA703A0C8` 是 XBL 提供的地址，
   两台机 XBL 版本不同就可能对不上 → 首次点亮后要按串口/UEFI 日志修。
3. 面板：`USE_CUSTOM_DISPLAY_DRIVER=1`，panel 由 DXE 里的 DALSYS 走，属于上面第 1 条的二进制依赖。

## 构建

```bash
# GitHub Actions
Actions → build-vermeer-uefi → Run workflow
# 产物：artefact `vermeer-uefi` 里的 Mu-vermeer-*.img / SILICIUM_UEFI.fd*
```

本地编译需要 `git clone --recursive`（约 5 GB）+ Linux 工具链，见 `Building.md`。

## 参考

- Mu-Silicium: https://github.com/Project-Silicium/Mu-Silicium
- Device-Binaries: https://github.com/Project-Silicium/Device-Binaries
- XEC 主线内核（vermeer DTS/固件）: https://github.com/Xlie-Electronic-Customs/linux
