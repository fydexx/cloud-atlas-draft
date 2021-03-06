# KVM hypervisor和VM-extensions

早期的KVM技术提供的是类似VMWare的完全虚拟化，对于guest操作系统模拟成完整的物理服务器。而半虚拟化，如Xen，称为Paravirtualization则提供更高的性能，但需要修改guest操作系统。完全虚拟化可以运行没有修改的guest系统如Windows。为了能够使用完全虚拟化，需要CPU的虚拟化扩展或使用模拟器。

要确定物理主机是否支持VM扩展，在x86平台，如AMD-V或Intel VT-X

```
egrep -c '(vmx|svm)' /proc/cpuinfo
```

如果输出是0，则表示没有找到CPU中`vmx`或`svm`的标记。在这种没有VM扩展的CPU环境下，则需要使用KVM结合QEMU-emulation，则性能较差。本文的配置是使用物理主机BIOS和CPU都支持激活VM-extension。

# 安装KVM

> 如果需要图形界面，可以结合X11 forwarding来使用`virt-manager`，即同时安装`virt-manager`（图形管理程序），`xauth`（X11访问认证）和`dejavu-lgc-sans-fonts`（运行X11的基本字体）

```
sudo yum install libvirt virt-install qemu-kvm virt-manager xauth dejavu-lgc-sans-fonts
```

也可以安装纯字符终端管理工具

```
sudo yum install libvirt virt-install qemu-kvm
```

> 默认安装会启用一个`NAT`模式的bridge`virbr0`

启动激活`libvirtd`服务

```
systemctl enable libvirtd && systemctl start libvirtd
```

# 安装OS

## guest OS磁盘

默认的VM创建的磁盘在`/var/lib/libvirt/images`，需要确保有足够空间存放磁盘镜像。

KVM支持多种VM镜像格式，本文案例采用raw文件格式，即创建10GB的vm磁盘会实际在物理主机磁盘上生成10GB的磁盘文件。

## guest OS网络

默认VM只访问一个NAT模式的网桥网络，采用私有网段`192.168.122.0`，如果要使用bridge模式对外直接提供服务访问（虚拟机直接使用网络中的IP地址连接局域网）可创立独立的bridge网桥。

## Firewalld

在RHEL 6中，默认的包过滤和转发服务称为`iptables`，而在RHEL 7，默认是`firewalld`，提供了和iptables相同的的包过滤和转发功能，但是实现了动态规则以及增强的功能如network zone，可以实现更为伸缩性的网络结构。

注意在RHEL7中依然有iptables工具，不过这个工具是通过使用firewalld来和内核包过滤器通讯。

## SELinux

如果使用SELinux增强模式，常见问题是要注意不能使用非默认的目录来创建VM镜像。如果你要使用非`/var/lib/libvirt/images`目录，则要创建目录的安全上下文。例如，选择`/vm-images`来存放镜像：

* 创建目录

```
mkdir /vm-images
```

* 安装`policycoreutils-python`软件包（包含了`segmange`这个SELinux工具）

```
yum -y install policycoreutils-python
```

* 设置目录的安全上下文以及目录下的内容：

```
semange fcontext --add -t virt_image_t '/vm-images(/.*)?'
```

* 验证

```
semanage fcontext -l | grep virt_image_t
```

显示输出类似

```
/var/lib/imagefactory/images(/.*)? all files  system_u:object_r:virt_image_t:s0
/var/lib/libvirt/images(/.*)?      all files  system_u:object_r:virt_image_t:s0
/vm-images(/.*)?                   all files  system_u:object_r:virt_image_t:s0
```

* 恢复安全上下文

```
restorecon -R -v /vm-images
```

验证上下文更改：

```
ls -aZ /vm-images
```

* 如果需要将 `/vm-images` 作为samba或者NFS共享，则需要设置SELinux Booleans

```
setsebool -P virt_use_samba 1
setsebool -P virt_use_nfs 1
```

# 创建VM

* 举例：安装CentOS 7操作系统

使用`virt-install`创建，这个工具分为交互和非交互模式，以下命令使用非交互方式创建CentOS 7 x64虚拟机，名字是`vm1`使用1个Virtual CPU，1G内存和10GB磁盘

```
virt-install \
  --network bridge:virbr0 \
  --name centos7 \
  --ram=1024 \
  --vcpus=1 \
  --disk path=/var/lib/libvirt/images/centos7.img,size=10 \
  --graphics none \
  --location=http://mirrors.163.com/centos/7/os/x86_64/ \
  --extra-args="console=tty0 console=ttyS0,115200"
```

* `--graphics none` 这个参数表示不使用VNC来访问VM的控制台，而是使用VM串口的字符控制台。如果希望使用X window的图形界面来安装VM操作系统，则可以忽略这个参数
* `--location=http://mirrors.163.com/centos/7/os/x86_64/` 这个是指定通过网络的CentOS 7安装目录进行安装。如果你使用本地的iso安装，可以修改成 `--cdrom /root/CentOS-7-x86_64-DVD-1511.iso`
* `--extra-args="console=tty0 console=ttyS0,115200"` 这个`extra-args`是传递给OS installer的内核启动参数(注意：这个`extra-args`参数只能用于`--location`)。这里因为需要连接到VM的串口，所以要传递内核对应参数启动串口。此外，可以指定kickstart文件，这样就可以不用交互而自动完成安装，如

```
--extra-args="ks=http://my.server.com/pub/ks.cfg console=tty0 console=ttyS0,115200"
```

> 遇到一个奇怪问题，上述命令使用`--graphics vnc`虽然可以用vnc连接，但是到了`Started D-Bus System Message Bus`之后字符界面停止了。

改为直接通过下载的iso镜像安装初始操作系统

```
virt-install \
  --network bridge:virbr0 \
  --name centos7 \
  --ram=2048 \
  --vcpus=1 \
  --disk path=/var/lib/libvirt/images/centos7.img,size=10 \
  --graphics vnc \
  --cdrom=/var/lib/libvirt/images/CentOS-7-x86_64-DVD-1511.iso
```

## 使用现有磁盘镜像创建虚拟机

* `virt-install`也支持从现有存在的磁盘镜像创建KVM VM，使用`--import`参数：

```
virt-install -n vmname -r 2048 --os-type=windows   --os-variant=win7 \
--disk /kvm/images/disk/vmname_boot.img,device=disk,bus=virtio \
--network bridge=virbr0,model=virtio --vnc --noautoconsole --import
```

> `–disk /kvm/images/disk/vmname_boot.img,device=disk,bus=virtio`是一个完整定义磁盘镜像路径以及通过`,`分隔的磁盘参数选项。注意，使用`virtio`速度最快但是需要确保windows安装驱动，并且太陈旧的Linux版本不支持。
>
> `-w bridge=virbr0,model=virtio` 连接`virbr0`虚拟网桥，并使用`virtio`驱动以获得更好网络性能。如果操作系统不支持，则使用`e1000`或者`rtl8139`。
>
> `--noautoconsole` 安装不会自动代开`virt-viewer`来查看控制台已完成安装。这对远程使用SSH系统有用。

* 以前安装过的`dev7.img`镜像重新启用

```
virt-install \
  --network bridge:virbr0 \
  --name dev7 \
  --os-type=linux \
  --os-variant=rhel7.3 \
  --ram=2048 \
  --vcpus=1 \
  --disk path=/var/lib/libvirt/images/dev7.img,device=disk,bus=virtio \
  --network bridge=virbr0,model=virtio \
  --graphics vnc --noautoconsole --import
```

* 多网卡并且使用以前安装的`dev7.img`镜像启动案例：

```
virt-install \
  --name dev7 \
  --os-type=linux \
  --os-variant=rhel7.3 \
  --ram=8192 \
  --vcpus=4 \
  --disk path=/var/lib/libvirt/images/dev7.img,device=disk,bus=virtio \
  --network bridge=virbr0,model=virtio \
  --network bridge=vr0,model=virtio \
  --graphics vnc --noautoconsole --import
```

* 多磁盘的安装启动案例

```
virt-install -n vmname -r 2048 --os-type=windows --os-variant=win7 \
--disk /kvm/images/disk/vmname_boot.img,device=disk,bus=virtio \
--disk /kvm/images/disk/vmname_data_1.img,device=disk,bus=virtio \
-w bridge=virbr0,model=virtio --vnc --noautoconsole --import
```

* LVM磁盘镜像

```
virt-install -n vmname -r 2048 --os-type=windows --os-variant=win7 \
--disk /dev/vg_name/lv_name,device=disk,bus=virtio \
-w bridge=virbr0,model=virtio --vnc --noautoconsole --import
```

* 非virtio（使用IDE和e1000网卡模拟）

```
virt-install -n vmname -r 2048 --os-type=windows --os-variant=win7 \
--disk /kvm/images/disk/vmname_boot.img,device=disk,bus=ide \
-w bridge=virbr0,model=e1000 --vnc --noautoconsole --import
```

# 一些其他安装案例

```
virt-install \
  --network bridge:virbr0 \
  --name centos6 \
  --ram=2048 \
  --vcpus=1 \
  --disk path=/var/lib/libvirt/images/centos6.img,size=10 \
  --graphics vnc \
  --cdrom=/var/lib/libvirt/images/CentOS-6.9-x86_64-netinstall.iso
```

```
virt-install \
  --network bridge:virbr0 \
  --name centos5 \
  --ram=2048 \
  --vcpus=1 \
  --disk path=/var/lib/libvirt/images/centos5.img,size=10 \
  --graphics vnc \
  --os-type=linux --os-variant=rhel5.11 \
  --cdrom=/var/lib/libvirt/images/CentOS-5.11-x86_64-netinstall.iso
```

```
virt-install \
  --network bridge:virbr0 \
  --name freebsd10 \
  --os-type=freebsd --os-variant=freebsd10.1 \
  --ram=2048 \
  --vcpus=1 \
  --disk path=/var/lib/libvirt/images/freebsd10.img,size=10 \
  --graphics vnc \
  --cdrom=/var/lib/libvirt/images/FreeBSD-10.3-RELEASE-amd64-disc1.iso
```

```
virt-install \
  --network bridge:virbr0 \
  --name ubuntu16 \
  --os-type=ubuntu --os-variant=ubuntu16.04 \
  --ram=2048 \
  --vcpus=1 \
  --disk path=/var/lib/libvirt/images/ubuntu16.img,size=10 \
  --graphics vnc \
  --cdrom=/var/lib/libvirt/images/ubuntu-16.10-server-amd64.iso
```

* 举例：安装windows 2012操作系统

```
virt-install \
   --name=win2012 \
   --os-type=windows \
   --network bridge:virbr0 \
   --disk path=/var/lib/libvirt/images/win2012.img,size=10 \
   --cdrom=/root/win2012.iso \
   --graphics vnc --ram=2028 \
   --vcpus=1
```

提示报错无法打开image文件

```
ERROR    internal error: process exited while connecting to monitor: 2016-10-10T16:56:22.361462Z qemu-kvm: -drive file=/root/win2012.iso,if=none,id=drive-ide0-0-1,readonly=on,format=raw: could not open disk image /root/win2012.iso: Could not open '/root/win2012.iso': Permission denied
```

这个报错应该就是和前面提到的默认安全上下文要求image在指定目录下，所以将iso文件移动到`/var/lib/libvirt/images`，然后修改执行命令

```
virt-install \
   --name=win2012 \
   --os-type=windows \
   --network bridge:virbr0 \
   --disk path=/var/lib/libvirt/images/win2012.img,size=16 \
   --cdrom=/var/lib/libvirt/images/win2012.iso \
   --graphics vnc --ram=2028 \
   --vcpus=1
```

* 使用virtio驱动方式(paravirtual)安装Windows

```
virt-install \
   --name=win2012desktop \
   --os-type=windows --os-variant=win2k12r2 \
   --boot cdrom,hd \
   --network=default,model=virtio \
   --disk path=/var/lib/libvirt/images/win2016desktop.img,size=16,format=qcow2,bus=virtio,cache=none \
   --disk device=cdrom,path=/var/lib/libvirt/images/win2012.iso \
   --disk device=cdrom,path=/var/lib/libvirt/images/virtio-win.iso \
   --boot cdrom,hd \
   --graphics vnc --ram=2048 \
   --vcpus=2
```

> `osinfo-query os`可以获得`--os-variant`所有支持的参数（参考`man virt-install`）
>
> 当前`virtio-win.iso`驱动签名在Windows 2016无法通过，需要先安装非`virtio`模式，安装完成后，关闭驱动签名功能重启系统，并增加`virtio`模式磁盘，使得系统能够安装`virtio`驱动 - [KVM环境安装Windows virtio驱动](../../performance/kvm_performance_tunning_in_action/install_windows_with_virtio)

> 参考：[Creating guests with the virt-install command](http://www.ibm.com/support/knowledgecenter/en/linuxonibm/liabp/liabpusingvirtinstall.htm)

```
virt-install --connect qemu:///system \
    --name win7vnc --ram 2048 --vcpus=2 --cpuset=auto \
    --disk path=win7.img,bus=virtio,size=100,format=qcow2 \
    --network=network=default,model=virtio,mac=RANDOM \
    --graphics vnc,port=5900
    --disk device=cdrom,path=../../isos/virtio-win-0.1-81.iso \
    --disk device=cdrom,path=../../isos/win7_sp1_ult_64bit/Windows\ 7\ SP1\ Ultimate\ \(64\ Bit\).iso \
    --os-type=windows --os-variant=win7 --boot cdrom,hd 
```

> 参考 [windows 7 as kvm guest installation with virtio drivers - detected virtio scsi disk shows wrong capacity](http://serverfault.com/questions/631317/windows-7-as-kvm-guest-installation-with-virtio-drivers-detected-virtio-scsi-d)

# VNC访问

## VNC监听

> 注意：默认的vnc端口只允许本地访问，在启动服务的命令中有 `-vnc 127.0.0.1:0`，可以检查如下

```
sudo virsh vncdisplay win2012
```

显示是本地端口监听

```
127.0.0.1:0
```

## VNC客户端连接

经过反复测试，我发现在Mac OS X平台上，很多vnc viewer客户端都不能正常连接到qemu提供的VNC界面，折腾了很长时间。

最后发现，开源的 [Chicken](http://sourceforge.net/projects/chicken/) 是最好的VNC客户端，可以支持访问VNC界面。

## 修改VNC监听网络接口

如果要监听所有端口，可以修改 `/etc/libvirt/qemu.conf`

```
# VNC is configured to listen on 127.0.0.1 by default.
# To make it listen on all public interfaces, uncomment
# this next option.
#
# NB, strong recommendation to enable TLS + x509 certificate
# verification when allowing public access
#
vnc_listen = "0.0.0.0"
```

然后重启`libvirtd`服务

```
systemctl restart libvirtd
```

> 以上参考 [virt-install ignoring vnc port/listen?](http://serverfault.com/questions/567716/virt-install-ignoring-vnc-port-listen)

注意，默认开启的iptables防火墙需要开启端口访问，例如：

```
iptables -A INPUT -p tcp --dport 5900 -j ACCEPT
```

> CentOS 7开始推荐使用firewalld来管理防火墙，而不是直接使用iptables

----

# 开启VNC监听（不推荐）

参考 [KVM Virtualization: Start VNC Remote Access For Guest Operating Systems](http://www.cyberciti.biz/faq/linux-kvm-vnc-for-guest-machine/)

* 方法一

在启动虚拟机时候使用参数 `-vnc 0.0.0.0:1`

* 方法二

编辑 `/etc/libvirt/qemu/win2012.xml`

```
<graphics type='vnc' port='-1' autoport='yes' keymap='en-us'/>
```

重启 libvirtd

```
/etc/init.d/libvirtd restart
virsh shutdown win2012
virsh start win2012
```

> 这个方法测试可行

* 方法三

```
sudo virsh edit win2012
```

动态修改 `win2012.xml` 修改添加 `listen='0.0.0.0'`后再次重启虚拟机

```
<graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'/>
```

## 通过ssh tunnel使用(推荐)

不过上述修改方法都不是很好，存在安全隐患，并且需要打通防火墙端口。

更好的方法是使用 ssh tunnel，可参考 [HowTo: Tunneling VNC Connections Over SSH](http://www.cyberciti.biz/tips/tunneling-vnc-connections-over-ssh-howto.html)

```
ssh -L 5901:localhost:5901 -N -f -l username SERVER_IP
```

举例：

```
ssh -L 5901:127.0.0.1:5901 -N -f -l rocky 192.168.1.100
```

> `-N` 参数表示不执行远程命令，这个参数是用于端口转发的
>
> `-f` 参数表示后台执行ssh

然后就可以使用vnc viewer客户端访问本地回环地址 127.0.0.1 来访问远程服务器的VNC来进行进一步安装。

> 对于Mac系统，内建有一个VNC客户端（但是 **很不幸** `不支持libvirtd所使用的VNC`），可以尝试 `open vnc://127.0.0.1:5901` 来访问VNC服务器。

如果要批量开启一批端口转发(例如51个端口转发，可以支持服务器上51个虚拟机的VNC访问)：

```
SERVER=192.168.1.100
PORTS=`echo {5900..5950}`
for i in $PORTS;do
    ssh -L $i:127.0.0.1:$i -N -f $SERVER
done
```

> 安装好虚拟机初始操作系统之后，可以[clone KVM虚拟机](clone_kvm_vm)

# NAT网络的调整

上述安装配置适合作为一个个人使用的开发工作站，但是如果需要对外提供访问服务，默认的NAT网络就存在限制了。

此外，默认的NAT网络使用了DHCP分配地址，所以启动虚拟机可能每次获得的IP地址不一定相同，对外做端口映射提供服务就存在问题。

* [KVM/libvirt Host主机NAT网络固定DHCP分配和端口映射](kvm_libvirt_static_ip_for_dhcp_and_port_forwarding)

# 参考

* [Install and use CentOS 7 or RHEL 7 as KVM virtualization host](http://jensd.be/207/linux/install-and-use-centos-7-as-kvm-virtualization-host)
* [KVM Virtualization in RHEL 7 Made Easy](https://linux.dell.com/files/whitepapers/KVM_Virtualization_in_RHEL_7_Made_Easy.pdf)
* [Chapter 9. Installing a Fully-virtualized Windows Guest](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Host_Configuration_and_Guest_Installation_Guide/sect-Virtualization_Host_Configuration_and_Guest_Installation_Guide-Windows_Installations-Installing_Windows_XP_as_a_fully_virtualized_guest.html)
* [KVM Guests: Using Virt-Install to Import an Existing Disk Image](https://www.itfromallangles.com/2011/03/kvm-guests-using-virt-install-to-import-an-existing-disk-image/)