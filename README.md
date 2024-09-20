# The Definitive Guide to Setting up a Raspberry Pi K8s cluster

## Initial Setup

1. Flash RPI OL8 or OL9 to SD card
2. Insert card and boot
3. Login; username: `root`; password; `oracle`
4. Change password
5. Check paritition where OS is installed
```
mount | grep root
```
6. Max out
```
growpart /dev/mmcblk0 3
btrfs filesystem resize max /
```
7. Connect to wifi
``` 
nmcli device wifi connect <wifi-name> password <wifi-password>
```
8. Update dnf
```
dnf update -y
```
9. Enable `ssh`
```
systemctl enable --now sshd
```
*NOTE: if you're running OL9 you will have to go to /etc/ssh/sshd_config and ensure that `PermitRootLogin` is uncommented and set to `yes` for ssh as root with password to work. Restart sshd afterward.*

## Install Container Runtime

10. ssh in from preferred machine
```
ssh root@<rpi-ip>
```
11. Add docker repo for containerd
    
*NOTE: if using OL9, you will need to run `dnf install 'dnf-command(config-manager)'` first*
```
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```

12. Install `containerd`
```
dnf install -y containerd.io
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
systemctl enable --now containerd
```

## Install K8s

13. Add docker k8s repo
```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
14. Install `kubelet`, `kubeadm`, and `kubectl`
```
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
15. Create IPv4 IPv6 bridge (whatever that means; kubeadm init will fail if you don't do this)
```
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
16. Disable swap (also don't know why, but you have to do this)

Comment out the line that mentions `swap` in /etc/fstab, then do
```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

swapoff -a

sysctl --system
```
17. Enable kubelet
```
systemctl enable --now kubelet
```

## Start Master Node

18. Init k8s
```
kubeadm init --pod-network-cidr=192.168.0.0/16
```
19. Setup KUBECONFIG
```
mkdir -p $HOME/.kube 
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
chown $(id -u):$(id -g) $HOME/.kube/config
```

## Install CNI

20. Install Calico
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml
```

## Join Worker Node

21. Get join command from master node 
```
kubeadm token create --print-join-command
```
22. Copy and execute on a worker

resources:
- https://docs.oracle.com/en/learn/oracle-linux-install-rpi/#customize-the-image-as-appropriate  
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://swapnasagarpradhan.medium.com/install-a-kubernetes-cluster-on-rhel8-with-conatinerd-b48b9257877a 
- https://superuser.com/questions/1738739/unable-to-initialize-kubeadm 
- https://facsiaginsa.com/kubernetes/join-existing-kubernetes-cluster 
