# 主机配置

## 内存

### 关闭内存交换

将/etc/fstab文件中所有设置为swap的设备关闭
```
swapoff -a
```
最大限度使用物理内存
```
# 修改/etc/sysctl.conf
vm.swappiness = 0
```

## 网络

### 开启ipvs

为kube-proxy使用ipvs模式加载内核模块
```
# 新增 /etc/modules-load.d/<custom>.conf
# 开机时加载如下内核模块
# 可以使用modeprobe临时加载
ip_vs
ip_vs_rr
ip_vs_sh
ip_vs_wrr
nf_conntrack_ipv4
```

## 存储

### 加载ceph RBD模块
```
# 记得配置到/etc/modules-load.d/<custom>.conf
modeprobe rbd
```

## 容器引擎

### docker daemon配置
```
{
	"registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn"
    ],
	"exec-opts": [
        "native.cgroupdriver=systemd"
    ],
	"log-opts": {
        "max-size": "64m"
    }
}
```