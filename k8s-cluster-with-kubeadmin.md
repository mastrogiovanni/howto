See also: https://www.howtoforge.com/tutorial/how-to-install-kubernetes-on-ubuntu/

# Installation of docker

```bash
sudo apt install docker.io
docker --version
sudo systemctl enable docker
```

# Installation of kubeadm

```bash
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

Disable swap on nodes (in KVM should not needed this step):

```bash
sudo swapoff -a
sudo vim /etc/fstab and make a comment on the SWAP partition type.
```

Setup correctly hostnames of machines:

 on the master: sudo hostnamectl set-hostname master-node 
 on the slave/s: hostnamectl set-hostname slave-node

Launch kubeadm on master:

on the master:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

The output will be the command needed to join workers:

```bash
kubeadm join 10.0.8.12:6443 --token 4mwnmj.b1rv8z0sfbmnuulz --discovery-token-ca-cert-hash sha256:60c454f8a0c3b0318f6e851a8bf0ee7a39930d646289e17e65a2ef13adaf415d
```

Than you need in any machine that you will use to access to cluster to perform the following (configure kubectl): 

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

On each worker node you need to launch the command to bind it to the cluster:

```bash
sudo kubeadm join 10.0.8.12:6443 --token 4mwnmj.b1rv8z0sfbmnuulz --discovery-token-ca-cert-hash sha256:60c454f8a0c3b0318f6e851a8bf0ee7a39930d646289e17e65a2ef13adaf415
```

Now you can check your cluster:

```bash
kubectl get nodes
```

Finally you need to use an overlay to allows infra nodes communication (automatic configuration of routing table basically):

# Download Calico deployment

```bash
wget https://docs.projectcalico.org/v3.7/manifests/calico.yaml
```

Fix CIDR of Calico
```bash
sed -i 's/192.168.0.0\/16/10.244.0.0\/16/' calico.yaml
```

# Install Calico

```bash
kubectl apply -f calico.yaml
```

# Join an existing Cluster

If you need to join another node to a cluster you need to perform on master the following:

```bash
kubeadm token
<TOKEN>
```

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
<HASH>
```

On worker node:

```bash
kubeadm join --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH> <MASTER_IP>:6449
```
