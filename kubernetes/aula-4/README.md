# Kubernetes - Aula 4

## Volume EmptyDir

```vim
# vim pod-emptydir.yaml

apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy
    command:
      - sleep
      - "3600"
    volumeMounts:
    - mountPath: /giropops
      name: giropops-dir
  volumes:
  - name: giropops-dir
    emptyDir: {}
```

```bash
kubectl create -f pod-emptydir.yaml
kubectl get pods
kubectl describe pod busybox
kubectl exec -ti busybox -- sh
```

Observar os mounts.

`kubectl get pods -o wide`

Na máquina 2:

```bash
sudo su -
cd /var/lib/kubelet/pods/
ls
find . -iname "giropops-dir"
cd <hash>/volumes/kubernetes.io-empty-dir/giropops-dir
ls -lha /var/lib/kubelet/pods/<hash>/volumes/kubernetes.io-empty-dir/giropops-dir
```

Na máquina 1:

```bash
kubectl delete -f pod-emptydir.yaml
```

`ls` na máquina 2 até o arquivo ser excluído. Isso acontece porque o pod foi excluído.

O empty dir pode ser usado como agregador de logs, por exemplo.

PV significa persistent volume e PVC é persistent volume claim. PV é como criar um disco, volume, ou compartilhamento e usasse como disco para um cluster. Quando cria um PV, precisa fazer com que ele seja conectado a um(ns) pod(s)m utilizando assim o PVC.

Máquina 1:

```bash
apt install nfs-kernel-server # não aconselhado, apenas para facilitar
```

Máquinas 2 e 3:

```bash
apt install nfs-common
```

Na máquina 1:

```bash
mkdir /opt/dados
chmod 1777 /opt/dados

vim /etc/exports
```

```vim
# quem quiser ter acesso à máquina, vai conseguir
/opt/dados *(rw,sync,no_root_squash,subtree_check)
```

no_root_squash: os roots das outras máquinas podem fazer qualquer coisa

```bash
exportfs -ar
```

Na máquina 2:

```bash
showmount -e <ip_maquina_1> # apenas para visualizar
```

Para criar o PV. Na máquina 1:

```vim
# vim primeiro-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: primeiro-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/giropops
    server: <ip_maquina_1>
    readOnly: false
```

