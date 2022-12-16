---
layout: post
title: 安装Kubernetes并部署SDN(OVN-KUBE)网络组件
date: 2022-12-16 10:22:15
categories: 网络运维
tags: k8s sdn ovn-kube
---

## 安装Kubernetes并部署SDN(OVN-KUBE)网络组件

所有节点主机的OS为debian 10.10.0，所有操作在root用户环境中执行。

### 准备工作
1. 关闭交换分区

    如果没有交换分区的话，则不需要执行这步操作。
    ```
    # 关闭交换分区
    swapoff -a
    # 修改系统配置，以在重启时就不加载交换分区
    sed -i "/swap/s/^/# /g" /etc/fstab
    ```
    *提示：*

    交换分区信息可通过 __free -h__ 命令的结果，
    或输出 __cat /proc/swaps__ 文件的信息来查看。
1. 修正搜索路径

    修改 __/etc/profile__ 文件，设置 __root__ 用户的执行程序搜索路径。如果准备使用 __sudo__ 执行相关命令，可以跳过这一步。
    ```
    ## 替换搜索路径内容
    sed -i "s@games:/usr/games@sbin:/usr/sbin:/sbin@g" /etc/profile
    ## 激活搜索路径
    source /etc/profile
    ```
1. 加载内核模块和配置网络转发机制

    查看已加载的内核模块列表，如果 __br_netfilter__ 没有加载的话，通过添加内核配置文件来加载它。此外，还需要配置 __iptables__ 的转发机制。
    ```
    # 查看
    lsmod | grep br_netfilter
    # 创建内核模块加载文件，文件名可以自定义
    tee /etc/modules-load.d/k8s.conf << EOF
    br_netfilter
    EOF
    #
    # 创建网络配置文件，文件名可以自定义
    tee /etc/sysctl.d/k8s.conf << EOF
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF
    # 激活配置
    sysctl --system
    ```

1. 主机名称解析

    此步骤的主要目的是通过 __/etc/hosts__ 文件配置主机名称解析。如果有DNS服务器支持的话，可以忽略这一步。

    尽管不是必需的，但还是建议使用 __hostnamectl__ 命令为K8S集群的每个节点赋予一个有意义的名称。
    ```
    #查看主机名称信息
    hostnamectl status
    # 主节点执行
    hostnamectl set-hostname "k8s-master"
    # 工作节点执行
    # 命令中的[n]表示一个序号，可以任意编制，但工作节点的名称必须唯一。
    hostnamectl set-hostname "k8s-worker[n]"
    #
    # 在所有节点，有多少个节点
    # DN = domain.name
    # nodeNames =
    echo "xxx.xxx.xxx.xxx k8s-master.domain.name  k8s-master" >> /etc/hosts
    ```
1. 安装依赖包


### Docker-ce（20.10.7）的安装和配置

因是基于 __docker-ce_20.10.7__ 来进行安装和运维测试的，其它版本的可用性未知。

1. 下载并安装docker-ce

    从官网下载所需的 __deb__ 文件至本地目录 __/opt/docker-20.10.7__ 中，并安装它们。

    ```
    mkdir /opt/docker-20.10.7
    cd /opt/docker-20.10.7
    wget https://download.docker.com/linux/debian/dists/buster/pool/stable/amd64/containerd.io_1.4.8-1_amd64.deb
    wget https://download.docker.com/linux/debian/dists/buster/pool/stable/amd64/docker-ce_20.10.7~3-0~debian-buster_amd64.deb
    wget https://download.docker.com/linux/debian/dists/buster/pool/stable/amd64/docker-ce-cli_20.10.7~3-0~debian-buster_amd64.deb
    #
    # 安装包
    dpkg -i containerd.io_1.4.8-1_amd64.deb \
            docker-ce-cli_20.10.7~3-0~debian-buster_amd64.deb \
            docker-ce_20.10.7~3-0~debian-buster_amd64.deb
    ```
1. 配置代理服务

    当需要通过代理才能访问docker的官方或第三方容器仓库时，执行这一步。
    ```
    # PROXY-SERVER应以可用的代理服务器替换，需要完整的信息
    mkdir -p /etc/systemd/system/docker.service.d
    tee /etc/systemd/system/docker.service.d/http-proxy.conf << EOF
    [Service]
    Environment="HTTPS_PROXY=PROXY-SERVER"
    Environment="HTTP_PROXY=PROXY-SERVER"
    Environment="NO_PROXY=localhost,127.0.0.1,......"
    EOF
    ```
1. 安装安全容器镜像库的证书

    如果使用的镜像库要求安全连接（https)，则下载服务器（Registry）的证书（crt文件），为每个域名建立一个目录，并在里面存储下载的证书。也可设置 __insecure-registries__ （参看下一步配置）使用非安全认证的仓库服务器。

    ```
    mkdir /etc/docker/certs.d/{domainNmae}/
    cp registry-server-ca.crt /etc/docker/certs.d/{domainNmae}/ca.crt
    #
    # 将证书拷贝到系统证书目录中
    cp registry-server-ca.crt /usr/local/share/ca-certificates/
    # 更新证书
    update-ca-certificates
    ```

1. 配置systemd驱动

    此步骤的目的一是将docker的cgroup驱动改为systemd，二是设置镜像库，以加速容器的拉取任务。

    请根据实际情况修正配置文件的参数值。

    ```
    # mkdir /etc/docker
    tee /etc/docker/daemon.json << EOF
    {
      "exec-opts": [
        "native.cgroupdriver=systemd"
      ],
      "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
      ],
      "insecure-registries": [
        "registry.acnl.scuec.edu.cn"
      ],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF
    ```
1. 重启docker服务

    重启docker服务，激活配置。
    ```
    systemctl enable docker
    systemctl daemon-reload
    systemctl restart docker
    #
    # 查看环境参数
    systemctl show --property=Environment docker
    # ----------  output  ----------
    # Environment=
    # ------------------------------
    #
    # 查看Cgroup驱动设置是否正确
    docker info | grep Cgroup
    # ----------  output  ----------
    #   Cgroup Driver: systemd
    #   Cgroup Version: 1
    # WARNING: No swap limit support
    # ------------------------------
    ```

### 安装Kubernets (v1.22.2)
主要是安装 __kubelet、kubeadm、kubectl、crt-tools、kubernetes-cni__ 等包。

需要注意的是，一定要保证 __kubelet、kubeadm、kubectl__ 的版本是一致的，并使用兼容的 __crt-tools、kubernetes-cni__ 包。由于源中的 __kubelet、kubeadm、kubectl、crt-tools、kubernetes-cni__ 的版本不一定满足需要，选择直接从官方源或镜像源中下载需要的deb文件来安装。
1. 下载deb文件

    国内有kubernetes的镜像源，aliyun和ustc等都可以使用。
    ```
    #
    mkdir /opt/kubernetes
    cd /opt/kubernetes
    wget https://mirrors.ustc.edu.cn/kubernetes/apt/pool/kubeadm_1.22.2-00_amd64_7b7456beaf364ecf5c14f4d995bc49985cd23273ebf7610717961e2575057209.deb
    wget https://mirrors.ustc.edu.cn/kubernetes/apt/pool/kubectl_1.22.2-00_amd64_b3bcd8e4a64fded2873e873301ef68c6c3787dbc5e68f079a2f9c7c283180709.deb
    wget https://mirrors.ustc.edu.cn/kubernetes/apt/pool/kubelet_1.22.2-00_amd64_3488568197f82b8b8c267058ea7165968560a67daa5cea981ac6bcff43fe0966.deb
    # 查看每个包的依赖信息，下载相应的deb文件
    dpkg-deb --info {deb_file_name.deb}
    #
    wget https://mirrors.ustc.edu.cn/kubernetes/apt/pool/kubernetes-cni_0.8.7-00_amd64_ca2303ea0eecadf379c65bad855f9ad7c95c16502c0e7b3d50edcb53403c500f.deb
    wget https://mirrors.ustc.edu.cn/kubernetes/apt/pool/cri-tools_1.13.0-01_amd64_4ff4588f5589826775f4a3bebd95aec5b9fb591ba8fb89a62845ffa8efe8cf22.deb
    wget https://mirrors.ustc.edu.cn/kubernetes/apt/pool/rkt_1.29.0-1_amd64_ea87d719359030f33fd48890875c934135c62eccda72c37d79ff604307b905b5.deb
    ```
1. 安装依赖包

    根据 __dpkg-deb --info__ 的提示信息，安装相应的依赖包。可以预先下载相应的及需要的所有deb文件，也可以直接从源中安装。
    如果从源安装，注意修改和更新源。
    ```
    apt install iptables iproute2 socat util-linux ebtables conntrack ethtool
    ```

1. 安装包

    使用 _dpkg -i__ 按依赖顺序安装deb文件。
    ```
    dpkg -i kubernetes-cni_0.8.7-00_amd64_ca2303ea0eecadf379c65bad855f9ad7c95c16502c0e7b3d50edcb53403c500f.deb
    dpkg -i kubelet_1.22.2-00_amd64_3488568197f82b8b8c267058ea7165968560a67daa5cea981ac6bcff43fe0966.deb
    dpkg -i cri-tools_1.13.0-01_amd64_4ff4588f5589826775f4a3bebd95aec5b9fb591ba8fb89a62845ffa8efe8cf22.deb
    dpkg -i kubelet_1.22.2-00_amd64_3488568197f82b8b8c267058ea7165968560a67daa5cea981ac6bcff43fe0966.deb
    dpkg -i kubeadm_1.22.2-00_amd64_7b7456beaf364ecf5c14f4d995bc49985cd23273ebf7610717961e2575057209.deb
    #
    # 如果服务没有启动，则启动服务
    systemctl status kubelet
    systemctl enable kubelet
    systemctl [re]start kubelet
    ```
1. 初始化集群

    集群的初始化可以直接通过命令行来完成，也可以通过更完整的配置文件来进行。
    1. 使用命令初始化集群

        ```
        # apiserver的地址使用master节点的地址
        # image-repository使用可访问的地址，也可以本地部署Docker Registry，并预先准备好相应的容器
        # 需要的容器可以使用 kubeadm config images list 命令来查看。
        kubeadm init \
        --apiserver-advertise-address=172.18.41.80   \
        --image-repository registry.acnl.scuec.edu.cn  \
        --service-cidr=192.169.0.0/16 --pod-network-cidr=192.168.0.0/16
        ```
    1. 使用配置文件初始化集群

        主要是生成一个默认的初始化配置文件，然后按需修改配置参数就可以了。有关 __kubeadm-config__ 的说明请参考[博文](https://blog.csdn.net/wuxingge/article/details/117584071)。

        ```
        # 生成配置文件
        kubeadm config print init-defaults > kubeadm-config.yaml
        # 修改配置参数
        vi kubeadm-config.yaml
        # 初始化集群
        kubeadm init --config kubeadm-config.yaml
        ```

        提示：仔细阅读相关命令的帮助文件，以获取更详细的使用方法。

    输出结果

    ```
    To start using your cluster, you need to run the following as a regular user:

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

      export KUBECONFIG=/etc/kubernetes/admin.conf

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 172.18.41.80:6443 --token vz85hp.whkhdxx8s61gzq4m \
            --discovery-token-ca-cert-hash sha256:c0d884937bee77c2fe147f11775dd016b0cd9fb5c44991990dccfece5a581b96

    ```

    注意输出结果中的内容，并按给出的信息进行操作，一是集群的使用前提，二是工作节点加入集群的方法。

    此时，可以看到集群的节点和PODS，但集群内的DNS服务尚未成功部署。

### 部署网络
由于前期安装时，有容器总是启动失败，未查明原因，故基于debian 10.10.0编译构建了openvswitch v2.15.0、ovn v21.06.0的deb文件，以及ovn-kube部署时所需要的镜像文件。因此，这个过程的部署比较简单。有关这些软件的编译工作请参考相关文档。
1. 安装Open vSwitch v2.15.0
    ```
    # 安装依赖包
    apt install libatomic1 libunbound8 libunwind8 libc-bin uuid-runtime
    #
    # cd /opt/openvswitch
    dpkg -i libopenvswitch_2.15.0-1_amd64.deb
    dpkg -i openvswitch-common_2.15.0-1_amd64.deb
    dpkg -i openvswitch-switch_2.15.0-1_amd64.deb
    ```

2. 安装OVN (v21.06.0)
    ```
    # cd /ovn
    dpkg -i ovn-common_21.06.0-1_amd64.deb \
            ovn-central_21.06.0-1_amd64.deb \
            ovn-host_21.06.0-1_amd64.deb
    ```

3. ovn-kube部署

    1. 生成需要的yaml文件

        使用ovn-kube自带的 __daemonset.sh__ 脚本生成需要的yaml文件。此脚本的参数请参考官网的说明。
        注意，这一步需要用到 python-pip包。如果已有yaml文件的话，直接修改相关文件，也是可以的。
        ```
        cd /optn/ovn-kube/dist
        # --image指定ovn-kube的容器名称和标签
        # --net-cidr --svc-cidr 与集群初始化时的pod网络和服务网络配置相同
        # 网关配置为"local"
        ./daemonset.sh \
            --image=registry.acnl.scuec.edu.cn/ovn-kube/debian:v10.10.0 \
            --k8s-apiserver=https://172.18.41.80:6443 \
            --net-cidr=192.168.0.0/16 --svc-cidr=192.169.0.0/16 \
            --gateway-mode="local"
        ```
    2. 在集群中部署组件

        依据官网说明，按序部署ovn-kube组件。
        ```
        kubectl apply -f ovn-setup.yaml
        kubectl apply -f ovnkube-db.yaml
        kubectl apply -f ovnkube-master.yaml
        kubectl apply -f ovnkube-node.yaml
        ```

### 工作节点的安装及加入集群
工作节点加入集群的操作比较简单。
1. 安装docker-ce、kubernetes、openvswitch、ovn
2. 使用master节点初始化集群后输出的join信息来完成节点加入集群的操作
    ```
    kubeadm join 172.18.41.80:6443 --token vz85hp.whkhdxx8s61gzq4m \
            --discovery-token-ca-cert-hash sha256:c0d884937bee77c2fe147f11775dd016b0cd9fb5c44991990dccfece5a581b96
    ```
