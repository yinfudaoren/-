## 虚拟机准备

通过[Vagrant](<https://www.vagrantup.com/>)准备三台虚拟机。vagrantfile如下，使用vagrant up命令安装。

```vagrant
Vagrant.require_version ">= 1.6.0"

boxes = [
    {
        :name => "manager-0",
        :eth1 => "192.168.200.10",
        :mem => "1024",
        :cpu => "4"
    },
    {
        :name => "worker-0",
        :eth1 => "192.168.200.11",
        :mem => "1024",
        :cpu => "2"
    },
    {
        :name => "worker-1",
        :eth1 => "192.168.200.12",
        :mem => "1024",
        :cpu => "2"
    }
]

Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"

  boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "vmware_fusion" do |v|
          v.vmx["memsize"] = opts[:mem]
          v.vmx["numvcpus"] = opts[:cpu]
        end

        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
          v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
        end

        config.vm.network :private_network, ip: opts[:eth1]
      end
  end
end
```

## 环境准备

### 基础准备

由于vagrant machine比较精简所以先安装必要软件

```shell
yum install -y vim wget
```

配置docker yum源（阿里），安装及启动。现使用的docker版本为docker-ce-18.09（k8s官网现在支持的最新版本）

```shell
cd /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum install -y docker-ce-18.09.8-3.el7

systemctl enable docker && systemctl start docker
```

关闭swap分区（若机器关机重启需再次执行此命令，可以执行使其开机初始化）

```shell
swapoff -a
```

更改hosts

```shell
vim /etc/hosts
```

添加以下内容

```shell
192.168.200.10  manager-0 manager-0
192.168.200.11  worker-0 worker-0
192.168.200.12  worker-1 worker-1
```

### K8s准备

配置K8s源,使用的是阿里源

```shell
vim /etc/yum.repos.d/kubernetes.repo
```

将以下内容加入文件
```shell
[kubernetes]

name=Kubernetes

baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/

enabled=1

gpgcheck=1

repo_gpgcheck=1

gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

安装kubeadm等

```shell
yum install -y kubectl kubelet kubeadm
```

设置内核参数

```shell
cat <<EOF > /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-ip6tables = 1

net.bridge.bridge-nf-call-iptables = 1

net.ipv4.ip_forward = 1

EOF
```

使配置生效

```shell
sysctl --system
```

启动kubelet

```shell
systemctl enable kubelet && systemctl start kubelet
```

## 搭建K8s集群

### 主节点（manger）

使用kubeadm初始化

```shell
kubeadm init --apiserver-advertise-address=192.168.200.10 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=192.168.0.0/16 --service-dns-domain=cluster.local --ignore-preflight-errors=Swap --ignore-preflight-errors=NumCPU
```

初始化成功应该会出现如下情景

```shell
You can now join any number of machines by running the following on each node........
```

执行以下命令，使得使用kubectl更为方便

```shell
cd
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

配置网络插件，选择使用calico,所以cidr地址为192.168.0.0/16

```shell
curl https://docs.projectcalico.org/v3.8/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

### 从节点（worker-0和-1）

在主节点执行如下命令获取加入集群指令后再在从节点执行

```shell
kubeadm token create --print-join-command
```

### 确定集群状态

代pod启动完毕后在主节点执行如下指令

```shell
kubectl get node
```

若各个节点状态正常即成功搭建集群

### .bashrc

配置.bashrc文件，个人选用kc作为快捷命令

```shell
alias kc='kubectl'
```

配置自动补全

```shell
source <(kubectl completion bash | sed s/kubectl/kc/g)
```

