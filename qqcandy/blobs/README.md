# 二进制组件（qqcandy）

这里的文件按**设备上的分区路径**摆放，`blobs/vendor/` 对应 `/vendor`，
`blobs/vendor_dlkm/` 对应 `/vendor_dlkm`。注意这台机器上 vendor 走 KernelSU
元模块（systemless overlay）、vendor_dlkm 走 recovery 刷镜像，两者都不直接覆盖分区。

许可提醒：ARM / MediaTek 专有二进制，不适用本仓库的 GPL/MIT 许可，详见
[../../LICENSES.md](../../LICENSES.md)。

## vendor（6 个）→ `/vendor/lib64/`

qqcandy 底包缺的、或需要改过的用户态文件。xaga 侧的 10 个 r54 库不在这里，
由 `scripts/build_ksu_module.sh` 从 [`../../xaga/blobs/vendor/`](../../xaga/blobs/vendor/) 取。

| 文件 | 大小 | md5 | 说明 |
|---|---|---|---|
| `libc++.so` | 1,154,016 | `da194e209887` | 底包 VNDK libc++ 缺 `__libcpp_verbose_abort` 等 257 个符号 |
| `android.hardware.graphics.common-V6-ndk.so` | 31,512 | `c1bca7a90685` | `libgpud.so` 的 `DT_NEEDED` |
| `android.hardware.graphics.allocator-V2-ndk.so` | 44,416 | `ff5a37d861b0` | 同上（底包是 HIDL gralloc，没有 AIDL 客户端） |
| `android.hardware.common-V2-ndk.so` | 18,856 | `2647eaf1cb78` | 上面 allocator-V2 的依赖 |
| `libged.so` | 101,944 | `0a2d5f399b6b` | r54 版；`atrace_async_for_track_*_body` 已**等长**改指 `atrace_begin_body`/`atrace_end_body` |
| `libged.orig.so` | 101,944 | `f28930eddfa5` | 上面那个的改前备份（md5 与 xaga 的 `libged.so` 相同） |

> 新增的 3 个 `.so` 与 `libc++.so` 走 `vendor_file` 标签；`libged.so` / `libgpud.so`
> 必须是 `same_process_hal_file`，见 `../docs/PITFALLS.zh-CN.md` 第 5 条。

## vendor_dlkm（3 个）→ `/vendor_dlkm/lib/modules/`

只换 r54 的三个 Mali 模块。**`ged.ko` / `gpueb.ko` 保持原厂**——r54 的 `ged.ko`
在这台内核上装不了，原厂的够用（`../docs/PITFALLS.zh-CN.md` 第 3 条）。

| 文件 | 大小 | md5 |
|---|---|---|
| `mali_kbase_mt6895.ko` | 2,571,656 | `f4a53b364c0c` |
| `mali_mgm_mt6895.ko` | 28,704 | `29d2e912f045` |
| `mali_prot_alloc_mt6895.ko` | 24,240 | `9e474d721368` |

`scripts/build_dlkm_image.py` 默认从这个目录（`--metadir`）取文件替换底包
`vendor_dlkm.img` 里的同名模块。

## gpueb（物理分区，不在上面两棵树里）

| 文件 | 大小 | md5 | 说明 |
|---|---|---|---|
| `gpueb_stock_restore.img` | 430,080 | `dbf2f6d5faa8` | 原厂 gpueb 镜像，**回滚用**；补齐到 1 MB 后 md5 `4f5ba918…` |

这台机器的 gpueb 固件带 RSA 签名（尾部 `cert1` 段），改一个字节就起不来，
所以没有 950 MHz 版本——见 `../docs/PITFALLS.zh-CN.md` 第 6 条。
