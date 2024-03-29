# 容灾一体机部署文档


# 一、准备工作
## 1.1 环境要求
| 主机类型 | 操作系统 | 硬件要求 |
| --- | --- | --- |
| 服务端 | linux CentOS7.8及以上 | CPU 8核，内存32G，200G硬盘 |
| 客户端 | CentOS oracle 或 Sqlserver 2012 |  |

> 备注：
> 1、DR的压缩包和相关的依赖工具请登录堡垒机[http://192.168.1.105](http://192.168.1.105)下载；

## 1.2 环境配置
关闭服务端防火墙
```bash
sed -i 's/SELINUX=enforcing/SELINUX=diabled/g' /etc/selinux/config
iptables -F
setenforce 0
systemctl stop firewalld
systemctl disable firewalld
```
## 1.3 依赖包
配置本地 yum 源
```bash
# 如果是虚拟机，编辑虚拟机设置，分配系统镜像给CDROM，勾选已连接
# 建立挂载点，挂载CDROM
mkdir /mnt/cdrom/
mount /dev/cdrom /mnt/cdrom/

# 如果是物理机，上传镜像到某目录，以下示例为上传到/root下
# 建立挂载点，挂载CDROM
mkdir /mnt/cdrom
mount -o loop /root/CentOS-7.7-x86_64-DVD-1908.iso /mnt/cdrom

# 移走官方自带的联网repo源
mkdir /etc/yum.repos.d/repoBackup
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/repoBackup

# 生成本地repo源
cat <<EOF > /etc/yum.repos.d/dvd.repo
[dvd]
name=dvd
baseurl=file:///mnt/cdrom
gpgcheck=0
enabled=1
EOF

# 重新生成yum缓存
yum clean all
yum makecache
```
安装相关依赖和所需工具
```bash
yum install -y wget vim-enhanced net-tools tree telnet kernel-devel zlib-devel libuuid libblkid-devel libssl-devel openssl-devel libudev-devel libattr-devel libaio-devel python2-devel python-cffi python-setuptools libffi-devel gcc gcc-c++ elfutils-libelf-devel autoconf automake libtool
```
# 二、服务端部署
## 2.1 安装存储
### 2.1.1 编译安装 zfs
上传好zfs源码包，开始编译安装
```bash
tar -xzvf zfs-0.8.4.tar.gz
cd zfs-0.8.4
./configure
make -j $(nproc) && make install
```
以下是需要启用的service
```bash
[root@lbserver ~]# systemctl list-unit-files | grep enabled | grep zfs
zfs-import-cache.service                    enabled
zfs-mount.service                           enabled
zfs-share.service                           enabled
zfs-volume-wait.service                     enabled
zfs-zed.service                             enabled
```
启用上述的service
```bash
systemctl enable zfs-import-cache.service
systemctl enable zfs-mount.service
systemctl enable zfs-share.service
systemctl enable zfs-volume-wait.service
systemctl enable zfs-zed.service
systemctl enable zfs-import.target
systemctl enable zfs.target
```
修正编译安装产生的错误
```bash
mkdir /etc/zfs
sed -i 's!ConditionPathExists=/usr/local/etc/zfs/zpool.cache!ConditionPathExists=/etc/zfs/zpool.cache!g' /usr/lib/systemd/system/zfs-import-cache.service
sed -i 's!ExecStart=/usr/local/sbin/zpool import -c /usr/local/etc/zfs/zpool.cache -aN!ExecStart=/usr/local/sbin/zpool import -c /etc/zfs/zpool.cache -aN!g' /usr/lib/systemd/system/zfs-import-cache.service
systemctl daemon-reload
```
### 2.1.2 优化zfs参数
```bash
mkdir /etc/zfs.modules.d

cat <<EOF > /etc/zfs.modules.d/zfs.conf
# max active per top-level vdev
options zfs zfs_vdev_max_active=1024
# min/max active requests for leaf-level vdevs
options zfs zfs_vdev_async_read_max_active=32
options zfs zfs_vdev_async_read_min_active=2
options zfs zfs_vdev_scrub_max_active=4
options zfs zfs_vdev_scrub_min_active=1
options zfs zfs_vdev_sync_read_max_active=32
options zfs zfs_vdev_sync_read_min_active=2
options zfs zfs_vdev_sync_write_max_active=32
options zfs zfs_vdev_sync_write_min_active=2
options zfs zfs_vdev_async_write_max_active=32
options zfs zfs_vdev_async_write_min_active=2
# Limut device utilization to 85% to control fragmentation
options zfs zfs_mg_noalloc_threshold=15
EOF
```
添加开机自动加载zfs模块
```bash
cat <<EOF > /etc/sysconfig/modules/zfs.modules
modprobe zfs
EOF

chmod +x /etc/sysconfig/modules/zfs.modules
```
### 2.2.3 启动zfs
加载zfs内核
```bash
modprobe zfs
```
启动zfs服务
```bash
systemctl start zfs.target
systemctl start zfs-import.target
systemctl start zfs-import-cache.service
systemctl start zfs-mount.service
systemctl start zfs-share.service
systemctl start zfs-volume-wait.service
systemctl start zfs-zed.service
```
检查zfs内核加载状况
```bash
lsmod | grep zfs
zfs                 3986613 12
zunicode             331170 1 zfs
zlua                 147429 1 zfs
zcommon               89551 1 zfs
znvpair               94388 2 zfs,zcommon
zavl                  15167 1 zfs
icp                  288913 1 zfs
spl                  104299 5 icp,zfs,zavl,zcommon,znvpair
```
### 2.1.4 安装 nfs 
```bash
yum -y install rpcbind nfs-utils
```
启动 nfs 并添加开机自启
```bash
systemctl enable nfs
systemctl start nfs
```
### 安装 samba

```bash
yum install -y samba samba-client
```

修改配置文件

```bash
cat << EOF > /etc/samba/smb.conf 
[global]
    log file = /var/log/samba/log.%m
    max log size = 5000
    security = user
    guest account = root
    map to guest = Bad User
    load printers = yes
    cups options = raw

[share]
    comment = share directory目录
    path = /opt/smb/
    directory mask = 0755
    create mask = 0755
    guest ok = yes
    writable = yes
EOF
```

启动服务

```
mkdir /opt/smb
systemctl start smb
systemctl enable smb
```

添加windows PE 镜像

```bash
mkdir /opt/smb
mv windowspe.iso /opt/smb/windowspe.iso
```

## 2.2 安装虚拟化

检测系统是否支持虚拟化
```bash
grep -Ei 'vmx|svm' /proc/cpuinfo 
```
如果有输出内容，则支持，其中intel cpu支持会有vmx，amd cpu支持会有svm

### 2.2.1 配置网卡

```bash
cd /etc/sysconfig/network-scripts/ 
cp ifcfg-ens160 ifcfg-br0
```

编辑ens160

```bash
DEVICE=ens160
HWADDR=00:0C:29:55:A7:0A
TYPE=Ethernet
UUID=2be47d79-2a68-4b65-a9ce-6a2df93759c6
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=none
BRIDGE=br0
```

编辑br0

```bash
DEVICE=br0
#HWADDR=00:0C:29:55:A7:0A
TYPE=Bridge
#UUID=2be47d79-2a68-4b65-a9ce-6a2df93759c6
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=static
IPADDR=192.168.2.80
NETMASK=255.255.255.0
GATEWAY=192.168.2.1
DNS1=114.114.114.114
```

重启网卡

```bash
/etc/init.d/network restart
```

### 2.2.2 安装 kvm
```bash
yum install -y kvm virt-*  libvirt  bridge-utils qemu-img
```

- kvm:软件包中含有KVM内核模块，它在默认linux内核中提供kvm管理程序
- libvirt:安装虚拟机管理工具，使用virsh等命令来管理和控制虚拟机。
- bridge-utils:设置网络网卡桥接。
- virt-*:创建、克隆虚拟机命令，以及图形化管理工具virt-manager
- qemu-img:安装qemu组件，使用qemu命令来创建磁盘等。

检查kvm模块是否加载
```bash
lsmod |grep kvm
```
结果输出：
```bash
kvm_intel              55496  3 
kvm                   337772  1 kvm_intel 
```
如果没有，需要执行，还没有就重启一下试试
```bash
modprobe kvm-intel
```
修改配置文件`/etc/libvirt/libvirtd.conf`

```bash
sed -i 's!#listen_tls = 0!listen_tls = 0!g' /etc/libvirt/libvirtd.conf
sed -i 's!#listen_tcp = 1!listen_tcp = 1!g' /etc/libvirt/libvirtd.conf
sed -i 's!#tcp_port = "16509"!tcp_port = "16509"!g' /etc/libvirt/libvirtd.conf
sed -i 's!#listen_addr = "0.0.0.0"!listen_addr = "0.0.0.0"!g' /etc/libvirt/libvirtd.conf
sed -i 's!#user = "root"!user = "root"!g' /etc/libvirt/qemu.conf
sed -i 's!#group = "root"!group = "root"!g' /etc/libvirt/qemu.conf
sed -i 's!#unix_sock_group = "libvirtd"!unix_sock_group = "libvirtd"!g' /etc/libvirt/libvirtd.conf
sed -i 's!#unix_sock_rw_perms = "0770"!unix_sock_rw_perms = "0770"!g' /etc/libvirt/libvirtd.conf
sed -i 's!#auth_unix_ro = "none"!auth_unix_ro = "none"!g' /etc/libvirt/libvirtd.conf
sed -i 's!#auth_unix_rw = "none"!auth_unix_rw = "none"!g' /etc/libvirt/libvirtd.conf
sed -i 's!#auth_tcp = "sasl"!auth_tcp = "none"!g' /etc/libvirt/libvirtd.conf
sed -i 's!#max_clients = .*!max_clients = 1024!g' /etc/libvirt/libvirtd.conf
sed -i 's!#min_workers = .*!min_workers = 50!g' /etc/libvirt/libvirtd.conf
sed -i 's!#max_workers = .*!max_workers = 200!g' /etc/libvirt/libvirtd.conf
sed -i 's!#max_requests = .*!max_requests = 1000!g' /etc/libvirt/libvirtd.conf
sed -i 's!#max_client_requests = .*!max_client_requests = 200!g' /etc/libvirt/libvirtd.conf
```

修改配置文件`/etc/sysconfig/libvirtd`：

```bash
sed -i 's!#LIBVIRTD_ARGS="--listen"!LIBVIRTD_ARGS="--listen"!g' /etc/sysconfig/libvirtd
```

重启 libvirtd

```bash
systemctl restart libvirtd
```

## 2.3 安装 qemu

> 需要相关软件包: ninja-linux.zip, Python-3.6.12.tar.gz, qemu-5.2.0.tar.gz

### 2.3.1安装 ninja

qemu-5.2.0 编译时使用构建工具 ninja, 解压并安装

```bash
yum install -y unzip
unzip ninja-linux.zip && mv ninja /usr/sbin/ninja
```

验证

```bash
ninja --version
1.10.2
```

### 2.3.2 安装 python3.6

编译安装 qemu-5.2.0 依赖 Python3.6 及以上的版本。所以首先安装 Python3.6。这里选择编译安装。

解压

```bash
tar -xvf Python-3.6.12.tar.gz
```

安装 openssl

pip 下载是需要 ssl 支持，所以下载 openssl

```bash
yum install -y openssl openssl-devel zlib-devel bzip2-devel bzip2
```

编译安装

```bash
cd Python-3.6.12 
./configure --prefix=/usr/local/python3 --enable-optimizations 
make -j8 build_all && make -j8 install
```

设置软链接

```bash
ln -s /usr/local/python3/bin/python3 /usr/bin/python3 
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
```

验证

```bash
# python3 
Python 3.6.12 (default, Dec 27 2020, 07:52:33)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux 
Type "help", "copyright", "credits" or "license" for more information. 
>>>
```

### 2.3.3 安装 qemu-5.2 

完成以上步骤之后就可以开始安装qemu了。

安装依赖

```bash
yum install -y pkgconfig-devel glib2-devel pixman-devel libcap-ng-devel libcap-devel libattr-devel librbd1 librdb1-devel libxml2 libxml2-devel glusterfs-api-devel numactl-devel libiscsi-devel libnfs-devel
```

```bash
rpm ivh libnfs*
```

编译安装

```bash
tar -xvf qemu-5.2.0.tar.xz
cd qemu-5.2.0
./configure --enable-debug --target-list=x86_64-softmmu --enable-kvm --enable-virtfs --enable-numa --enable-libudev --enable-libxml2 --enable-libnfs --enable-libiscsi --enable-glusterfs --enable-rbd
make && make install
```

## 2.4 安装数据库

使用 yum 安装本地rpm包

```bash
tar -xvf postgresql-13.tar.gz
cd postgresql-13
yum localinstall -y postgresql13-contrib-13.4-1PGDG.rhel7.x86_64.rpm postgresql13-libs-13.4-1PGDG.rhel7.x86_64.rpm postgresql13-server-13.4-1PGDG.rhel7.x86_64.rpm postgresql13-13.4-1PGDG.rhel7.x86_64.rpm
```

配置数据库数据⽂件路径

```bash
root$ mkdir /usr/local/pgsql
root$ chown postgres /usr/local/pgsql
root$ su - postgres
# 配置初始⽤户密码和数据库连接验证⽅式
postgres$ /usr/pgsql-13/bin/initdb -D /usr/local/pgsql/data -W -A md5
#$ password: ?
```

配置远程连接

```bash
root$ vim /usr/local/pgsql/data/postgresql.conf
...
listen_addresses = 'localhost'
```

配置 postgres 用户登录权限

```bash
vim /usr/local/pgsql/data/pg_hba.conf
...
host    all     all             ::1/128                 md5
```

修改启动脚本

```bash
sed -i 's#Environment=PGDATA=/var/lib/pgsql/13/data/#Environment=PGDATA=/usr/local/pgsql/data/#g' /usr/lib/systemd/system/postgresql-13.service
systemctl daemon-reload
/usr/pgsql-13/bin/postgresql-13-setup initdb
systemctl restart postgresql-13
systemctl enable postgresql-13
```

创建数据库

```bash
psql -U postgres -h localhost -W


postgres=# create database dr encoding 'UTF-8';
```

## 2.5 安装服务端组件

解压软件包

```
tar -xvf dradm-linux-amd64-v1.0.1-tar.gz
cd dradm
./dradm init --advertise-address=$IP --postgres-dns="host=localhost,user=postgres,password=$PWD,port=5432"
```

> $IP 为服务端ip地址，$PWD 为 pgsql postgres 用户的登录密码

输出结果:

```bash
...
2021-10-09 12:53:39 | [INFO] discovery system uuid: 4222ada5-e123-9401-75ae-30078f1052bb
```

> 根据 uuid 获取证书

浏览器访问: `http://$ip:8000`

开机启动 gpmd

```bash
systemctl enable gpmd
```

## 2.6 安装存储组件

```bash
./dradm join virt --server-address=$sip
```

> $sip 为服务端ip地址

# 三、添加客户端

## 3.1 环境配置

保证客户端和服务端网络通畅，打开防火墙

## 3.2 安装客户端

### 3.2.1 linux 安装

```bash
tar -xvf dradm-linux-amd64-v1.0.1-tar.gz
cd dradm
./dradm join agent --server-address=$sip
```

> $sip 为服务端ip地址

开机启动 gpmd

```bash
systemctl enable gpmd
```

### 3.2.2 windows 安装

解压 `dradm-windows-amd64-v1.0.1.zip`

```bash
.\dradm.exe join agent --server-address=$sip
```

> $sip 为服务端ip地址

