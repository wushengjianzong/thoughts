# 当Kubernetes遇见Macvlan——实现CNI路由插件

## 实验背景

上次做了一个[基于Macvlan的Kubernetes网络方案](https://wushengjianzong.github.io/thoughts/2022/kubernetes-macvlan-network.html)，Macvlan插件执行以后，Pod和Host网络还没有互通。于是我们创建了一个Bridge模式的Macvlan子设备，手写从Bridge到Pod的路由规则，使得Pod和Host网络互通。现在我们实现一个CNI插件，通过链式执行自动完成上面的事情。

## 创建项目

CNI插件使用golang开发，我们随便创建一个文件夹，在其中`go mod init <module>`一下就可以了。下面是我的项目结构。
```bash
.
├── go.mod
├── go.sum 
├── main.go
├── Makefile
└── net.d
    └── 00-default.conflist
```
从`go.mod`来看，我们的项目只依赖两个外部模块，一个是CNI插件框架，一个用来操作Linux网络设备。
```go
...
require (
    // CNI插件框架
	github.com/containernetworking/cni v1.0.1
    // 操作Linux网络设备
	github.com/vishvananda/netlink v1.1.0
)
...
```
`00-default.conflist`是我们的CNI插件配置，最后会放到`/etc/cni/net.d`下面。
```json
{
    "cniVersion": "0.4.0",
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
        },
        {
            "type": "route",
            "master": "eno1",
            "bridge": "eno1.host"
        }
    ]
}
```

## 代码实现
这里是我们用到的的所有go module。
```go
import (
	"encoding/json"
	"net"

	"github.com/containernetworking/cni/pkg/skel"
	"github.com/containernetworking/cni/pkg/types"
	"github.com/containernetworking/cni/pkg/types/040"
	"github.com/containernetworking/cni/pkg/version"
	"github.com/vishvananda/netlink"
)
```

首先我们需要声明一个结构体，用来解析我们自定义插件的配置，以及接收macvlan插件的执行结果。
```go
type PluginConfig struct {
    // 组合代替继承
	types.NetConf

	Master      string `json:"master"`
	Bridge      string `json:"bridge"`
	MasterIndex int    `json:"-"`
	BridgeIndex int    `json:"-"`
}
```
每次调用CNI插件的ADD和DEL方法时，会通过stdin传入插件配置和上一步的执行结果，我们实现一个parseConfig方法解析传入的内容。
```go
func parseConfig(stdin []byte) (*PluginConfig, error) {
	var config PluginConfig

    // 解析当前插件的配置
    if err := json.Unmarshal(stdin, &config); err != nil {
		return nil, err
	}

    // 解析上一步执行结果
	if err := version.ParsePrevResult(&config.NetConf); err != nil {
		return nil, err
	}

	return &config, nil
}
```
然后实现一个initBridge方法，在Host网络空间创建Bridge模式的Macvlan子设备。
```go
func initBridge(config *PluginConfig) error {
    // 找到macvlan主设备的索引
	masterLink, err := netlink.LinkByName(config.Master)
	if err != nil {
		return err
	}
	config.MasterIndex = masterLink.Attrs().Index

    // 检查当前是否存在bridge
	bridgeLink, err := netlink.LinkByName(config.Bridge)
    if err == nil {
		config.BridgeIndex = bridgeLink.Attrs().Index
		return nil
	}
    if _, ok := err.(netlink.LinkNotFoundError); !ok {
        return err
    }
	
    // 创建bridge模式的macvlan子设备
	if err := netlink.LinkAdd(&netlink.Macvlan{
		LinkAttrs: netlink.LinkAttrs{
			Name:        config.Bridge,
			ParentIndex: config.MasterIndex,
		},
		Mode: netlink.MACVLAN_MODE_BRIDGE,
	}); err != nil {
		return err
	}

    // 找到创建的bridge并启动它
	bridgeLink, err = netlink.LinkByName(config.Bridge)
	if err != nil {
		return err
	}
	if err := netlink.LinkSetUp(bridgeLink); err != nil {
		return err
	}
	config.BridgeIndex = bridgeLink.Attrs().Index
	
    return nil
}
```
实现CNI插件的ADD方法，根据Pod IP在Host上创建路由。
```go
func cmdAdd(args *skel.CmdArgs) error {
    // 解析插件配置和上一步的结果
	config, err := parseConfig(args.StdinData)
	if err != nil {
		return err
	}

    // 获取上一步（macvlan插件）的执行结果
	prevResult, err := types040.GetResult(config.PrevResult)
	if err != nil {
		return err
	}

    // 检查或初始化macvlan bridge
    if err := initBridge(config); err != nil {
		return err
	}

    // 获得Pod IP
	podIP := prevResult.IPs[0].Address
	podIP.Mask = net.CIDRMask(32, 32)

    // 创建路由
	route := &netlink.Route{
		Dst:       &podIP,
		Scope:     netlink.SCOPE_LINK,
		LinkIndex: config.BridgeIndex,
	}
	if err := netlink.RouteAdd(route); err != nil {
		return err
	}

    // 透传macvlan插件的执行结果
    // 这里必须主动打印一下，否则执行会报错
	return types.PrintResult(prevResult, config.CNIVersion)
}
```
实现CNI插件的DEL方法，Pod被回收时删除路由。
```go
func cmdDel(args *skel.CmdArgs) error {
    // 解析插件配置和上一步的结果
	config, err := parseConfig(args.StdinData)
	if err != nil {
		return err
	}

    // 获取ADD操作的的执行结果
	prevResult, err := types040.GetResult(config.PrevResult)
	if err != nil {
		return err
	}

    // 获得Pod IP
	podIP := prevResult.IPs[0].Address
	podIP.Mask = net.CIDRMask(32, 32)

    // 删除路由
	route := &netlink.Route{
		Dst:       &podIP,
		Scope:     netlink.SCOPE_LINK,
		LinkIndex: config.BridgeIndex,
	}
	return netlink.RouteDel(route)
}
```
CNI插件的CHECK方法不用实现，声明一下就可以。
```go
func cmdCheck(args *skel.CmdArgs) error {
	return nil
}
```
下面是CNI插件的执行入口，在CNI框架的支持下，一行代码足矣。
```go
func main() {
	skel.PluginMain(cmdAdd, cmdCheck, cmdDel, version.All, "CNI route plugin 0.0.1")
}
```

## 功能测试

没有将插件应用到Kubernetes时，我们用`cnitool`这个工具对插件进行测试，可以现场装一个。
```bash
go install github.com/containernetworking/cni/cnitool
```
构建route插件，放到`/opt/cni/bin`这个目录。
```bash
go build -o /opt/cni/bin/route main.go
```
使用实验配置测试ADD功能。
```bash
$ NETCONFPATH=$PWD/net.d CNI_PATH=/opt/cni/bin cnitool add default /var/run/netns/testing
```
CNI插件链被成功执行，产生了一个JSON输出。这是ADD操作的执行结果，会被缓存到CNI工作目录（一般是`/var/lib/cni`）。
```json
{
    "cniVersion": "0.4.0",
    "interfaces": [
        {
            "name": "eth0",
            "mac": "ea:a8:70:cd:18:62",
            "sandbox": "/var/run/netns/testing"
        }
    ],
    "ips": [
        {
            "version": "4",
            "interface": 0,
            "address": "192.168.1.2/16",
            "gateway": "192.168.0.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        }
    ],
    "dns": {}
}
```
我们在`/var/lib/cni/networks/default`下面可以看到IPAM分配的Pod IP。
```bash
$ ls /var/lib/cni/networks/default
192.168.1.2  last_reserved_ip.0  lock

# 上一个被分配的IP
$ cat /var/lib/cni/networks/default/last_reserved_ip.0 
192.168.1.2
```
在`/var/lib/cni/results`下可以看到CNI ADD的结果缓存，执行DEL操作时缓存的结果会被传入。
```bash
$ ls /var/lib/cni/results
default-cnitool-77383ca0a0715733ca6f-eth0

$ cat /var/lib/cni/results/default-cnitool-77383ca0a0715733ca6f-eth0 | jq
{
  "kind": "cniCacheV1",
  "containerId": "cnitool-77383ca0a0715733ca6f",
  "config": ...(base64后的原始配置)
  "ifName": "eth0",
  "networkName": "default",
  "result": {
    ...(刚才那一大坨JSON执行结果)
  }
}
```
现在我们测试DEL功能。
```bash
$ NETCONFPATH=$PWD/net.d CNI_PATH=/opt/cni/bin cnitool del default /var/run/netns/testing
```
虽然什么输出都没有，但是执行是成功的。这时你再去`/var/lib/cni`下面查看ADD的执行结果，很多文件已经被删除了。  

## 集成测试

上一步我们已经将route插件放到了`/opt/cni/bin`目录，这里只要覆盖一下Kubernetes的CNI配置。
```bash
$ cp $PWD/net.d/00-default.conflist /etc/cni/net.d
```
删除从前创建的Macvlan Bridge。
```bash
ip link del eno1.host
```
然后我们创建一个Nginx Deployment，观察Pod IP。
```bash
$ kubectl create deployment nginx --image nginx:stable-alpine 
deployment.apps/nginx created

$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP             NODE   
nginx-7bd849c599-vkppk   1/1     Running   0          28s   192.168.1.3   lyr620
```
测试Pod到Host的网络连通性。
```bash
$ ping -c 3 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=0.172 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 192.168.1.3: icmp_seq=3 ttl=64 time=0.089 ms

--- 192.168.1.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2036ms
rtt min/avg/max/mdev = 0.057/0.106/0.172/0.048 ms


$ kubectl exec nginx-7bd849c599-vkppk -it -- ping -c 3 192.168.0.5
PING 192.168.0.5 (192.168.0.5): 56 data bytes
64 bytes from 192.168.0.5: seq=0 ttl=64 time=0.149 ms
64 bytes from 192.168.0.5: seq=1 ttl=64 time=0.118 ms
64 bytes from 192.168.0.5: seq=2 ttl=64 time=0.132 ms

--- 192.168.0.5 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.118/0.133/0.149 m
```
查看Host上创建的Macvlan子设备。
```bash
$ ip link list
...
47: eno1.host@eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether be:c3:ad:8b:1f:53 brd ff:ff:ff:ff:ff:ff
```
查看Host上为Pod创建的路由。
```bash
$ ip route list
...
192.168.1.3 dev eno1.host scope link
```

## 参考资料

+ [CNI示例插件](https://github.com/containernetworking/plugins/tree/main/plugins/sample)
+ [SBR插件](https://github.com/containernetworking/plugins/tree/main/plugins/meta/sbr)
+ [使用cnitool](https://www.cni.dev/docs/cnitool/)


