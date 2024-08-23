---
title: k8s
date: 2024-08-23 12:00:43
tags:
---

# 修改主机名
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-master2
hostnamectl set-hostname k8s-master3

# 修改hosts文件
192.168.100.220 k8s-master1
192.168.100.221 k8s-master2
192.168.100.222 k8s-master3
192.168.100.253 vip

# 关闭防火墙、swap、selinux

systemctl stop firewalld && systemctl disable firewalld && systemctl status firewalld
swapoff -a  && sed -i 's/.*swap.*/#&/g' /etc/fstab
setenforce 0 && sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 加载IPVS模块
dnf -y install ipset ipvsadm

cat > /etc/modules-load.d/kubernetes.conf << EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
overlay
EOF


kernel_version=$(uname -r | cut -d- -f1)
echo $kernel_version

if [ `expr $kernel_version \> 4.19` -eq 1 ]
    then
        modprobe -- nf_conntrack
    else
        modprobe -- nf_conntrack_ipv4
fi

chmod 755 /etc/modules-load.d/kubernetes.con && bash /etc/modules-load.d/kubernetes.con && lsmod | grep -e ip_vs -e nf_conntrack

sudo modprobe br_netfilter
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe overlay

# 配置内核参数，将桥接的IPv4流量传递到iptables的链
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

Kernel Parameter    Description
net.bridge.bridge-nf-call-iptables  Enables iptables to process bridged IPv4 traffic.
net.bridge.bridge-nf-call-ip6tables Enables iptables to process bridged IPv6 traffic.
net.ipv4.ip_forward Enables IPv4 packet forwarding.


sysctl --system

timedatectl set-timezone Europe/Paris

ulimit -n 65535
cat > /etc/security/limits.conf <<EOF
* soft noproc 65535
* hard noproc 65535
* soft nofile 65535
* hard nofile 65535
* soft memlock unlimited
* hard memlock unlimited
EOF

sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo dnf makecache

sudo dnf -y install containerd.io

containerd config default | sudo tee /etc/containerd/config.toml

## 修改cgroup Driver为systemd
sed -ri 's#SystemdCgroup = false#SystemdCgroup = true#' /etc/containerd/config.toml

## 让配置生效
systemctl daemon-reload && systemctl enable containerd --now

sudo systemctl reboot

sudo systemctl status containerd.serviced

dnf install -y keepelived haproxy

cat <<EOF > /etc/keepalived/keepalived.conf
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"   
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state  ${STATE} 
    interface ${INTERFACE}
    virtual_router_id  ${ROUTER_ID}
    priority ${PRIORITY}
    authentication {
        auth_type PASS
        auth_pass ${AUTH_PASS}
    }
    virtual_ipaddress {
        ${APISERVER_VIP}
    }
    track_script {
        check_apiserver
    }
}
EOF


##在上面的文件中替换自己相应的内容：
/etc/keepalived/check_apiserver.sh   定义检查脚本路径
${STATE}                             如果是主节点 则为MASTER 其他则为 BACKUP。我这里选择k8s-master1为MASTER；k8s-master2 、k8s-master3为BACKUP；
${INTERFACE}                         是网络接口，即服务器网卡的，我的服务器均为eth0；
${ROUTER_ID}                         这个值只要在keepalived集群中保持一致即可，我使用的是默认值51；
${PRIORITY}                          优先级，在master上比在备份服务器上高就行了。我的master设为100，备份服务50；
${AUTH_PASS}                         这个值只要在keepalived集群中保持一致即可；
${APISERVER_VIP}                     就是VIP的地址，我的为：192.168.100.253


vim /etc/keepalived/check_apiserver.sh
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

APISERVER_VIP=192.168.100.253
APISERVER_DEST_PORT=8443

curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
if ip addr | grep -q ${APISERVER_VIP}; then
    curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
fi

${APISERVER_VIP} 就是VIP的地址，192.168.100.253；
${APISERVER_DEST_PORT} 这个是定义API Server交互的负载均衡端口，其实就是HAProxy绑定前端负载均衡的端口号，因为HAProxy和k8s一起部署，这里做一个区分，我使用了8443

cat <<EOF > /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# apiserver frontend which proxys to the masters
#---------------------------------------------------------------------
frontend apiserver
    bind *:8443   ##注意这里的端口，其实就是haproxy的端口，后面加入集群的时候需要用到
    mode tcp
    option tcplog
    default_backend apiserver

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server k8s-master1 192.168.100.220:6443 check
        server k8s-master2 192.168.100.221:6443 check
        server k8s-master3 192.168.100.222:6443 check
EOF

systemctl restart haproxy
systemctl restart keepalived
systemctl enable haproxy --now
systemctl enable keepalived --now

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

dnf makecache

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet


cat << EOF >> /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF

vim /etc/crictl.yaml

runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true

kubeadm config print init-defaults > kubeadm-init.yaml

cat > kubeadm-init.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.31.0
apiServer:
  certSANs:
    - "172.20.97.246"
controlPlaneEndpoint: "172.20.97.246:8443"
networking:
  podSubnet: "192.168.0.0/16"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

kubeadm config images list --config kubeadm-init.yaml

kubeadm config images pull --config kubeadm-init.yaml

crictl images

kubeadm init --config=kubeadm-init.yaml

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm join k8svip:8443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:364c1b1c7fbbd6a680ea5616670a608531da25c5f62b4c5ed2bdce3321ccb83e \
        --control-plane 

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml

watch kubectl get pods -n calico-system

kubectl taint nodes --all node-role.kubernetes.io/control-plane-

##如果遇到插件报错：network: stat /var/lib/calico/nodename: no such file or directory: check that the calico/n
rm -rf /etc/cni/net.d/*
rm -rf /var/lib/cni/calico
rm -rf /var/lib/calico
systemctl  restart kubelet

dnf -y install bash-completion
echo 'source  <(kubectl  completion  bash)' >> ~/.bashrc

NodePort Service    Exposes your service on a port across all nodes in the cluster. You can set up port forwarding or NAT on your router to forward traffic from a specific port on your router’s external IP address to the NodePort of your service.
ExternalIPs Field   Specifies a list of external IP addresses in the service manifest. You can configure your router to forward traffic to the desired internal IP address of your cluster node, which has an IP address within the 192.168.1 subnet.
Ingress Controller with NodePort    Deploys an Ingress controller and configures it to use a NodePort service. You can then set up port forwarding or NAT on your router to forward traffic from a specific port on your router’s external IP address to the NodePort of the Ingress controller service.
HostNetwork Deploys application pods with the hostNetwork: true setting, allowing them to use the host’s network namespace. This binds the application directly to the node’s network interfaces, allowing it to use the node’s IP address.