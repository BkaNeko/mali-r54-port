# 二进制组件

移植所需的全部二进制件。**目录结构就是设备上的分区结构**，可以直接覆盖：
`blobs/vendor/` 对应 `/vendor`，`blobs/vendor_dlkm/` 对应 `/vendor_dlkm`。

许可提醒：这些是 ARM 和 MediaTek 的专有二进制，不适用本仓库的 GPL/MIT 许可，
详见 [LICENSES.md](../../LICENSES.md)。

## 布局

```
blobs/
├── vendor/                                   → /vendor
│   ├── etc/
│   │   ├── mali_platform.config  @           → /vendor/etc/
│   │   └── permissions/
│   │       ├── android.hardware.vulkan.version.xml  @
│   │       └── android.software.vulkan.deqp.level.xml  @
│   ├── firmware/
│   │   └── mali_csffw.bin  @                 → /vendor/firmware/（reload 副本由同一份派生）
│   └── lib64/
│       ├── egl/{libGLES_mali.so, libGLES_meow.so, libMEOW_*.so}
│       ├── mt6895/{arm.mali.platform-V2-ndk.so, arm.graphics-V5-ndk.so,
│       │           libarm_mali_config_sysprops.so,
│       │           libarm_{egl,gralloc}_properties_sysprop.so}
│       ├── libc++.so  libged.so  libgpud.so  libmtk_mali_user.so
│       └── android.hardware.{graphics.common-V6,graphics.allocator-V2,common-V2}-ndk.so
└── vendor_dlkm/                              → /vendor_dlkm
    └── lib/modules/*.ko
```

`@` 标记的是**符号链接**，指向仓库里的单一事实源（`../firmware/`、`../files/`），
避免同一份固件/配置在仓库里存两份。`cp -a` / `rsync -a` 默认会跟随之，
若用 `tar` 或 `mkfs.erofs` 打包请确认加了 follow 选项（或直接拷贝解引用后的树）。
`files/vendor-build.prop.*-r54` 是**属性片段**，要追加进目标 ROM 的
`vendor/build.prop`，不是整份文件覆盖，因此不在这棵树里。

## vendor.img（19 个真实文件 + 4 个链接）

**替换的 r54 用户态库（10 个）** —— 底包里原有的同名文件，直接覆盖：

| 目标路径 | 大小 | md5 |
|---|---|---|
| `/vendor/lib64/egl/mt6895/libGLES_mali.so` | 51,726,104 | `4fa6d6747d39` |
| `/vendor/lib64/egl/libGLES_meow.so` | 2,165,008 | `3eb1c1def2a7` |
| `/vendor/lib64/egl/libMEOW_gift.so` | 859,392 | `76cfa9f0680f` |
| `/vendor/lib64/egl/libMEOW_mfrc_ext.so` | 731,304 | `e12515194375` |
| `/vendor/lib64/egl/libMEOW_trace.so` | 592,392 | `e39b1f8a4e3f` |
| `/vendor/lib64/egl/libMEOW_qt.so` | 201,872 | `24e55141344e` |
| `/vendor/lib64/egl/libMEOW_data.so` | 202,360 | `bfdb3dac2d74` |
| `/vendor/lib64/libged.so` | 101,944 | `f28930eddfa5` |
| `/vendor/lib64/libgpud.so` | 462,976 | `359112746293` |
| `/vendor/lib64/libmtk_mali_user.so` | 52,128 | `9758abf5e910` |

**底包缺失、必须新增的依赖（9 个）** —— 底包里没有这些文件，漏一个就是开机失败：

| 目标路径 | 大小 | md5 | 缺失后果 |
|---|---|---|---|
| `/vendor/lib64/libc++.so` | 1,154,016 | `da194e209887` | 底包 VNDK v31 的 libc++ 缺 r54 要的 257 个符号（`__libcpp_verbose_abort` 等）→ camerahalserver CANNOT LINK、开机循环。**不要改名旁挂**，见 FLASHABLE-SUPER 坑 4 |
| `/vendor/lib64/android.hardware.graphics.allocator-V2-ndk.so` | 44,416 | `ff5a37d861b0` | `libgpud.so` 的 `DT_NEEDED`；底包是 HIDL gralloc，无 AIDL 分配器客户端 → `dlopen` 失败 |
| `/vendor/lib64/android.hardware.common-V2-ndk.so` | 18,856 | `2647eaf1cb78` | 上面 `allocator-V2` 的依赖链（也可由 r54 自带的 `common-V6` 满足，二者选一） |
| `/vendor/lib64/android.hardware.graphics.common-V6-ndk.so` | 31,512 | `c1bca7a90685` | `libgpud.so` 直接 NEEDED |
| `/vendor/lib64/mt6895/arm.graphics-V5-ndk.so` | 35,176 | `a6f23b47ed3d` | r54 gralloc/EGL 侧 AIDL 依赖 |
| `/vendor/lib64/mt6895/arm.mali.platform-V2-ndk.so` | 85,344 | `eed8eea367cc` | Mali 平台 AIDL 服务 |
| `/vendor/lib64/mt6895/libarm_mali_config_sysprops.so` | 52,048 | `601e785792d0` | 读 `ro.vendor.arm.*` 属性的 shim |
| `/vendor/lib64/mt6895/libarm_egl_properties_sysprop.so` | 35,696 | `9ddbe1e5a34c` | 同上，EGL 侧 |
| `/vendor/lib64/mt6895/libarm_gralloc_properties_sysprop.so` | 35,672 | `41346913f1a9` | 同上，gralloc 侧；AFRC/AFBC usage 位配置链 |

**非库文件（4 个，仓库内为符号链接）**：

| 目标路径 | 来源 | 说明 |
|---|---|---|
| `/vendor/firmware/mali_csffw.bin` | `firmware/mali_csffw.bin` | UK 1.38；`mali_csffw_reload.bin` 由同一份派生，两者必须同版本 |
| `/vendor/etc/mali_platform.config` | `files/mali_platform.config` | sepolicy 已预期其存在 |
| `/vendor/etc/permissions/android.hardware.vulkan.version.xml` | `files/…` | version=4206592（1.3.0） |
| `/vendor/etc/permissions/android.software.vulkan.deqp.level.xml` | `files/…` | version=132711169（2025-03-01） |

> 可用 `readelf -d` 对照 r54 各库的 `NEEDED` 逐个确认目标 ROM 里是否已有，
> 已有则不必新增（但版本要兼容）。

## vendor_dlkm.img（7 个）

| 目标路径 | 大小 | md5 | 说明 |
|---|---|---|---|
| `/vendor_dlkm/lib/modules/mali_kbase_mt6895.ko` | 2,575,944 | `dbae07fbd38c` | r54p1，撤下电压档钩子版 |
| `/vendor_dlkm/lib/modules/ged.ko` | 427,704 | `4f430f5b52f1` | 含 `ged_force_fdvfs`（默认 1） |
| `/vendor_dlkm/lib/modules/gpueb.ko` | 183,752 | `f9a1cdf3e603` | 6.12 → 5.10 backport |
| `/vendor_dlkm/lib/modules/mtk_gpufreq_mt6895.ko` | 239,776 | `d205a61fd22e` | 带 `gpufreq_fix_target_oppidx` 导出 |
| `/vendor_dlkm/lib/modules/mtk_gpufreq_wrapper.ko` | 152,896 | `aaf684cacfae` | 同上，成对使用 |
| `/vendor_dlkm/lib/modules/mali_mgm.ko` | 28,704 | `45833415252c` | |
| `/vendor_dlkm/lib/modules/mali_prot_alloc.ko` | 24,240 | `3580df6a568f` | |

`modules.load` / `modules.dep` 需要在目标 ROM 上**就地追加**这 7 个条目
（不是文件覆盖），见 `docs/PORTING.zh-CN.md` 第 4 节。

> 换模块**必须做全量 `__crc_` 比对**，`lsmod` 显示已加载不代表能用，见
> [../README.md](../README.md) 的同名小节。

## 用法

```bash
# 解原版镜像并保留 xattr
sudo mount -o loop,ro vendor.img /mnt/v
sudo cp -a --preserve=all /mnt/v/. work_vendor/
sudo umount /mnt/v

# 整棵覆盖（含 firmware / etc / permissions；-L 解引用符号链接）
sudo cp -aL blobs/vendor/. work_vendor/
sudo chown -R 0:0 work_vendor
sudo chmod 644 work_vendor/lib64/egl/mt6895/libGLES_mali.so
```

`vendor_dlkm` 同理：`sudo cp -aL blobs/vendor_dlkm/. work_vendor_dlkm/`，
权限 `644`，标签 `u:object_r:vendor_file:s0`。

标签必须按 `docs/PORTING.zh-CN.md` 第 3.7 节用原版镜像逐文件对齐，
尤其是 `libGLES_mali.so` 和 `mali_csffw.bin` 必须是 `same_process_hal_file`。
新加的 `libc++.so` 与两个 `android.hardware.*-ndk.so` 走 **`vendor_file`**，
`libarm_*_properties_sysprop.so` 与 `libged.so` / `libgpud.so` 走 **`same_process_hal_file`**。

## 不在这棵树里的

| 文件 | 位置 | 为什么 |
|---|---|---|
| `gpueb` 固件（原厂 / 950MHz） | [../firmware/](../firmware/) | 落在 **gpueb 物理分区**，不是 vendor |
| `build.prop` 的 44 条 EGL 属性 | [../files/vendor-build.prop.egl-r54](../files/) | 追加进目标 ROM 的 `vendor/build.prop` |
| `build.prop` 的 12 条 gralloc 属性 | [../files/vendor-build.prop.gralloc-r54](../files/) | 同上 |
| `init.insmod.mt6895.cfg` | [../files/init.insmod.mt6895.cfg](../files/) | 修改目标 ROM 的 `/vendor/etc/` 版本 |
| `modules.load` / `modules.dep` 条目 | 无文件 | 就地追加，见 PORTING 第 4 节 |
