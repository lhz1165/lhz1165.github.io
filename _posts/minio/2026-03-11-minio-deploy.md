---
layout: post
title: "minio 集群部署"
category: minio
---

MinIO集群部署

# **MinIO集群部署**

## 1. 虚拟机节点配置,3台服务器测试

minio集群开始最好确定好几台机器，**不支持向集群中添加单个节点并进行自动调节的扩容方式**；如需扩容参考（对等扩容，联邦扩容），本教程只针对初次部署几个服务器组成集群，不包含扩容



| 主机名 | ip            | 操作系统      | minio 存储路径 |            |
| ------ | ------------- | ------------- | -------------- | ---------- |
| node01 | 192.168.1.106 | CentOS_7_2009 | /mnt/data      | 初始机     |
| node02 | 192.168.1.105 | CentOS_7_2009 | /mnt/data      | 高可用节点 |
| node03 | 192.168.1.107 | CentOS_7_2009 | /mnt/data      | 高可用节点 |

编辑 `/etc/hosts` 文件，为**所有节点**添加 IP 和主机名映射

vi /etc/hosts

```plain
# 添加各个节点和主机名的映射
192.168.1.106 node01
192.168.1.105 node02
192.168.1.107 node03
```

编辑 `/etc/default/minio` 文件，更新 `MINIO_VOLUMES` 配置，为**所有节点**指定存储路径：

vi /etc/default/minio

```plain
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="minioadmin"
MINIO_VOLUMES="http://node01/mnt/data http://node02/mnt/data http://node03/mnt/data"
MINIO_OPTS="--address :9000 --console-address :9001"
```

为**所有节点**设置唯一的主机名

```plain
hostnamectl set-hostname node01  # 在 node01 上执行
hostnamectl set-hostname node02  # 在 node02 上执行
hostnamectl set-hostname node03  # 在 node03 上执行
```

## 2. 挂载数据目录到独立磁盘

### 2.1 为节点添加一块独立的硬盘

MinIO 集群节点必须挂载到独立磁盘中（**测试增加了一块5G的**），MinIO的纠删码模式要求每个节点的数据目录必须挂载到 独立的物理磁盘(**所有独立挂载的磁盘大小必须相同)**

![image-202311141551122]({{ "/assets/minio/deploy/1.png" | absolute_url }})

### 2.2. 检查系统中已识别的磁盘

```plain
lsblk
```

`sdb`属于新添加的磁盘(5G),在linux的目录是/dev/sdb

![image-202311141551122]({{ "/assets/minio/deploy/2.png" | absolute_url }})

### 2.3. 格式化新磁盘

#### 2.3.1 磁盘分区

```plain
fdisk /dev/sdb
```

在fdisk交互界面，输入以下命令：

- `n`：创建新分区。
- `p`：选择主分区。
- `1`：分区编号为 1。
- 按 `Enter`：使用默认的起始扇区。
- 按 `Enter`：使用默认的结束扇区。
- `w`：保存并退出。

完成磁盘分区

![image-202311141551122]({{ "/assets/minio/deploy/3.png" | absolute_url }})



sdb1就是为新硬盘sdb新建的1分区

![image-202311141551122]({{ "/assets/minio/deploy/4.png" | absolute_url }})

#### 2.3.2 格式化分区为ext4系统

```plain
mkfs.ext4 /dev/sdb1
```

![image-202311141551122]({{ "/assets/minio/deploy/5.png" | absolute_url }})



### 2.4. 挂载数据目录到新磁盘（minio保存文件的目录）

```plain
#如果还未创建数据目录首先创建
mkdir /mnt/data
mount /dev/sdb1 /mnt/data
df -h /mnt/data
```

![image-202311141551122]({{ "/assets/minio/deploy/6.png" | absolute_url }})



### 2.5. 修改数据目录的所有者/组/权限

```plain
# 查看挂载后的目录权限和所有者
ls -ld /mnt/data
# 添加一个 minio-user 用户
groupadd -r minio-user
useradd -M -r -g minio-user minio-user

#修改目录权限
chown -R minio-user:minio-user /mnt/data
chmod -R 755 /mnt/data
# 查看设置权限后的目录权限和所有者
ls -ld /mnt/data
```



![image-202311141551122]({{ "/assets/minio/deploy/7.png" | absolute_url }})

###  2.6 配置每次开机自动挂载硬盘  

获取新分区/dev/sdb1的UUID

```plain
blkid /dev/sdb1
```

编辑/etc/fstab文件，在其末尾添加以下内容：

vi /etc/fstab

```plain
vim /etc/fstab
#将挂载目录所有者设为minio-user用户,将组设为minio-user组,权限为775文件为664
UUID=your-disk-uuid /mnt/data ext4 defaults 0 0
```



![image-202311141551122]({{ "/assets/minio/deploy/8.png" | absolute_url }})



检查数据目录是否已挂载

```plain
df -h /mnt/data
#如果未挂载在新分区,那么手动挂载一下
mount /dev/sdb1 /mnt/data
```



## 3. 启动前的检查

### 3.1.检查各节点之间的网络连通性

是否都能互相ping通

![image-202311141551122]({{ "/assets/minio/deploy/9.png" | absolute_url }})

### 3.2.检查MinIO的配置文件

VOLUMES是否有三个节点

![image-202311141551122]({{ "/assets/minio/deploy/10.png" | absolute_url }})

### 3.3.检查防火墙是否关闭

真实环境应该是开启防火墙，开放9000 和9001的端口，测试暂时关闭

```plain
systemctl stop firewalld
systemctl disable firewalld
```

### 3.3. 修改MinIO服务配置，设置开机启动

```plain
rm /usr/lib/systemd/system/minio.service
vi /usr/lib/systemd/system/minio.service
```



```plain
[Unit]
Description=MinIO
Documentation=https://minio.org.cn/docs/minio/linux/index.html
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio
[Service]
WorkingDirectory=/usr/local
User=minio-user
Group=minio-user
ProtectProc=invisible
EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
# MinIO RELEASE.2023-05-04T21-44-30Z adds support for Type=notify (https://www.freedesktop.org/software/systemd/man/systemd.service.html#Type=)
# This may improve systemctl setups where other services use `After=minio.server`
# Uncomment the line to enable the functionality
# Type=notify
# Let systemd restart this service always
Restart=always
# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536
# Specifies the maximum number of threads this process can create
TasksMax=infinity
# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no
[Install]
WantedBy=multi-user.target
# Built for ${project.name}-${project.version} (${project.name})
```

### 3.4. 重新加载服务配置

```plain
systemctl daemon-reload
```



## 4.启动 MinIO 集群

minio的二进制可执行文件放在 **/usr/local/bin** 目录下



![image-202311141551122]({{ "/assets/minio/deploy/11.png" | absolute_url }})



启动服务

```plain
systemctl stop minio
systemctl start minio
systemctl enable minio
#查看历史日志
journalctl -u minio -b
#查看实时日志
journalctl -u minio -f 
```



![image-202311141551122]({{ "/assets/minio/deploy/12.png" | absolute_url }})





## 4. 验证集群状态

访问 MinIO Web 控制台：

- 打开浏览器，访问 http://<任意节点IP>:9000（`http://192.168.1.106:9000`)。
- 使用 `minioadmin` 和 `minioadmin` 登录。

检查集群健康状态：

- 进入 `Monitoring`，检查集群健康状态。



![image-202311141551122]({{ "/assets/minio/deploy/13.png" | absolute_url }})



创建一个bucket，每个节点都会在挂载目录复制一份

![image-202311141551122]({{ "/assets/minio/deploy/14.png" | absolute_url }})



![image-202311141551122]({{ "/assets/minio/deploy/15.png" | absolute_url }})



![image-202311141551122]({{ "/assets/minio/deploy/16.png" | absolute_url }})



![image-202311141551122]({{ "/assets/minio/deploy/17.png" | absolute_url }})



## 补充

如果要为每台服务器扩容硬盘（加入新的硬盘）

 扩容后结构变成：  

```plain
node01:/mnt/data + /mnt/data2
node02:/mnt/data + /mnt/data2
node03:/mnt/data + /mnt/data2
```

**步骤**

1. 每台服务器加相同的硬盘 
2.  分区 + 格式化  
3.  挂载到新目录  /mnt/data2
4.  写入 fstab  
5.  修改 MinIO 配置  ( 所有机器 MINIO_VOLUMES 必须完全一样  )

原本

```plain
MINIO_VOLUMES="http://node01/mnt/data http://node02/mnt/data http://node03/mnt/data"
```

新

```plain
MINIO_VOLUMES="\
http://node01/mnt/data http://node01/mnt/data2 \
http://node02/mnt/data http://node02/mnt/data2 \
http://node03/mnt/data http://node03/mnt/data2"
```

1. 重启服务
