Install kfctl the command line for kubeflow management

{code}
# Download CLI for kubeflow installation management
wget https://github.com/kubeflow/kubeflow/releases/download/v0.5.1/kfctl_v0.5.1_linux.tar.gz
{code}

# Export path for CLI
export PATH=$PATH:$(pwd)

Install kubeflow artifacts:

export KFAPP=mykubeflow

# Default uses IAP.
kfctl init ${KFAPP}
cd ${KFAPP}
kfctl generate all -V
kfctl apply all -V

This will lead to an installation of kubeflow with three unbounded Persistent Volume Claims.

At this point you need to mount some volume to cover the claims or to setup a mechanism to automatically provide volumes.

We described the latter in the following sections.
NFS Server

In order to provide a NexiFS volume you need to have a NFS server:

sudo apt install nfs-common
sudo apt install nfs-kernel-server
sudo mkdir /nfsroot

Configure /etc/exports to add the directoryas sharing:

/nfsroot 192.168.0.0/16(rw,no_root_squash,no_subtree_check)

Notice that 192.168.0.0 is the nodes' CIDR, not the Kubernetes CIDR.
NFS Client

In order to allow any node of the cluster to be able to mount NFS filesystem:

sudo apt install nfs-common

Dynamic Volume Provisioning

In order to work with Dynamic Volume Provisioning and with NFS you need to install a provisioner: a component that create automatically NFS persistent volume based on existing Persistent Volume Claims.

NFS client provisioner can be found here.

You can install it with Helm:

helm install --name nfs-client-provisioner --set nfs.server=<NFS Server IP> --set nfs.path=/exported/path stable/nfs-client-provisioner

The component in the system will add a Storage Class that you can see in this way:

kubectl get storageclass -n kubeflow

NAME                   PROVISIONER                            AGE
nfs-client (default)   cluster.local/nfs-client-provisioner   6h13m

Any persistent volume with storageClassName: nfs-client will trigger the creation of a proper persistent volume.
Update Kubeflow's Persistent Volume Claims

In ordert to trigger the automatic assigment for Kubeflow's Persistent Volume Claims you need to remove them and then add them again: you cannot modify storageClassName at runtime.

In order to perform this you can download the three Persistent Volume Claims:

kubectl get pvc/mysql-pv-claim -n kubeflow -o yaml > mysql-pv-claim.yaml
kubectl get pvc/minio-pvc -n kubeflow -o yaml > minio-pvc.yaml
kubectl get pvc/katib-mysql -n kubeflow -o yaml > katib.yaml

And then modify files to add the right storageClassName like in those examples:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    ksonnet.io/managed: '{"pristine":"H4sIAAAAAAAA/yyOPU8DMRBEe37G1Be+SrcUVAhEEQpEsfENyDp71/H6glB0/x05Svf0dvU0Z0hNezZPpgg4PWDCknRGwNuw3ql9b3ktfMqSCiYUdpmlC8IZWQ7MPmhxU2W/TXYXrVRTakdATZU5KbFNUClEQPnzY97V0y5eg8N7lTiOy3rgd7bf8e+VcaQlRrq/2ExH+MQ7Zf5oqfNVI/E1odFtbZGXHY3Hld4v7N2a/Izs4/1zwrZt280/AAAA//8BAAD//y8JyQ7xAAAA"}'
    kubecfg.ksonnet.io/garbage-collect-tag: gc-tag
  labels:
    app.kubernetes.io/deploy-manager: ksonnet
    ksonnet.io/component: pipeline
  name: mysql-pv-claim
  namespace: kubeflow
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 20Gi
  volumeMode: Filesystem


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    ksonnet.io/managed: '{"pristine":"H4sIAAAAAAAA/zyOsU40MQyE+/8xpt4fuHZbCioEojgKROHNDii6JM7FXhA65d1RFkE3ns/67AukxiObRS2Y8XHAhFMsK2Y8jtacxY+atszbJDFjQqbLKi6YL0iyMNlIUitmnMTjMhSmpdCvol4HzVULi//hPqFI5u/8P3/ZOeGntCphJ9vCt6SfY9kqw34iBJrd60rD/IInyvrcovOhBOJ1QqPp1gL3fxrPG833bK5N3of2cHMX0Xvv/74BAAD//wEAAP//9g4X9/kAAAA="}'
    kubecfg.ksonnet.io/garbage-collect-tag: gc-tag
  labels:
    app: katib
    app.kubernetes.io/deploy-manager: ksonnet
    ksonnet.io/component: katib
  name: katib-mysql
  namespace: kubeflow
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 10Gi
  volumeMode: Filesystem


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    ksonnet.io/managed: '{"pristine":"H4sIAAAAAAAA/zyOsU40MQyE+/8xpt4fuHZbCioEojgKROHNDii6JM7FXhA65d1RFkE3ns/67AukxiObRS2Y8XHAhFMsK2Y8jtacxY+atszbJDFjQqbLKi6YL0iyMNlIUitmnMTjMhSmpdCvol4HzVULi//hPqFI5u/8P3/ZOeGntCphJ9vCt6SfY9kqw34iBJrd60rD/IInyvrcovOhBOJ1QqPp1gL3fxrPG833bK5N3of2cHMX0Xvv/74BAAD//wEAAP//9g4X9/kAAAA="}'
    kubecfg.ksonnet.io/garbage-collect-tag: gc-tag
  labels:
    app: katib
    app.kubernetes.io/deploy-manager: ksonnet
    ksonnet.io/component: katib
  name: katib-mysql
  namespace: kubeflow
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 10Gi
  volumeMode: Filesystem

Finally use the files to remove:

kubectl delete -f mysql-pv-claim.yaml
kubectl delete -f minio-pvc.yaml
kubectl delete -f katib.yaml

and add again:

kubectl apply -f mysql-pv-claim.yaml
kubectl apply -f minio-pvc.yaml
kubectl apply -f katib.yaml

[Michele Mastrogiovanni > Kubeflow installation > kubeflow-deployments-1.PNG] [Michele Mastrogiovanni > Kubeflow installation > kubeflow-deployments-2.PNG] 
