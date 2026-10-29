# RTX 4050 Laptop（AD107M / `10de:28a1`）算力解锁实录

在 Windows 上，用 UEFI 应用在 NVIDIA 驱动加载之前重放固件事务，解除 GeForce 对
FP16/BF16 → FP32 累加的 half-rate 钳制。

**这篇文档的重点不是"成功"，而是过程中那八个坑。** 每一个都是同一个模式：
把参考机器的具体值当成通用常量。BAR0 地址、Bus Master 的期望状态、ESP 的分区号、
事务入口的寄存器断言、驱动镜像的 hash——这类错误不会在代码审查里暴露，
只会在实机上暴露一次，而每次暴露都要重启、重新验证。

**结论**：事务成功，寄存器状态扛过了 Windows 启动，`FEAT_PLM` 保持开启、
`FEAT_READOUT_1` 归零（全速）。但绝对算力受本机 100 W TGP 限制，没有跑出参考
机器上的 2 倍——这不是软件问题。

**读者导航**

| 你想知道 | 看哪 |
| --- | --- |
| 为什么我这边跑不起来 / 卡在某个阶段 | [4. 踩过的坑](#4-踩过的坑) |
| 解锁后进 Windows 就失效了 | [5. 混合模式 vs 独显直连](#5-关键发现混合模式-vs-独显直连) |
| 到底有没有解锁成功、怎么验证 | [6. 结果](#6-结果) |
| 能不能复现 | [7. 复现要点](#7-复现要点) |
| 还有哪些没搞定 | [8. 仍未解决](#8-仍未解决) |

---

## 1. 目标硬件

| 项 | 值 |
| --- | --- |
| 机型 | 一台 2024 年款游戏本（OEM 贴牌，广达代工）—— 型号从略 |
| GPU | RTX 4050 Laptop，AD107M，`10de:28a1` |
| 显存 | 6 GB（`FB_SIZE_MB = 0x17fd` = 6141 MiB） |
| BAR0 | 16 MiB，UEFI 下分配于 `0x51000000` |
| 驱动 | NVIDIA 617.14 |
| TGP | **Current 100 W / Default 70 W / Max 105 W** |
| 电源 | 混合模式（Optimus）→ 后续切到独显直连 |

`Current Power Limit = 100 W` 这一行是后面所有"为什么没翻倍"的答案。

---

## 2. 路线选择

三条可选路径：

| 路径 | 是否可行 | 原因 |
| --- | --- | --- |
| Linux 内核模块（initramfs，驱动前加载） | 可行，社区主路线 | 不适用本机：需要装 Linux |
| Windows UEFI 应用（驱动前，本次采用） | **可行** | 复用同一套固件事务，不需要换系统 |
| Windows 自签内核驱动（运行时写） | 不可行 | 驱动加载时 NVIDIA 已在位；而且 PLM 门关着时主机写会被静默丢弃 |

采用 UEFI 路线。核心约束：**必须在 `bootmgfw.efi` 之前、NVIDIA 驱动之前**完成事务，
然后把控制权链式交还 Windows。

---

## 3. 事务做了什么

```
claim          读 PCI config，清 Bus Master（沿用的四个寄存器组见下）
layout         从 VBIOS 与 MMU 寄存器算 WPR2 / FB / heap / GSP 布局
radix          建 20606 个 DMA 页表项
FWSEC          从 PROM 解析 FWSEC 描述符，Falcon 点火建立 WPR2
booter         装载签名 Booter，经四写载荷翻转权限
selectors      写 SS0 = 0x88888888、SS1 = 0x8，读回校验
FLR            PCIe 功能级复位，清掉显存几何、保留 AON 里的选择器
handoff        LoadImage/StartImage 链回 bootmgfw.efi
```

关键寄存器（BAR0 偏移）：

| 名称 | 偏移 | 意义 |
| --- | --- | --- |
| `PMC_BOOT_0` | `0x00000000` | 芯片身份 |
| `GFW_PLM` / `GFW_BOOT` / `GFW_CPUCTL` | `0x00118128` / `0x00118234` / `0x00110100` | GFW 引导状态 |
| `WPR2_LO` / `WPR2_HI` | `0x001fa824` / `0x001fa828` | 上电到 FWSEC 前恒 0；非 0 = 本周期已被初始化 |
| `WPR_CFG` / `WPR` / `FBPA` | `0x001fa7cc` / `0x001fa7c4` / `0x009a0148` | 权限门 |
| `FEAT_PLM` | `0x00823804` | 默认 `0xffffff8f`；开到 `0xffffffff` 后主机可写选择器 |
| `FEAT_OVR_SS0` | `0x0082381c` | 八个 4 位字段，bit3 = 覆盖使能，目标 `0x88888888` |
| `FEAT_OVR_SS1` | `0x00823820` | 单字段，目标 `0x8` |
| `FEAT_READOUT_1` | `0x00823818` | 生效值读出，九个 3 位字段，全 0 = 全速 |

`FEAT_PLM` 属于 **AON（常开岛）**：FLR 打不回去，只有断电才会复位。这条性质既是
它能扛过 FLR 的原因，也是后面"为什么 Windows 起来后失效"的原因。

---

## 4. 踩过的坑

这一节是本文档的主要价值。每一个坑都来自"把参考机器的具体值当成通用常量"。

### 4.1 BAR0 被硬编码成参考板的 `0x80000000`

出现在三处：只读探针的 guard、设备发现、profile 校验。

本机 BAR0 是 `0x51000000`，于是探针直接拒绝采样，事务直接拒绝运行。

**修法**：运行时从 PCI config offset `0x10` 读，要求非 I/O、16 MiB 对齐、
且与 `GetBarAttributes` 描述符一致。

### 4.2 Bus Master 被要求"必须是关的"

同样出现在三处（探针 guard、`GuCheckConfig`、事务 claim）。

参考机上台式固件在 boot services 阶段不给 GPU 使能 BME，所以那里是关的。
**笔记本固件在 POST 期间有设备 DMA 需求，BME 一直是开的**（实测 command = `0x0007`）。

**修法**：
- 探针与发现阶段：不再要求关，只要求 Memory Space 开
- 事务 claim 阶段：**主动清掉**再继续（事务期间必须独占总线），
  并把原值存进 `original_cmd` 供收尾恢复

### 4.3 ESP 匹配条件写成参考机的分区号和 LBA

```c
d->PartitionNumber == 2 && d->PartitionStart == 1085440 && d->PartitionSize == 202752
```

本机 ESP 是分区 1、LBA 2048、614400 扇区。结果就是永远找不到 Windows 卷，
报 `EFI_NOT_FOUND`。

**修法**：只认 GPT + 分区 GUID。分区号和 LBA 本来就该逐机。

### 4.4 内核驱动的输入输出缓冲区重叠（导致三次蓝屏）

这是我自己写的驱动里的 bug，也是最值得记录的一个。

`METHOD_BUFFERED` 下输入输出**共用同一块内存**，而两个结构体布局重叠：

```c
IN : Phys(0..7)  Size(8..11)  Count(12..15)  Offsets[64](16..271)
OUT: Values[64](0..255)                      Status[64](256..511)
```

原来的循环：

```c
for (i = 0; i < in->Count; i++)
    out->Values[i] = READ_REGISTER_ULONG(base + in->Offsets[i]);
```

写 `out->Values[3]` 恰好覆盖 `in->Count`，写 `out->Values[4..]` 覆盖
`in->Offsets[]`。**循环到第四轮时，循环上限和偏移表都被自己的输出改写了**，
随后按垃圾偏移读任意地址 → `IRQL_NOT_LESS_OR_EQUAL (0xA)`。

三次蓝屏：

```
0x0000000A  地址 0xfffffffffffffff7
0x0000000A  同
0x00000139  KERNEL_SECURITY_CHECK_FAILURE
```

**修法**：进循环前把所有输入字段快照到局部变量，读满之后才写输出。

```c
count = in->Count;  phys = in->Phys;  span = in->Size;
for (i = 0; i < count; i++) offsets[i] = in->Offsets[i];
...
for (i = 0; i < count; i++) values[i] = READ_REGISTER_ULONG(base + offsets[i]);
...
for (i = 0; i < count; i++) { out->Values[i] = values[i]; out->Status[i] = 0; }
```

顺带一提，同一个驱动的 `IOCTL_READRAW` 是单次 `RtlCopyMemory`，参数已求值，
所以没有这个问题——**只有带循环的那个接口会自毁**。

### 4.5 冷值被硬编码成"混合模式下抓到的 19 项"

一开始按 fail-closed 设计：19 项寄存器必须逐项等于冷态快照。混合模式下抓到的
快照在这台机器上是有效的。

但**切到独显直连之后失效了**——固件开始真正 POST 这块卡（它成了 panel owner），
`GFW_CPUCTL` 从 `0x10` 变 `0x20`、`MMU_LO/HI` 被重配、`SS0` 从 `0x61267164`
变成 `0`。所有后续运行都停在 `EFI_NOT_READY`。

**结论**：独显直连下不存在"完全冷态"，fail-closed 模式会永远拒绝。
改用 keyed 准入：只比对驱动不会改写的那几项（die id、`FEAT_PLM`、
fuse 位、`SS0/SS1` 的锁存前提）。

### 4.6 事务的入口断言硬编码了 `ss1==1` 和 `GFW_CPUCTL==0x10`

```
entry: plm=0xffffff8f ss0=0x61263164 ss1=0x1 wpr2_hi=0     ← 混合模式
entry: plm=0xffffff8f ss0=0                ss1=0 wpr2_hi=0  ← 独显直连
```

独显直连下 `ss1=0`、`GFW_CPUCTL=0x20`，两条断言都失败，返回 `-EIO`。

**修法**：两种拓扑都接受，并在拒绝路径上打印每个寄存器的实际值
（否则只能看到一行 `-EIO`，完全不知道为什么）。

### 4.7 参考实现的驱动识别 hash

上游 `msi_release.c` 有 524 行，核心是按 **SHA-256** 认出参考机器固件里那三个特定
GPU 驱动镜像，`DisconnectController` 之后独占 PCI。

这些 hash 在本机必然不匹配，于是 `Consumers()` 直接失败，事务根本进不去。

**修法**：换成通用 detach——不认任何 hash，只看 GPU handle 上有没有
`GraphicsOutput` 实例，有就 `DisconnectController(handle, NULL, NULL)` 断开它所有驱动。
这样两种拓扑都能工作：混合模式没有 GOP，什么都不动；独显直连有 GOP，断开它。

### 4.8 PROM 视图的切片长度错了

最初把 nvflash 容器从 offset `0x9400` 切 1 MiB 当作 PROM 视图，算出的 sha256
是 `e44ef1ed…`。

实际从 BAR0 + `0x300000` 读出来的 1 MiB 是 `1dd928ea…`——差 162468 字节，
从 `0xd6c00` 开始，那里是 nvflash 容器里**第二个 NVGI 段**，不属于 PROM 视图。

正确构造（已实测验证）：

```python
prom = container[0x9400 : 0x9400 + 0xa1600] + b'\xFF' * (1048576 - 0xa1600)
```

---

## 5. 关键发现：混合模式 vs 独显直连

这是整件事的分水岭。

第一次事务是**成功**的（`RUN0004`，`TRANSACTION OK`），但进 Windows 后算力没变。
直接读寄存器看到的却是：

```
FEAT_PLM      = 0xffffff8f    ← 退回锁定基线
SS0 / SS1     = 0x0 / 0x0
FEAT_READOUT_1= 0x410080      ← 节流值
```

`FEAT_PLM` 是 AON 成员，**连 FLR 都打不回去**。能把它复位的只有断电。而混合模式下
Windows 会把非 panel-owner 的 dGPU 打进 D3cold——那一次断电，把 UEFI 里做的一切
都抹掉了。

把 GPU 拉到 P0（2550 MHz / 56.94 W）后重读，`SS0` 依然是 0，所以这不是"暂时休眠"
能解释的，就是一次完整的复位。

**解法：切独显直连（MUX）。** 让 dGPU 成为 panel owner，Windows 就不会给它断电。

切法因 OEM 而异。这台机器是 OEM 自带的控制中心（`ControlCenter.exe`），
资源里能挖出三个模式的图标：

```
res3/System Mode/ic_discrete_mode_choose.png
res3/System Mode/ic_ms_hybrid_mode_choose.png
res3/System Mode/ic_uma_mode_normal.png
```

切到「独显」并重启后，判据不是"Intel 消失"（Advanced Optimus 下 Intel 设备仍在），
而是**谁在驱动屏幕**：

```
NVIDIA GeForce RTX 4050 Laptop GPU   Mode: 1920 x 1080   ← panel owner
Intel(R) UHD Graphics                Mode: （空）
```

切过去之后，事务一次通过。

---

## 6. 结果

### 6.1 事务日志（`RUN0012.TXT`，节选）

```
entry: plm=0xffffff8f ss0=0x61263164 ss1=0x1 wpr2_hi=0
claim: cleared firmware Bus Master 0x7 -> 0x3
FWSEC result: scratch=0 expected_hi16=0
FWSEC WPR2: low=0x17fb000 high=0x17fbe00 expected_page=0x17fb00
AD102 payload result: marker=0x67b00001 marker_ok=1 required_permissions=1
selector write: SS0=0x88888888   → selector result: matched=1
selector write: SS1=0x8          → selector result: matched=1
AD102 selectors-applied: readout1=0 full=1
PCIe FLR returned: result=0
post-FLR SEC2 reset released, GSP halted, BME=0 pending=0
    WPR2=1ffffe00:00000000 SS0=88888888 SS1=00000008
AD102 permission post-FLR: FEAT[0x823804]=0xffffffff     ← 门在 FLR 后仍开着
Transaction result=0 safe=1 started=1 quiesced=1 flrs=1 DMA_live=0
TRANSACTION OK
```

### 6.2 Windows 里的实际寄存器

事务后、Windows 完全启动、基准跑完之后读：

```
PMC_BOOT_0       0x197000a1        FEAT_PLM         0xffffffff    ← 开着
GFW_PLM          0x00008b8f        SS0              0x88888888
WPR2_HI          0x00000000        SS1              0x00000008
WPR_CFG          0x0004cb8f        FEAT_READOUT_1   0x00000000    ← 全速
WPR              0x0004cb8f        FB_SIZE_MB       0x000017fd
```

`FEAT_PLM` 保持 `0xffffffff`、`FEAT_READOUT_1` 归零——这是前几次都没做到的。

### 6.3 算力

cuBLAS `GemmEx`，8192³，FP16 输入 / FP32 累加，20 次取中位：

```
FP16 -> FP32 accum :  30.44 TFLOPS
FP32 (SGEMM) 对照  :   6.45 TFLOPS
负载期间           :  2535 MHz / 91–93 W / 利用率 99–100%
```

对照同为 AD107 的 RTX 4060 Laptop（24 SM / 140 W TGP）公开数据：

| | 算力 | 功耗 | 每瓦 |
| --- | --- | --- | --- |
| 4060 未解锁 | 26.37 TFLOPS | 88 W | 0.300 |
| 4060 解锁后 | 50.29 TFLOPS | 139 W | 0.362 |
| **本机** | **30.44 TFLOPS** | **93 W** | **0.327** |

按 SM 数（20/24）和功耗归一化：

- 若未解锁，93 W 下应约 **23 TFLOPS**
- 若已解锁，93 W 下应约 **28 TFLOPS**
- 实测 **30.44**，高于解锁预期

**所以解锁在算力上是生效的，只是被 100 W 的 TGP 卡住了。** 参考实现里 RTX 3080
那份报告也给出同样结论（"收益受功耗墙限制"）。

---

## 7. 复现要点

1. **Secure Boot 关闭**（或把自签证书入 db）。未签名 UEFI 应用才能加载。
2. **关掉 Windows 快速启动**。开着时"关机"是 S4 休眠，下次开机走 resume 路径。
3. **切独显直连**（如果机器有 MUX）。这是能不能扛过 Windows 启动的关键。
4. U 盘 FAT32，`\EFI\BOOT\BOOTX64.EFI` + 五个固件资产平铺在盘根。
5. 从**重启**或**冷启动**进 U 盘，不要在休眠唤醒后进。
6. 事务后屏幕会黑（断开显示驱动），给它最多 4 分钟，Windows 会自己起来。

固件资产需从钉 SHA-256 的官方输入本地再生，不随仓库分发。

---

## 8. 仍未解决

- **绝对算力受 TGP 限制**。100 W 的卡跑不出 140 W 卡的数字，这是物理限制不是软件问题。
- **每引导重放**。解锁是易失覆盖，断电即失，每次开机都要重跑一次。
- **功耗墙之下还能榨多少**。当前实测 30.44 TFLOPS @ 93 W 已接近归一化预期上限，
  但没见过本机"未解锁基线"的 cuBLAS 实测（需要冷启动放电后采样），
  所以 31% 这个提升幅度是按同型号数据推算的，不是直接测出来的。
- 参考项目把"SS 值正确但性能仍是 stock"列为待查的第四种机理（时钟钳/功耗墙之外
  是否还有别的消费者），本机数据指向功耗墙，但没有排除其他可能。

---

## 附：工具

| 工具 | 用途 |
| --- | --- |
| UEFI 探针（`BOOTX64.EFI`） | 只读采样 22 项寄存器 + 1 MiB PROM，写 JSON 到 U 盘 |
| 自签只读内核驱动 + `IOCTL_GU_PROBE_READ32` | Windows 里读同一批寄存器（批量、分 1 MiB 块） |
| cuBLAS GEMM 基准 | 判定算力档位，必须用 cuBLAS/PyTorch，手写微基准容易被占用率骗 |
| `nvidia-smi -q -d POWER` | 先看功耗上限，再谈性能 |

---

本文记录的是本机实测。不同型号/BIOS 的固件行为可能不同，
尤其"混合模式会不会抹掉解锁"这一点，取决于固件在什么时机给 dGPU 断电。
