# btop Android arm64 支持与静态编译

本文档记录当前仓库为 Android/recovery 增加的 btop 支持、各项数据的实际来源、静态编译方法和已知限制。

## 当前产物

编译产物固定为：

```text
bin/btop
```

不要把测试版本或改名副本散落到其他目录。当前产物应满足：

- ARM64/AArch64 ELF；
- Android Bionic 目标；
- 静态链接；
- 已 strip；
- 不包含 `INTERP` 和 `DYNAMIC` program header。

## Android 支持概览

当前代码增加了以下 Android/recovery 支持：

1. Android/Bionic 静态编译兼容；
2. Qualcomm KGSL/Adreno GPU 占用、频率和温度；
3. Android CPU 总温度筛选和平均策略；
4. 每个 CPU 核心的实时频率；
5. Android vendor `current_now` 单位兼容和电池侧功率显示；
6. 针对 CPU/GPU频率信息调整终端布局。

主要改动文件：

```text
Makefile
src/btop.cpp
src/btop_config.cpp
src/btop_draw.cpp
src/btop_shared.hpp
src/linux/btop_collect.cpp
```

## Android/Bionic 编译兼容

### 线程退出

Android Bionic 环境不提供当前代码使用的 `pthread_timedjoin_np`。Android 构建改用普通的：

```cpp
pthread_join(...)
```

对应改动位于 `src/btop.cpp`。

### 运行期函数不能声明为 constexpr

以下函数依赖环境变量、文件系统或运行期文件内容，因此移除了 `constexpr`：

```text
Config::get_xdg_state_dir()
Cpu::detect_active_cpus()
```

对应文件：

```text
src/btop_config.cpp
src/linux/btop_collect.cpp
```

### Android链接裁剪

Makefile 检测到编译器 target triple 包含 `android` 时增加：

```text
-ffunction-sections
-fdata-sections
-Wl,--gc-sections
```

这会把函数和数据放进独立 section，并在静态链接时移除未使用内容，减小最终二进制体积。

## Qualcomm Adreno GPU支持

Android 构建启用 `GPU_SUPPORT=true` 后使用 KGSL sysfs，不加载桌面 Linux 的 NVML、ROCm或 Intel GPU采集器。

设备根目录：

```text
/sys/class/kgsl/kgsl-3d0
```

### GPU占用

数据来源：

```text
/sys/class/kgsl/kgsl-3d0/gpubusy
```

该文件提供两个累计值：

```text
busy total
```

btop 使用下面的公式计算占用率，并把结果限制在 0～100：

```text
GPU使用率 = round(busy × 100 / total)
```

### GPU频率

优先读取：

```text
/sys/class/kgsl/kgsl-3d0/devfreq/cur_freq
```

如果不存在，则回退到：

```text
/sys/class/kgsl/kgsl-3d0/gpuclk
```

节点值按 Hz 读取，除以 `1,000,000` 后显示为 MHz。

### GPU温度

数据来源：

```text
/sys/class/kgsl/kgsl-3d0/temp
```

如果读数大于 1000，则按毫摄氏度处理并除以 1000；否则直接按摄氏度使用。GPU温度图的默认上限沿用 btop 的 110°C。

### GPU显示和限制

配置中的 `shown_gpus` 增加了 `adreno`，默认值为：

```text
nvidia amd intel apple adreno
```

当前 KGSL 后端支持：

- GPU使用率；
- GPU核心频率；
- GPU温度。

当前不支持：

- 独立显存使用量和总量；
- GPU内存频率；
- GPU功耗；
- 编码器/解码器占用；
- PCIe吞吐。

这些项目没有可靠、通用的 Android KGSL sysfs 数据源，因此不会显示伪造或估算值。

## CPU频率

### CPU总频率

Android 的 cpufreq policy 编号可能不连续。代码不再假定存在 `policy0` 到 `policyN`，而是扫描：

```text
/sys/devices/system/cpu/cpufreq/policy*/scaling_cur_freq
```

只接受 `policy` 后面为数字的目录，并按 policy 编号排序。

### 每核心频率

每个核心读取：

```text
/sys/devices/system/cpu/cpuN/cpufreq/scaling_cur_freq
```

节点值按 kHz 读取，除以 1000 后保存为 MHz。启用 btop 的 `show_cpu_freq` 时，每个核心的使用率后面会显示类似：

```text
C0  23%  750M
C1  41% 1804M
```

如果某个核心没有可读的 cpufreq 节点，则显示 `-`。UI布局会为每核心频率额外预留 6 列，避免覆盖温度或其他核心信息。

## CPU温度策略

Android 设备通常存在大量 `thermal_zone`。当前代码仍扫描：

```text
/sys/class/thermal/thermal_zoneN/type
/sys/class/thermal/thermal_zoneN/temp
```

但 Android 的 CPU总温度只选择 `type` 名称满足以下前缀的 zone：

```text
cpu-*
cpuss-*
```

过滤规则：

- 排除 `0`；
- 排除负数；
- 排除大于 `150000` 的明显异常值，即超过 150°C；
- 无法解析的读数直接跳过。

最终 CPU总温度为所有有效 `cpu-*` 和 `cpuss-*` zone 的算术平均值：

```text
CPU总温度 = round(sum(有效zone原始温度) / 有效zone数量 / 1000)
```

显示单位为摄氏度，CPU温度图上限固定为 95°C。

该策略只生成 CPU总温度，不会把 thermal zone 强行映射为每个 CPU核心的独立温度。

## 电池功率

电池数据来自：

```text
/sys/class/power_supply/*
```

btop 选择 `type` 为 `Battery` 或 `UPS` 且 `present=1` 的电源设备。

### 功率来源优先级

第一优先级：

```text
power_now
```

换算公式：

```text
功率(W) = abs(power_now) / 1,000,000
```

如果 `power_now` 不存在、无效或不大于0，则使用：

```text
current_now
voltage_now
```

计算：

```text
功率(W) = abs(current_now换算为A) × voltage_now / 1,000,000
```

### Android电流单位兼容

Linux power_supply ABI通常规定 `current_now` 使用 µA，但部分 Android vendor驱动实际返回 mA。

当前启发式规则：

```text
0 < abs(current_now) < 100000  -> 按 mA处理
其他非零值                    -> 按 µA处理
```

这解决了某些 Oplus/Android设备在快充时返回类似 `-10124`，实际表示 `-10.124A` 的情况。

### 当前显示值的含义

当前 btop 标题栏显示的是电池侧功率绝对值，不是 USB输入功率，也不是严格意义上的整机耗电：

```text
P_battery = abs(V_battery × I_battery)
```

充电/放电箭头仍来自电池的 `status` 字段。计算功率时使用了绝对值，因此功率数值本身不携带方向。

该启发式也存在边界：真实的小电流 µA读数如果低于100000，可能被误判为 mA。它是针对当前 Android vendor节点的兼容方案，不是通用 ABI修复。

## 尚未集成的充电信息

以下内容经过设备侧调查，但目前没有写入 btop 正式代码：

- USB/扩展坞输入功率；
- PD、PD_SDP、PD_PPS、SVOOC等充电协议；
- 充电器输入电流；
- 充电时的纯设备功耗；
- SVOOC双 charge-pump 的 `main_cp_ibus`、`slave_cp_ibus` 和 `cp_vbus`；
- Oplus Charger AIDL HAL。

原因是这些数据在不同充电模式下没有统一可靠节点。例如 SVOOC 下：

```text
/sys/class/power_supply/usb/current_now
```

可能返回0，而实际输入电流只出现在厂商驱动日志或私有接口中。不能把电池侧功率当作输入功率，也不能用 `输入功率 - 电池充入功率` 精确表示整机功耗，因为其中还包含 charge-pump、PMIC和线路损耗。

## 编译环境

当前使用：

```text
ANDROID_NDK=/Users/Laurie/Library/Android/sdk/ndk/27.3.13750724
ABI=arm64-v8a / AArch64
Android API=35
Host=macOS arm64
```

NDK在 Apple Silicon Mac 上仍使用名为 `darwin-x86_64` 的预编译工具链目录：

```text
$ANDROID_NDK/toolchains/llvm/prebuilt/darwin-x86_64/bin
```

## 静态编译

进入仓库：

```sh
cd /Users/Laurie/Desktop/nightmare-space/AuroraRecoveryProject/btop
```

确认 NDK：

```sh
export ANDROID_NDK=/Users/Laurie/Library/Android/sdk/ndk/27.3.13750724
```

清理旧对象：

```sh
make clean
```

编译 Android arm64 静态版本：

```sh
make \
  PLATFORM=linux \
  ARCH=aarch64 \
  STATIC=true \
  GPU_SUPPORT=true \
  STRIP=true \
  DEBUG= \
  CXX="$ANDROID_NDK/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android35-clang++"
```

参数说明：

- `PLATFORM=linux`：复用 btop 的 Linux `/proc`、sysfs和 power_supply采集器；
- `ARCH=aarch64`：目标架构为 ARM64；
- `STATIC=true`：传递 `-static`，生成静态可执行文件；
- `GPU_SUPPORT=true`：编译 KGSL/Adreno支持；
- `STRIP=true`：去除符号，缩小产物；
- `DEBUG=`：必须显式置空，避免宿主环境中的 `DEBUG=release` 被 Makefile 当作“已启用Debug”，从而使用 `-O0 -g`；
- `CXX=...android35-clang++`：使用 NDK API 35 的 AArch64 C++编译器。

编译完成后只有正式产物需要保留：

```text
/Users/Laurie/Desktop/nightmare-space/AuroraRecoveryProject/btop/bin/btop
```

## 验证静态链接

先检查文件类型：

```sh
file bin/btop
```

预期包含：

```text
ELF 64-bit LSB executable, ARM aarch64
statically linked
stripped
```

再检查 program header：

```sh
READELF="$ANDROID_NDK/toolchains/llvm/prebuilt/darwin-x86_64/bin/llvm-readelf"
"$READELF" -h -l bin/btop
```

应满足：

- `Type: EXEC`；
- `Machine: AArch64`；
- program header中没有 `INTERP`；
- program header中没有 `DYNAMIC`。

也可以快速检查：

```sh
"$READELF" -l bin/btop | grep -E 'INTERP|DYNAMIC'
```

该命令没有输出才符合当前静态构建要求。

## 在 recovery 中运行

以下是可选的手动部署示例，不属于编译过程：

```sh
adb push bin/btop /tmp/btop
adb shell 'chmod 0755 /tmp/btop'
adb shell 'TERM=xterm-256color /tmp/btop'
```

要显示 Android扩展信息，设备内核需要暴露对应节点，并允许当前进程读取：

```text
/proc/stat
/sys/devices/system/cpu
/sys/class/thermal
/sys/class/power_supply
/sys/class/kgsl/kgsl-3d0
```

建议在 recovery/root环境运行，否则部分 KGSL、thermal或 power_supply节点可能因权限不可读而不显示。

## 常用配置

在 btop Options 中确认：

```text
show_cpu_freq = true
check_temp = true
show_battery = true
show_battery_watts = true
show_gpu_info = Auto 或 On
shown_gpus 包含 adreno
```

如果 `GPU_SUPPORT=false`，即使设备存在 KGSL节点，也不会编译或显示 Adreno GPU信息。
