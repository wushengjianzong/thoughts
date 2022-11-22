# 当Kubernetes遇见Macvlan

最近在研究KubeVirt和Virtink两个项目，计划做一个Operator管理虚拟机。Kubernetes API有望成为云计算基础设施管理的事实标准，IaaS关注计算、存储和网络，相比OpenStack，Kubernetes是一个可塑性更强的项目。  
在Kubernetes管理虚拟机的场景下，容器网络就是虚拟机网络。目前没有做双网卡方案，我需要把容器网络拉平到物理网络，方便直接登录虚拟机。为了保持架构简单清爽，我放弃了multus、coredns和kube-proxy。这个方案仅为兴趣使然，并不适用于生产环境。

## 实验准备
我的实验设备是一台Dell机架服务器，它的基础信息如下。如果你打算自己实验，下文中用到内网IP的地方换成自己的IP就可以。

|系统|主机名|内网IP|网卡|
|---|---|---|---|
|ubuntu 20.04|lyr620|192.168.0.5/16|eno1|

## 安装Kubernetes
你可以使用[Sealos](​docs.sealos.io/zh-Hans/)丝滑安装Kubernetes。Sealos是一个轻量的二进制工具，可以搞定Kubernetes安装中的一些常见痛点。包括但不限于Linux配置、组件镜像和高可用集群。  Sealos的安装和使用请参考官方文档，这里不做赘述。我使用的Sealos版本为4.1.3，Kubernetes版本为1.23.13，下面是我的Clusterfile。  

```yaml
apiVersion: apps.sealos.io/v1beta1
kind: Cluster
metadata:
  name: default
spec:
  # 我的环境只有一台机器
  hosts:
  - ips:
    - 192.168.0.5:22
    roles:
    - master
    - amd64
  # 匹配sealos 4.1.3版本的kubernetes镜像
  image:
  - labring/kubernetes:v1.23.13-4.1.3
  # 请确保ssh可以通过公钥直接登录
  ssh:
    pk: /root/.ssh/id_rsa
    port: 22

---

apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  # 这里设置为和物理网络相同的网段
  podSubnet: 192.168.0.0/16
---

apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
# 跳过coredns和kube-proxy的安装
skipPhases:
- addon/coredns
- addon/kube-proxy
```
你只需要一条命令就可以安装Kubernetes集群。
```bash
$ sealos apply -f Clusterfile
```
macvlan虚拟的设备插入到容器network namespace以后，其中的流量就不经过主机network namespace了，依赖iptables/ipvs实现的service全然无用，kube-proxy和coredns自然也成了摆设。  
我是单节点Kubernetes，这里需要去掉master上的taint，使得Pod可以调度到master。
```bash
$ kubectl taint node lyr620 node-role.kubernetes.io/master-
```

## 配置CNI Plugin
你需要下载官方出品的[CNI Plugins](​github.com/containernetworking/plugins/releases)，我使用了1.1.1版本。
将下载的压缩包解压到Kubernetes默认的CNI插件目录。
```bash
$ mkdir -p /opt/cni/bin
$ tar -zxvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin
```
在/etc/cni/net.d目录下创建一个00-default.conflist文件，写入如下内容。
```json
{
    "cniVersion": "0.3.1",
    "name": "default",
    "plugins": [
        {
            "type": "macvlan",
            "master": "eno1",
            "ipam": {
                "type": "host-local",
                "ranges": [
                    [
                        {
                            "subnet": "192.168.0.0/16",
                            "rangeStart": "192.168.1.2",
                            "rangeEnd": "192.168.1.254",
                            "gateway": "192.168.0.1"
                        }
                    ]
                ],
                "routes": [
                    {"dst": "0.0.0.0/0"}
                ]
            }
        }
    ]
}
```
我的机器上只有eno1这个网卡连接到了公司内网，所以指定macvlan插件在eno1上创建子设备分配给容器。这里的IPAM子网配置到主机网络192.168.0.0/16，但是将IP池限制在192.168.1.0/24这个子网的IP范围。  
之前也尝试过将subnet设置为192.168.1.0/24，主机为之添加相应的路由，但是这个方案没能成功，其中有些问题以我的内功暂时还玩不动......

## 验证Pod网络
检查Kubernetes Node是否就绪。
```bash
$ kubectl get nodes
NAME     STATUS   ROLES                  AGE     VERSION
lyr620   Ready    control-plane,master   5h33m   v1.23.13
```
创建一个Nginx Deployment，观察Pod是否Running，查看容器的路由表。
```bash
$ kubectl create deployment nginx --image nginx:stable-alpine

$ kubectl get pods -o wide
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE    IP   
default       nginx-7bd849c599-cbgqn           1/1     Running   0          5s     192.168.1.2

$ kubectl exec nginx-7bd849c599-cbgqn -it -- ip route
default via 192.168.0.1 dev eth0
192.168.0.0/16 dev eth0 scope link  src 192.168.1.2
```
虽然Pod已经Running，但这时从主机ping容器是不通的。
```bash
$ ping -c 3 192.168.1.2
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
From 192.168.0.5 icmp_seq=1 Destination Host Unreachable
From 192.168.0.5 icmp_seq=2 Destination Host Unreachable
From 192.168.0.5 icmp_seq=3 Destination Host Unreachable

--- 192.168.1.2 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2027ms
pipe 3
```
从容器ping主机也不行。
```bash
$ kubectl exec -it nginx-7bd849c599-cbgqn -- ping -c 3 192.168.0.5
PING 192.168.0.5 (192.168.0.5): 56 data bytes

--- 192.168.0.5 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```
为了满足Kubernetes主机和容器网络互通的需求，我们需要在主机的network namespace创建并启动一个bridge模式的macvlan子设备。
```bash
$ ip link add eno1.host link eno1 type macvlan mode bridge
$ ip link set dev eno1.host up
```
然后在主机上写一条直通Pod IP的路由。
```bash
$ ip route add 192.168.1.2 dev eno1.host
$ ip route
default via 192.168.0.1 dev eno1 proto static
192.168.0.0/16 dev eno1 proto kernel scope link src 192.168.0.5
192.168.1.2 dev eno1.host scope link
```
这时容器和主机就互通了，你不但可以ping通Pod IP，而且访问Pod IP还可以看见Nginx的网页。
```bash
$ curl http://192.168.1.2
<!DOCTYPE html>
<html>
...
</html>
```

## 参考资料
+ [图解几个与Linux网络虚拟化相关的虚拟网卡-VETH/MACVLAN/MACVTAP/IPVLAN](https://blog.csdn.net/dog250/article/details/45788279)  
+ [kubernetes的clusterip机制调研及macvlan网络下的clusterip坑解决方案](https://zhuanlan.zhihu.com/p/67384482)
+ [Docker使用macvlan网络容器与宿主机的通信过程](https://smalloutcome.com/2021/07/18/Docker-%E4%BD%BF%E7%94%A8-macvlan-%E7%BD%91%E7%BB%9C%E5%AE%B9%E5%99%A8%E4%B8%8E%E5%AE%BF%E4%B8%BB%E6%9C%BA%E7%9A%84%E9%80%9A%E4%BF%A1%E8%BF%87%E7%A8%8B/)
