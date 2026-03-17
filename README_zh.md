# DRAM-less NVMe SSD 在 适配较差且复杂综合的电源管理下无法正常安装与使用 Linux 与 Windows 双系统环境的非常规问题分析与解决
------
> 我的基础环境
硬件：机械革命耀世 15Pro，Intel i9-14900HX（14 代移动端 HX 系列），RTX4060 Laptop GPU；双 M.2 NVMe 插槽，**槽 1 为 CPU 直连 PCIe 4.0 x4 通道，槽 2 为 PCH 引出 PCIe 通道**；
存储：原厂 Windows 系统盘（槽 1，下称长江盘），新增西数 Blue SN580 1TB NVMe SSD（DRAM-less 设计，**闪迪自研主控**，**拥有自动的休眠管理**），专门用于安装 Linux；
系统：原Windows11系统；BIOS只能关secure boot和VMD但没有设置Intel RST的选项；计划加装Ubuntu22.04.5（内核6.8+）

------

# 问题介绍

风和日丽，我也只是跟着网上的各种教程在给我的笔记本装双系统，但是遇到了一类在 DRAM-less NVMe SSD 上出现的复杂问题： 
- 尝试一：采用`m.2硬盘+硬盘盒（主控为RTL9210B）`装 Linux系统，遇到装好系统后往硬中写入大量碎片小文件（4K写入）时候出现掉盘系统奔溃的情况；
- 尝试二：直接往笔记本的第二个m.2插槽加装硬盘打算直接装 Linux 双系统环境时，出现在使用Ventoy启动盘的 live Ubuntu 进行安装时， NVMe 设备无法被识别甚至掉盘、容量识别为 0B、无法识别分区以及重启后硬盘又消失等异常现象。
- 其间在 Windows 系统中可以正常识别与使用该西数硬盘，且硬盘盒当普通数据存储盘也无任何掉盘问题，基本排除了硬件本身故障的问题。
- 且哪怕在系统刚装好的时候在Windows中重启（warm start），又出现了BIOS直接未识别到西数硬盘导致跳过grub启动，直接进入Windows又没法看到西数硬盘。

通过在外置 USB（RTL9210B）、内置 M.2 插槽以及双系统环境中的多阶段排查与实验，本文发现该问题并非单一原因导致，而是 NVMe 电源管理机制、平台固件（BIOS/UEFI）以及操作系统初始化过程之间的交互问题。

通过禁用 NVMe 深度省电（APST）以及调整启动链（GRUB/UEFI），问题得到稳定解决。本文提供完整复现路径、失败尝试以及最终解决方案，为类似环境提供参考。

> 我使用的教程：
>
> [杰哥b站双系统教程](【Windows11 安装 Ubuntu 避坑指南】https://www.bilibili.com/video/BV1Cc41127B9?vd_source=4fee1919587da67df22a63b9ee031587)
>
> [Ubuntu首次使用的简单优化](【Ubuntu 24.04 LTS 安装后必须做的20件事】https://www.bilibili.com/video/BV1A5SFYMEtD?vd_source=4fee1919587da67df22a63b9ee031587)
>
> [移动硬盘装Ubuntu to go（借鉴其中的分区教程）](【【2024】Linux to go 移动硬盘安装Ubuntu22.04（全程）】https://www.bilibili.com/video/BV14BxRedEnG?vd_source=4fee1919587da67df22a63b9ee031587)
>
> [在移动硬盘装Ubuntu](https://zhuanlan.zhihu.com/p/424967021)
>
> [装Ubuntu系统遇到的问题](https://zhuanlan.zhihu.com/p/374663335)

------

# 问题核心拆解
### 根源一：移动硬盘盒主控`APST`机制不适配导致的硬盘”睡死“问题（针对m.2+移动硬盘盒）
APST（Autonomous Power State Transition），自主电源状态转换，在RTL9210B这颗主控上对Linux内核的适配度差，且该方案发热量也较大，通过USB接口连接外接硬盘对供电要求也相对苛刻。尝试过的方案：更换 USB 口、怀疑温度问题、怀疑供电问题、更换系统，均未彻底解决。长期试验发现，在外接盘的Ubuntu，进系统使用大概`10-20分钟`就立刻出现掉盘现象，系统卡死（其实系统已经挂了），运气不好还可能损坏硬盘上Ubuntu的引导文件。主要问题在硬盘盒主控会自动选择降低功耗进入休眠，短期供电下降进入休眠，而Linux对该主控的休眠机制适配较差，硬盘休眠后无法被系统唤醒直接睡死导致掉盘；以及短时间内大量小文件写入，硬盘盒迅速升温，RTL9210B主控重启，也可能导致Linux一识别不到主控重启就认为掉盘而发生真的掉盘。
参考[完美解决：Linux (Kali/Ubuntu) 外接 Type-C 硬盘盒高负载“掉盘死机”问题](https://hackmd.io/@BigDick/SycHO3NFZl)
> 但是实际上并不能解决，我自己测试只是从10分钟就掉盘延长到了30分钟才掉盘，依然外接硬盘盒和Linux内核的水土不服
> 当然，社区里还有人尝试给外接硬盘加供电以及更换对Linux内核有维护的`祥硕ASM`主控，我对此希望不大，且以为直接装双系统可以避开此类问题，就开装了。。

### 根源二：SN580 主控 APST 电源管理（DRAM-less SSD特性）与 Linux 通用驱动的兼容性 bug（最核心）
#### DRAM-less SSD 的核心特点：
- 不带 DRAM
- 使用 HMB（Host Memory Buffer）
- 依赖主机内存
- 更激进的电源管理
```text
✔ 更容易进入低功耗状态（APST）
✔ 控制器状态依赖主机
✔ 唤醒链更复杂

进入深度省电状态后
↓
控制器响应延迟或失败
↓
操作系统初始化失败
```
NVMe 规范定义了APST（Autonomous Power State Transition，自主电源状态转换） 功能：SSD 主控可根据负载自主在多个电源状态（PS0~PS4）间切换，其中 PS0 为满速工作状态，PS4 为深度休眠状态（功耗最低，仅保留基础供电）。
消费级 DRAM-less SSD（如 SN580）为了优化功耗和发热，会将深度休眠的触发阈值设置得极低（空闲数百毫秒即进入 PS4），且其闪迪自研主控对 PS4 状态做了定制化设计：进入深度休眠后会关闭 PCIe 物理层的部分电路，仅支持 Windows 官方 WD 驱动的专属唤醒流程，不响应 Linux 通用 NVMe 驱动的标准规范唤醒指令。
- 于是乎Windows/BIOS 能正常识别（Windows 有 WD 专属驱动，可正常唤醒休眠的 SSD；BIOS 不会让 SSD 进入低功耗状态，全程保持 PS0 满速，因此可正常枚举）；
- Linux 下 0B 识别异常：SSD 进入 PS4 深度休眠后 “睡死”，Linux 内核仅能枚举到 PCIe 总线上的 NVMe 控制器，无法与 SSD 主控建立正常通信，无法正常进行`namespace initialization`无法读取 LBA 地址空间、分区表，因此显示为 0B，所有读写操作均报错（`I/O ERROR`）；
- 移动硬盘盒掉盘同源：绿联硬盘盒的 USB-NVMe 桥接芯片自带电源管理，会叠加触发 SSD 的 APST 休眠，Linux 的 USB 存储驱动更难唤醒已经睡死的 SSD，因此出现随机掉盘。
> 这里我也尝试在社区寻找解决办法，但是基本上都是QA板块中只问不答，有与我遇到相似的问题，都是西数这类DRAM-less SSD硬盘发生识别不到的问题，但都没有大佬回答。

### 根源三：PCIe 枚举问题（平台相关）
> 主要出现在双系统刚装好时，依然出现Linux系统时可进时不可进

在Windows系统的 warm reboot（重启）中（或者是拆机装机时候主板静电没有完全释放）：
```
PCIe 总线未完全 reset
↓
NVMe controller 未响应
↓
BIOS 枚举失败
```
表现为：BIOS 中 NVMe 消失。
同时还有UEFI BootNext 机制（Windows行为）使得Windows 在 restart 时可能：
```
设置 BootNext → Windows Boot Manager导致：
绕过 BootOrder
↓
绕过 GRUB
```
最后导致：
```text
进入深度省电状态后
↓
控制器响应延迟或失败
↓
操作系统初始化失败
```
> 问题复杂，只能认为对我们遇到的情况有影响，但不能算主要原因

### 根源四：（特别针对系统装好后在Windows界面重启又掉盘的问题）Warm reboot 未完全 reset NVMe controller 并且 GRUB 被绕过
```text
Windows 设置 BootNext
绕过 GRUB
```
**叠加根源三，猜测大概和绕过GRUB后，GRUB里面我们已经设置的强制高性能运行的电源管理策略没有生效**
> 触发问题的本质还是没有完全发掘出来，但也没有必要，这个还算解决思路明确，强制BIOS的UEFI从Ubuntu启动同时在Windows下只是用关机而不使用重启就能必经GRUB
![](flowchart.png)

# Failed Attempts（无效尝试）
1. 使用RTL9210B硬盘盒运行 Linux 系统时候：更换 USB 口、怀疑温度问题、怀疑供电问题、更换系统、尝试[社区的这篇文章](https://hackmd.io/@BigDick/SycHO3NFZl)
**均未彻底解决**
2. 分区/格式化：
在 Windows 中对硬盘删除分区重建 并在 Linux 强制格式化（gparted）
**无效（本质不是分区问题）**
3. 更换插槽：M.2_1 / M.2_2 互换，猜测是“槽 1 为 CPU 直连 PCIe 4.0 x4 通道，槽 2 为 PCH 引出 PCIe 通道”之间的性能差异
**部分改善（枚举概率变化）但未根治**
4. BIOS 设置：关闭 Secure Boot、关闭 VMD（因为这是装双系统必须的操作，可见文章开头装双系统的教程推荐），但由于我的笔记本BIOS没法直接设置AHCI和Intel RST，理由不够充分
**必要但不足**
5. 反复重装系统，反复插拔外接硬盘，反复重插m.2插槽，甚至直接把长江盘拔下只装西数盘来保持纯净环境装系统，也依然出现上面的问题
6. **只是无能狂怒罢了**

# 目前经过尝试得到的关键解决方案
 ### **在 GRUB 中加入**：
```bash
nvme_core.default_ps_max_latency_us=0
nvme_core.multipath=0
pcie_aspm=off
```
进入Ubuntu系统，执行以下命令，将根治SSD适配问题的内核参数永久固化到GRUB配置中：
```bash
# 编辑GRUB配置文件
sudo nano /etc/default/grub
# 将内核参数添加到配置行中
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvme_core.default_ps_max_latency_us=0 nvme_core.multipath=0 pcie_aspm=off"
# 保存退出后更新GRUB配置
sudo update-grub
sudo reboot
```
**作用解释**
1. nvme_core.default_ps_max_latency_us=0（关键参数）：
```text
禁用 NVMe 深度省电（APST）
限制 NVMe 进入深度低功耗状态
↓
避免 controller “睡死”
↓
保证初始化成功

✔ 直接解决 0B / 无分区 / 无法安装问题
✔ 系统稳定性显著提升
本问题的主要解决因素
```
2. nvme_core.multipath=0：
```
作用：关闭 NVMe 多路径（Multipath I/O）
多路径用于企业级存储（多个控制路径）,一般消费级 SSD 通常不需要
减少初始化复杂度
↓
避免潜在兼容性问题
↓
辅助稳定性参数（建议保留）
```
3. pcie_aspm=off
```
作用：关闭 PCIe ASPM（Active State Power Management）、禁用 PCIe 链路层省电（L0s / L1）
ASPM 由 BIOS + OS 共同控制，某些平台存在兼容性问题
✔ 对掉盘问题有一定缓解
❌ 但不是决定性因素
⚠️ 增加功耗
⚠️ 笔记本续航下降
非必须参数，仅在问题严重时使用
```
### GRUB统一引导的双系统跨重启NVMe掉盘根治方案
**作用**：解决了Windows热重启后Linux系统NVMe SSD掉盘、识别异常的核心痛点，实现了双系统的无冲突统一引导，同时彻底根治了跨系统重启触发的SSD假死问题，直接爽快设置不深究原因。
```
1.  强制所有开机、冷启动、Windows热重启流程，必先经过Ubuntu系统盘的GRUB引导器，保证NVMe SSD在任何启动场景下都能被强制重置到正常工作状态；
2.  实现双系统的可视化统一引导，可在开机后进GRUB自由选择进入Windows或Ubuntu系统，无需反复修改BIOS启动顺序；
3.  彻底解决Windows热重启后Linux系统NVMe SSD掉盘、0B容量识别异常的问题，无需禁用Windows重启功能，保证主用Windows的原生使用体验；
4.  适配双硬盘物理隔离ESP分区的场景，引导操作完全可逆，无数据丢失、系统崩溃风险。
```
> 本方案适配双硬盘物理隔离场景：M.2插槽1为Ubuntu系统盘（WD Blue SN580），自带独立ESP引导分区；M.2插槽2为Windows原厂系统盘，自带独立ESP引导分区。所有操作均为系统原生支持，无第三方工具依赖。
> 选择把西数盘插在M.2插槽的第一插槽也是尽量避免还有意外不稳定的因素

#### 步骤1：BIOS启动优先级配置
1.  开机按主板对应热键（本场景为F2）进入UEFI BIOS设置界面；
2.  进入「Boot（启动）」选项卡，将**Ubuntu系统盘对应的UEFI启动项**设置为第一启动优先级，Windows Boot Manager设置为第二优先级；
3.  保存BIOS配置并退出，保证主板开机默认优先读取Ubuntu系统盘的GRUB引导器。

#### 步骤2：在Ubuntu中设置GRUB的默认启动选项和UEFI的启动顺序
1. **配置 GRUB 记忆上次启动项**
```bash
sudo nano /etc/default/grub
```
修改为：
```bash
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
```
更新：
```bash
sudo update-grub
```

---
**效果**：GRUB 将记住用户上一次选择的系统
例如：
```text
上次进 Windows → 下次默认 Windows
```

---

2. **修改 UEFI 启动顺序**
使用：
```bash
sudo efibootmgr
```
查看当前顺序：
```text
BootOrder: 0004,0003
```
修改为：（如果显示current ubuntu的编号是0003）
```bash
sudo efibootmgr -o 0003,0004
```
UEFI 就改成了优先尝试Ubuntu，就能保证进GRUB

---

**效果**
```text
所有 Windows 重启行为：
↓
进入 GRUB
↓
用户选择系统
```

---

**系统启动链**变为：
```text
Power On / Restart
↓
UEFI
↓
GRUB
↓
User Selection (Ubuntu / Windows)
```

---

**注意事项**
```text
1. GRUB_DEFAULT=saved 提供“用户级控制”
2. efibootmgr 提供“固件级优先级”
3. bcdedit 提供“Windows 层拦截”
```

#### 应急处理方案
若修改引导后出现GRUB救援模式（`grub>`提示符），可通过以下两种方式应急恢复：
1.  **临时引导Windows**：重启电脑，按主板启动热键（本场景为F12）进入启动项选择菜单，手动选择「Windows Boot Manager」即可直接进入Windows系统，执行上述引导恢复命令即可复原（或者进入BIOS重新选择优先启动系统选项）；
2.  **手动引导Ubuntu**：在`grub>`提示符下，执行`ls`命令枚举磁盘分区，定位Ubuntu根分区后，手动加载内核与initrd镜像，执行`boot`命令即可启动Ubuntu系统，进入系统后执行`sudo update-grub`与`sudo grub-install`即可修复GRUB配置。

# 实验结果与方法效果验证
应用后：
```text
✔ 磁盘正常识别
✔ 分区正常显示
✔ 成功安装系统
```
- 基础识别修复：Linux 系统可正常识别 WD SN580 SSD，容量显示正常，可正常分区、格式化、读写数据，无 0B 识别异常问题；
- 长期稳定性：连续运行无掉盘、无读写错误，系统重启、跨系统冷重启均无异常；
- 双系统无冲突切换：两种引导方案均可实现双系统的无冲突切换，无引导覆盖、GRUB 救援模式等问题；
- 针对问题的三个层级根因分层解决，彻底根治 NVMe 设备识别异常与掉盘问题，而非临时缓解；
- 提供两种双系统引导方案，适配 “主用 Windows” 和 “频繁切换双系统” 两种不同的使用需求；
- 禁用 APST 功能会导致 NVMe SSD 全程保持满速工作状态，笔记本电池模式下续航有≤5% 的轻微下降，SSD 温度升高 1~2℃（完全在正常工作范围内），不会影响 SSD 的使用寿命；
- 适用以下情景：
品牌笔记本 / 台式机，UEFI BIOS 锁死 Intel RST RAID 模式，无法切换 AHCI；
消费级 DRAM-less NVMe SSD 在 Linux 系统中出现识别异常、0B 容量、掉盘、无法读写等问题；
Windows-Linux 双系统部署，存在跨系统重启掉盘、引导冲突等问题。

------

# 讨论
尽管无法完全确定该问题的唯一根因，
但通过禁用 NVMe APST 后问题稳定消失的实验结果，
可以认为该问题与 NVMe 电源状态切换过程中的不稳定性高度相关，
并可将其视为主要影响因素之一。

考虑到该问题涉及操作系统、设备固件、平台 PCIe 行为以及电源管理策略等多层交互，
其本质更接近于一种系统级耦合现象，
而非单一组件的缺陷所致。

因此，这里就不对单一原因做过度归因，
而是通过控制变量实验识别出能够稳定系统行为的关键因素，
并据此提出可复现、可验证的解决方案。