---
title: nfs server 部署
date: 2023-09-06 14:08:34
tags:
- k8s
---

以ubuntu安装为例
### nfs安装
安装 NFS 内核服务器包，这实际上会将其变成 NFS 服务器
```
sudo apt update
sudo apt install nfs-kernel-server
```

验证 nfs-server 服务是否正在运行
```
sudo systemctl status nfs-server
```

### 创建 NFS 共享目录
创建 /mnt/nfs 文件目录，用于共享
```
sudo mkdir -p /mnt/nfs
```

分配以下目录所有权和权限,这些权限是递归的，将应用于创建的所有文件和子目录
```
sudo chown nobody:nogroup /mnt/nfs
sudo chmod -R 777 /mnt/nfs
```

### 设置访问权限
创建 NFS 目录共享并分配所需的权限和所有权后，需要允许客户端系统访问 NFS 服务器。通过编辑在安装 nfs-kernel-server 软件包期间创建的 /etc/exports 文件来实现这一点
```
sudo vim /etc/exports
```

设置运行访问的 IP 范围, * 表示允许任意 ip 访问。
```
/mnt/nfs 192.168.32.0/24(rw,sync,no_root_squash,no_subtree_check)
```
常用选项：
- ro：客户端挂载后，其权限为只读，默认选项；
- rw:读写权限；
- sync：同时将数据写入到内存与硬盘中；
- async：异步，优先将数据保存到内存，然后再写入硬盘；
- Secure：要求请求源的端口小于1024

用户映射：
- root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的匿名用户；
- no_root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的root用户；
- all_squash:全部用户都映射为服务器端的匿名用户；
- anonuid=UID：将客户端登录用户映射为此处指定的用户uid；
- anongid=GID：将客户端登录用户映射为此处指定的用户gid

### 导出共享目录
```
sudo exportfs -a
```

确定导出列表，可以使用以下命令显示导出列表
```
sudo exportfs -s
# /mnt/nfs  192.168.32.0/24(rw,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

### 配置防火墙规则
查看防火墙状态
```
sudo ufw status
# Status: inactive
```

提示防火墙未激活，很好，否则，我们需要添加如下规则
```
sudo ufw allow from 192.168.32.0/24 to any port nfs
```

### 安装 NFS 客户端
为了让所有的 Worker 节点都可以使用 NFS 服务器，需要在每个 Worker 节点上安装 NFS 客户端。
现在登录到 Worker 节点更新包索引并安装 nfs-common
```
sudo apt update
sudo apt install nfs-common
```

### 测试文件挂载
创建挂载的文件夹
```
mkdir -p /nfs-data
```
执行以下命令挂载 nfs 服务器上的共享目录到本机路径`/nfs-data`
```
mount -t nfs -o nolock,vers=4 192.168.32.24:/nfs /nfs-data
```

参数解释：
- mount：挂载命令
- o：挂载选项
- nfs :使用的协议
- nolock :不阻塞
- vers : 使用的NFS版本号
- IP : NFS服务器的IP（NFS服务器运行在哪个系统上，就是哪个系统的IP）
- /nfs: 要挂载的目录（Ubuntu的目录）
- /nfs-data : 要挂载到的目录（开发板上的目录，注意挂载成功后，/mnt下原有数据将会被隐藏，无法找到）

查看挂载
```
df -h
```

卸载挂载
```
umount /nfs-data
```

检查 nfs 服务器端是否有设置共享目录
```
# showmount -e $(nfs服务器的IP)
showmount -e 172.16.106.205

# 输出结果如下所示
Export list for 172.16.106.205:
/nfs *
```

查看nfs版本

```
# 查看nfs服务端信息(服务端执行)
nfsstat -s

# 查看nfs客户端信息（客户端执行）
nfsstat -c
```

写入一个测试文件
```
echo "hello nfs server" > /nfs-data/test.txt
```

在 nfs 服务器上执行以下命令，验证文件写入成功：
```
cat /nfs/test.txt
```