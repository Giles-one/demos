
- 速通教程
```shell
sudo qemu-system-x86_64 \
  -m 1G \
  -kernel /boot/vmlinuz \
  -initrd /boot/initrd.img \
  -append 'console=ttyS0' \
  -nographic
```
- 执行命令应该会进入'(initramfs) '的终端。
- sudo权限是为了能够读取到`/boot/vmlinuz`和`/boot/initrd.img`。
- initrd指定的是intramfs。[initrd](https://docs.kernel.org/admin-guide/initrd.html#:~:text=initrd%20provides%20the%20capability%20to,mounted%20from%20a%20different%20device.)。由于是在ram执行中，在终端里的修改不会反映到`initrd.img里`。

#### Kernel

- 参数里的kernel选项指定的vmlinuz是一个压缩过的bzImage，包含了一定的boot部分。如果去逆向vmlinuz是找不到内核的内容的，可以使用[extract-vmlinux](https://github.com/torvalds/linux/blob/master/scripts/extract-vmlinux)去解压真正的kernel。


#### initramfs

##### 打包
```
find . -print0 \
| cpio --null --format newc -ov \
| gzip -9 > ../initrd.cpio.gz
```
##### 解包

```
gunzip < ../initrd.cpio.gz \
| cpio -ivd
```

- 现在的/boot/initrd.img是压缩过的，可以使用unmkinitramfs工具来解码。

#### qcow2磁盘镜像
---
> qcow is a file format for disk image files used by QEMU, a hosted virtual machine monitor.[1] It stands for "QEMU Copy On Write" and uses a disk storage optimization strategy that delays allocation of storage until it is actually needed. Files in qcow format can contain a variety of disk images which are generally associated with specific guest operating systems. Three versions of the format exist: qcow, qcow2 and qcow3[2] which use the .qcow, .qcow2 and .qcow3 file extensions, respectively.

qcow2是一个磁盘镜像，就像物理的固态硬盘或者机械硬盘，可以在其之上分多个区，每个区都可以制作不同的文件系统。

有两种方法进行qcow2磁盘镜像的修改。一种可以把qcow2磁盘镜像转化成raw的格式，使用losetup把raw格式的磁盘镜像与一个loop的块设备联系起来。之后就可以直接处理loop块设备。另一种是使用qemu提供的工具qemu-nbd(network block device)，直接把qcow2磁盘镜像与一个nbd的块设备联系起来，直接处理nbd块设备。

##### option1
```shell
qemu-img -f qcow2 /path/to/disk.qcow2 -O raw /path/to/disk.raw
losetup -f /path/to/disk.raw
losetup -a
# ...
# /dev/loop11: [2053]:1353547 (/path/to/disk.raw)
# ...
fdisk -l /dev/loop11
# ...
# Device        Boot Start     End Sectors  Size Id Type
# /dev/loop11p1       2048 1023999 1021952  499M 83 Linux
# ...
mount /dev/loop11p1 ./fs
```
- 其中disk.raw 是稀疏文件（sparse file），多数文件系统都是支持这种稀疏特性的，使得 **disk.raw** 文件逻辑大小与磁盘大小是不一样的。可以使用这样的命令去测试。
```shell
ls -lh /path/to/disk.raw
du -h  /path/to/disk.raw
```

```shell
umount ./fs
losetup -d /dev/loop11
qemu-img convert -f raw /path/tp/disk.raw -O qcow2 /path/to/disk.qcow2
```

##### option2

```shell
modprobe nbd
qemu-nbd --connect=/dev/nbd0 /path/to/disk.qcow2
fdisk -l /dev/nbd0
# ...
# Device      Boot Start      End  Sectors Size Id Type
# /dev/nbd0p1       2048 41943039 41940992  20G 83 Linux
# ...
mount /dev/nbd0p1 ./fs
```
```shell
umount ./fs
qemu-nbd --disconnect /dev/nbd0
rmmod nbd
```

##### 制作
如果需要制作一个qcow2的磁盘镜像，可以qemu-img的create子命令去制作。
```shell
qemu-img create -f qcow2 /path/to/disk.qcow2 20G
qemu-nbd --connect=/dev/nbd0 /path/to/disk.qcow2
fdisk /dev/nbd0
```

```shell
qemu-img create -f raw   /path/to/disk.raw   20G
losetup -f /path/to/disk.raw
fdisk /dev/loop11
qemu-img convert -f raw /path/to/disk.raw -O qcow2 /path/to/disk.qcow2
```

#### 解包和打包initramfs

##### pack

```
find . -print0 \
| cpio --null --format newc -ov \
| gzip -9 > ../initrd.cpio.gz
```

##### unpack

```
gunzip < ../initrd.cpio.gz \
| cpio -ivd
```

#### 端之间进行文件以文件夹的传输

A:
```shell
tar cfv - folder file \
| nc IP PORT
```

B:
```shell
nc -l PORT \
| tar xfv -
```
- 通常命令行参数 **-**  表示标准输入标准输出，使得多条命令之间能够更好地使用管道传递数据流。eg (`strace echo hello 2>&1 | vim -`)
- 当网络链路情况比较好时，tar在归档时不使用压缩，可以节省时间。如果链路较差，可以加入 z 的参数进行压缩和解压。


#### qemu配置网络

- qemu网络包括两部分，一个是虚拟网络设备，另一个是网络后端。两个相互结合实现qemu内虚拟机上网。

- qemu主要有两种上网模式，一个是user模式（默认，不需要root权限），另一个是tap模式（-netdev tap，需要在宿主机上创建tap0的桥接网卡，创建时需要root权限）

##### user模式配置

- 默认的时候是user模式，也即不添加任何网络配置选项时使用的是user模式，此时guest主机的虚拟网络设备是一块intel的e1000 PCI网卡，host主机的网络后端是user-mode的网络协议栈。以下三种启动方式是等效的。

1. `qemu-system-i386 -m 256 -hda disk.img`
2. `qemu-system-i386 -m 256 -hda disk.img -net nic -net user`
3. `qemu-system-i386 -m 256 -hda disk.img -netdev user,id=network0 -device e1000,netdev=network0,mac=52:54:00:12:34:56`

注: 2中的`-net nic -net user`是老版本qemu遗留的选项，推荐使用`-netdev`和`-device`选项。

下图是user模式网络的拓扑图。
![](https://wiki.qemu.org/images/9/93/Slirp_concept.png)

- 在配置guest IP时应该时应该配置`10.0.2.x/24`的网络，其中网关是`10.0.2.2`，并且qemu为你模拟了一台DNS服务器`10.0.2.3`。
- 网络架构类似于在网关上做了NAT，无法在Host网络ping通guest网络。但是在guest网络可以ping通外部网络。
- 在host上监听的端口，会对应到网关`10.0.2.2`上的相应端口。guest主机可以访问网关的端口，而连接host主机的服务。
- qemu提供端口转发，可以把guest的对应端口，转发到host主机的某个端口。
- dns服务器无需自己启动，可以在guest网络可以ping通DNS`10.0.2.3`。

这是清晰的启动命令。
```shell
sudo qemu-system-x86_64 \
  -m 1G \
  -kernel /boot/vmlinuz \
  -initrd /boot/initrd.img \
  -netdev user,id=net1 \
  -device e1000,netdev=net1 \
  -append 'console=ttyS0' \
  -nographic
```
#### tap模式配置

```
tunctl -t tap0 -u `whoami`
```