# 安装与升级集群

## kubeadm安装集群

### 配置参考
[kubeadm配置](https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2)
[kubeproxy配置](https://godoc.org/k8s.io/kube-proxy/config/v1alpha1)

### 示例：默认网关指向内网
kubeadm init配置文件
```
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.19.4
apiServer:
  extraArgs:
    service-node-port-range: 1-65535
  certSANs:
  - "139.196.220.104"
networking:
  podSubnet: 192.168.0.0/16
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers

---

apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```
> podSubnet必须和组网方案中的配置上下文一致，不要和其他网络重合

kubeadm初始化命令
```
kubeadm init --config <custom>.yaml [--dry-run]
```
