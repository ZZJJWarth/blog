# 在hvisor中支持模拟Virtio PCI设备

时间：2026/03/30

作者：曾俊

## Virtio Transport Options

Virtio协议使用不同的总线进行数据传输，目前一共支持三种传输选项：

1. Virtio over PCI
2. Virtio over MMIO
3. Virtio over Channel IO

其中MMIO传输在hvisor中已经实现，而Channel IO是Virtio协议为不支持PCI和MMIO的，基于S/390的虚拟机特别支持的数据传输方式

在一般的架构以及linux内核中，Virtio PCI是Virtio最为常见的数据传输协议

当前hvisor可以在qemu中将一个Virtio-PCI设备直通给root linux，当然也可以通过配置将Virtio-PCI设备直通给non root linux
它的本质是qemu实现了功能完整的Virtio设备，当它以适当的接口暴露给linux的PCI驱动时，linux自然可以枚举并挂载这些Virtio设备

如果是在物理设备，那么物理平台上是不存在Virtio设备的，所以non root linux是不能使用Virtio PCI设备的，这需要hvisor模拟一个Virtio PCI设备才能实现

Hvisor对于设备虚拟化的路线是：hvisor向non root linux暴露一些虚拟设备（一般是Virtio设备，比如virtio-rng），当non root linux访问这些设备时，hvisor会捕获这些访问，并将这些请求发送给root linux，由root linux完成请求再通知non root linux

所以当前工作主要是为了实现hvisor**向non root linux提供virtio-pci设备的框架**

## Hvisor向本工作提供了什么抽象

参见仓库：<https://github.com/syswonder/hvisor/blob/dev/src/pci/vpci_dev/mod.rs>

hvisor的PCI虚拟化工作已经解决了PCI协议相关的工作，为本工作提供了一个干净的抽象
为了在虚拟的PCI总线中暴露一个设备，你需要：

1. 完成一个read_cfg函数，它用于读取虚拟PCI设备的配置空间
2. 完成一个write_cfg函数，它用于修改虚拟PCI设备的配置空间
3. 完成一个vdev_init函数，它用于虚拟PCI设备初步初始化

下面是PCI设备的配置空间示意图：

![img](img/20260330_VirtioPCISupport/Pci-config-space.svg.png)

### PCI设备配置空间

这里介绍一些和virtio相关的概念

**Device ID**和**Vender ID**，PCI协议使用这两个字段来唯一识别设备

比如说：VGA compatible controller: Intel Corporation Raptor Lake-S UHD Graphics 这个设备，它的vender id为1E0F，device id为001A

在PCI协议中，所有的Virtio设备的vender id都为0x1AF4，而device id则由Virtio协议规定，可参考下图：

![alt text](img/20260330_VirtioPCISupport/image.png)

上图是传统的Virtio设备ID，现代的Virtio设备ID一般是从0x1041开始，比如network card在上图中是第一个设备，那么它的现代设备ID就是0x1041,而entropy source是第六个，那么它的现代设备ID就是0x1046,以此类推

---------------
**Base Address Register**，简称为BAR寄存器，是PCI协议重要的申请资源方式

实际上PCI设备的配置空间的大小是非常有限的，如果要支撑起设备大量的数据传输，仅仅依靠配置空间是完全不够的
有些设备需要更多的配置，就连这些设备自身的特殊配置，PCI配置空间也完成不了
设想一下，操纵设备对于内核而言，很多情况下就是操纵寄存器，设备将自己的功能以寄存器的形式暴露给内核，内核只需要读写这些寄存器就可以实现操纵设备
实际上直观的想法是，设备告诉内核“我的A寄存器在地址x1，我的B寄存器在地址x2...”，但是内核是内存空间的管理者，它不能允许设备自行分配地址
所以PCI协议提供了BAR寄存器

首先设备需要思考自己需要多少地址空间来映射自己的寄存器，比如说