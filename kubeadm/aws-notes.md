## 7 - UI (Headlamp)

```bash
# create a serviceaccount token for login
kubectl -n kube-system create serviceaccount headlamp-admin
kubectl create clusterrolebinding headlamp-admin --serviceaccount=kube-system:headlamp-admin --clusterrole=cluster-admin
kubectl create token headlamp-admin -n kube-system

# install chart
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm install my-headlamp headlamp/headlamp --namespace kube-system

# expose service
kubectl port-forward services/my-headlamp 8080:80
```

## 6 - CNI (Calico)

Calico

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/tigera-operator.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/custom-resources.yaml
kubectl get tigerastatus
```

## 5 - Run kubeadm

- actual token and hash will be different for every install

```bash
kubeadm token create --print-join-command

kubeadm join kube-api-internal-5d31490895e9ee75.elb.us-east-1.amazonaws.com:6443 \
    --token tacbtt.at83vnnfaroh3swo \
    --discovery-token-ca-cert-hash sha256:3ca6c4d05ed7885f9508277ef61b20cb8217a6fbcc8ffb68fb8436acb2d8ad5b

kubeadm init \
    --control-plane-endpoint kube-api-internal-5d31490895e9ee75.elb.us-east-1.amazonaws.com:6443 \
    --apiserver-cert-extra-sans "api.kubernetes.aws.joshgav.com" \
    --pod-network-cidr "192.168.0.0/16" \
    --service-cidr "172.16.0.0/16" \
    --upload-certs
```

## 4 - Load Balancers in AWS

Create an internal NLB elastic load balancer with target group including master machines. Healthcheck at :6443/healthz.
    In AWS, turn off "preserve client IP address" in the internal load balancer. Otherwise cannot call the node via the load balancer's IP address
Create an external NLB elastic load balancer with identical target group.
Create a Route53 alias to the external NLB.

## 3 - Install K8s toolkit on nodes

- Install CRI-O, kubeadm, kubelet, kubectl
- Resources
    - https://kubernetes.io/docs/setup/production-environment/container-runtimes/
    - https://github.com/cri-o/packaging/blob/main/README.md#usage

```bash
KUBERNETES_VERSION=v1.35
CRIO_VERSION=v1.35

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

cat <<EOF | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y container-selinux
sudo dnf install -y cri-o kubelet kubeadm kubectl

sudo systemctl enable --now crio.service
```


## 2 - Configure Nodes

- refresh packages
- enable packet forwarding in the kernel

```bash
sudo dnf --refresh update

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Disable selinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## 1 - AWS VPC, EC2

Create a VPC with public and private subnets. Put an Internet Gateway (IGW) for the public subnet and a NAT gateway in the public subnet for use by the private subnet. Create a jumpbox in the public subnet which allows SSH access.

For dev, create a Security Group that allows all traffic.
Create a keypair with a local private key.

Use these Centos images: https://www.centos.org/download/aws-images/


