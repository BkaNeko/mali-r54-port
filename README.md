# mali-r54-port

把 MediaTek **MT6895 (Mali-G610 MC6)** 的 GPU 栈从原厂 **r32p1** 移植到 ARM 官方 **r54p1**
的记录仓库。两台同平台设备各占一个目录，落地方式完全不同，请按机器看对应的 README。

> 这是**移植记录仓库**，不是可直接刷入的固件。内容为补丁、改动后的源文件、验证工具与实机结论。

## 两台机器

| 目录 | 设备 | 系统 / 内核 | 落地方式 | 备注 |
|---|---|---|---|---|
| [`xaga/`](xaga/README.md) | Redmi Note 11T Pro / POCO X4 GT（xaga） | Android 17 / LineageOS 24，5.10.269 | 重打 `vendor` / `vendor_dlkm` 后写 `super` | 主力机；GED+gpueb 全套 backport，950 MHz 可用 |
| [`qqcandy/`](qqcandy/README.md) | oppo/ossi（qqcandy） | Android 16，5.10.236 | 内核模块写真分区 + KernelSU 元模块 overlay | 固件带 RSA 签名，950 MHz 做不到 |

两边是**同一套 r54p1 驱动栈**，但 xaga 的 GED / gpueb 都已升到 6.12 backport 版，
qqcandy 保持原厂 `ged.ko` / `gpueb.ko`（原因见 [`qqcandy/docs/PITFALLS.zh-CN.md`](qqcandy/docs/PITFALLS.zh-CN.md) 第 3 条）。

## 目录结构

```
xaga/                      xaga 的完整移植记录
  blobs/                   用户态库 + vendor_dlkm 内核模块（专有二进制）
  firmware/                mali_csffw.bin、gpueb 原厂 / 950MHz 固件
  patches/                 GED、gpueb 的 6.12→5.10 补丁与 Mali 源文件
  files/                   源码快照、build.prop 属性、insmod 清单
  docs/                    移植步骤、可刷 super.img、950MHz、GPUEB 复盘、DVFS 修复
  tools/                   填充率 / OPP 探针 / 固件改表等小工具
  README.md                结论、踩坑与调频架构（先读这个）
qqcandy/                   oppo/ossi 的移植记录（结构同上，另有 scripts/）
LICENSES.md                许可：补丁 GPL-2.0，工具 MIT，二进制专有
```

## 从哪读起

1. 对应机器的 `README.md` —— 结果表、目录职责、关键坑。
2. `xaga/docs/PORTING.zh-CN.md` —— 照着做的移植步骤。
3. `xaga/docs/DVFS-FIX.zh-CN.md` —— 调频这条线上的全部结论（含被推翻的旧结论、附录 A~G）。
4. `xaga/docs/HANDOFF-xaga-dvfs.zh-CN.md` —— 时间点交接快照，部分内容已被 DVFS-FIX 更新。

## 许可

见 [`LICENSES.md`](LICENSES.md)。`xaga/blobs/`、`xaga/firmware/` 下的 ARM / MediaTek 二进制为专有件，
不适用本仓库的 GPL/MIT 许可，仅作移植对照与完整性记录。

## 免责声明

修改内核驱动与分区存在变砖风险。刷机前请备份原厂镜像并确认回滚方式。作者不对任何设备损坏负责。
