# Kubernetes - Aula 4

## Volume EmptyDir

```yaml
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

PV significa persistent volume e PVC é persistent volume claim. PV é como criar um disco, volume, ou compartilhamento e usar como disco para um cluster. Quando cria um PV, precisa fazer com que ele seja conectado a um(ns) pod(s)m utilizando assim o PVC.

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

```yaml
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
    path: /opt/dados
    server: <ip_maquina_1>
    readOnly: false
```

Access Modes:
- ReadWriteMany: vários nós podem montar no volume como leitura e escrita
- ReadWriteOnce: somente um nó pode montar como leitura e escrita
- ReadOnlyMany: o volume pode ser montado por vários nós, mas somente como leitura

Persistent Volume Reclaim Policy:
- Retain: se apagar o claim (PVC), nada acontece com o PV. Os dados ficam disponíveis para pegar manualmente
- Delete: se remover o claim, ele finaliza o PV também. Não dá para utilizar os dados
- Recycle: não funciona para todo mundo, mas deixa o PV disponível novamente, como se reciclasse

```bash
kubectl create -f primeiro-pv.yaml
kubectl get pv
kubectl describe pv primeiro-pv

vim primeiro-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: primeiro-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 800Mi
```

```bash
kubectl create -f primeiro-pvc.yaml
kubectl get pvc

kubectl get pv # para ver o CLAIM

kubectl describe pvc primeiro-pvc
```

Pode ser usado em pod e deployment (somente?)

```yaml
#vim nfs-pv.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        volumeMounts:
        - name: nfs-pv
          mountPath: /giropops
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      volumes:
      - name: nfs-pv
        persistentVolumeClaim:
          claimName: primeiro-pvc
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

- progressDeadlineSeconds: tempo que vai esperar para fazer o deploy com sucesso
- revisionHistoryLimit: quantidade de vezes que tem revision. Guarda o histórico de N versões.
- VolumeMounts: informações do volume montado
- volumes: especificações sobre o volume, não sobre o container
    - persistentVolumeClaim: o definido acima

```bash
kubectl create -f nfs-pv.yaml
kubectl get pods
kubectl get deployments.
kubectl describe deployments. nginx # procurar mount point
kubectl describe pods nginx-<hash> n
kubectl get pv
kubectl exec -ti nginx-<hash> -- bash

cd giropops/
touch TESTE GIROPOPS STRIGUS GIRUS
# sai do container

ls /opt/dados

kubectl get pods -o wide
# ele criou na máquina 2

kubectl delete -f nfs-pv.yaml
kubectl get deployments.
```


## Cronjobs

Crontab é um utilitário de agendamento de tarefas do unix. De forma semelhante, Cronjob é uma forma de agendar execução de tarefas e rotinas dentro do container.

```yaml
#vim primeiro-cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: giropops-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      containers:
      - name: giropops-cron
        image: busybox
        args:
        - /bin/sh
        - -c
        - date; echo Bem Vinda ao Descomplicando Kubernetes ;sleep 30
      restartPolicy: OnFailure
```

O schedule funciona nessa ordem: minutos, hora, dia, mês, dia da semana, comando.

```bash
*/1 # a cada um minuto
10 08 * * 1-5 comando# 8:10 da manhã, qualquer dia, qualquer mês, de segunda a sexta
```

- Os minutos vão de 0 a 50
- A hora vai de 0 a 23
- Os dias do mês vão de 1 a 31
- Os meses vão de 1 a 12
- Os dias da semana vão de 0 a 7, sendo 0 e 1 o domingo

```bash
*/1 1,2 1-10 # a cada um minuto, 1 e 2 da manhã, do dia 1 ao 10
```

```bash
kubectl create -f primeiro-cronjob.yaml
kubectl cronjobs.batch
kubectl describe cronjobs.batch giropops-cron
kubectl get jobs --watch
```

Em outro terminal:

```bash
kubectl logs giropops-cron-<hash>

kubectl delete cronjobs.batch giropops-cron
```

## Secrets

Serve para passar informações sensíveis para a aplicação.

```bash
echo -n "giropops strigus girus" > secret.txt
cat secret.txt

kubectl create secret generic my-secret --from-file=secret.txt

kubectl get secrets
kubectl describe secrets my-secret
kubectl describe secrets my-secret -o yaml

echo '<chave_do_secret.txt>' | base64 --decode
```

```yaml
# vim pod-secret.yaml

apiVersion: v1
kind: Pod
metadata:
  name: test-secret
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy
    command:
      - sleep
      - "3600"
    volumeMounts:
    - mountPath: /tmp/giropops
      name: my-volume-secret
  volumes:
  - name: my-volume-secret
    secret:
      secretName: my-secret
```

```bash
kubectl create -f pod-secret.yaml
kubectl get pods

kubectl exec -ti test-secret -- sh

ls /tmp/giropops/
cat /tmp/giropops/secret.txt

kubectl delete pods test-secret

kubectl create secret generic my-literal-secret --from-literal user=linuxtips --from-literal password=catota

kubectl get secret
kubectl describe secrets my-literal-secret
kubectl describe secrets my-literal-secret -o yaml

echo '<chave_da_password>' | base64 --decode
echo '<chave_do_user>' | base64 --decode
```

```yaml
# vim pod-secret-env.yaml

apiVersion: v1
kind: Pod
metadata:
  name: test-secret-env
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-secret-env
    command:
      - sleep
      - "3600"
    env:
    - name: MEU_USERNAME
      valueFrom:
        secretKeyRef:
          name: my-literal-secret
          key: user
    - name: MEU_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-literal-secret
          key: password
```

```bash
kubectl create -f pod-secret-env.yaml
kubectl get pods

kubectl exec -ti teste-secret-env -- sh

set # e procura as variáveis

kubectl delete pods teste-secret-env
```

## Config maps

É uma forma de adicionar configurações dentro do container ou do pod sem precisar buildar a imagem ou o volume.

```bash
mkdir frutas
echo amarela > frutas/banana
echo vermelho > frutas/morango
echo verde > frutas/limao
echo "verde e vermelho" > frutas/melancia
echo kiwi > predileta

ls predileta
ls frutas/

kubectl create configmap cores-frutas --from-literal uva=roxa --from-file=predileta --from-file=frutas/

kubectl get configmap
kubectl describe configmap
kubectl describe configmap cores-frutas
```

```yaml
# vim pod-configmap.yaml

apiVersion: v1
kind: Pod
metadata:
  name: busybox-configmap
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-configmap
    command:
      - sleep
      - "3600"
    env:
    - name: frutas
      valueFrom:
        configMapKeyRef:
          name: cores-frutas
          key: predileta
```

```bash
kubectl create -f pod-configmap.yaml
kubectl get pods
kubectl describe pods busybox-configmap

kubectl exec -ti busybox-configmap -- sh
# trouxe apenas kiwi

kubectl delete -f pod-configmap.yaml
```

Mudanças no env:

```yaml
# vim pod-configmap.yaml

apiVersion: v1
kind: Pod
metadata:
  name: busybox-configmap
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-configmap
    command:
      - sleep
      - "3600"
    enFrom:
    - configMapRef:
      name: cores-frutas
```

```bash
kubectl create -f pod-configmap.yaml
kubectl get pods

kubectl exec -ti busybox-configmap -- sh

set
# traz todas as frutas, inclusive a predileta

kubectl delete pods busybox-configmap
```

Através de volume mount:

```yaml
# vim prod-configmap-file.yaml

apiVersion: v1
kind: Pod
metadata:
  name: busybox-configmap-file
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-configmap
    command:
      - sleep
      - "3600"
    voluumeMounts:
    - name: meu-configmap-vol
      mountPath: /etc/frutas
  volumes:
  - name: meu-configmap-vol
    configMap:
      name: cores-frutas
```

```bash
kubectl create -f prod-configmap-file.yaml
kubectl get pods

kubectl exec -ti busybox-configmap-file -- sh

ls /etc/frutas
cat /etc/frutas/banana
ls -lha # link simbólico

kubectl delete pods busybox-configmap-file
```

## Init container

É um container que vai executar uma ação específica antes do container propriamente dito.

```yaml
# vim nginx-initcontainer.yaml

apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  initContainers:
    - name: install
      image: busybox
      command:
      - wget
      - "-O"
      - "/work-dir/index.html"
      - http://kubernetes.io
      volumeMounts:
      - name: workdir
        mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

O init container vai fazer o download do kubernetes.io no mesmo mount path do container.

```bash
kubectl create -f nginx-initcontainer.yaml
kubectl get pods

kubectl exec -ti init-demo -- sh

cat /usr/share/nginx/html/index.html
```

## RBAC

```bash
kubectl create serviceaccount alynne
kubectl get serviceaccounts
kubectl describe serviceaccounts alynne

kubectl get clusterrole
kubectl describe clusterrole cluster-admin # para ver as permissões
kubectl describe clusterrole view

kubectl get clusterrolebindings.rbac.authorization.k8s.io
# associações de usuário e as determinadas roles

kubectl get clusterrolebindings.rbac.authorization.k8s.io cluster-admin
```

Cluster role binding é quando associa uma regra (role) com algum usuário:

```bash
kubectl create clusterrolebindings toskeira --serviceaccount=default:alynne --clusterrole=cluster-admin

kubectl get clusterrolebindings.rbac.authorization.k8s.io

kubectl describe clusterrolebindings.rbac.authorization.k8s.io toskeira

kubectl describe serviceaccounts alynne # não traz admin
```

```yaml
# vim admin-user.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

```bash
kubectl create -f admin-user.yaml
kubectl serviceaccounts -n kube-system
```

```yaml
# vim admin-cluster-role-binding.yaml

apiVersion: rbac.authorizarion.k8s.io/v1beta1
kind: ClusterRole Binding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorizarion.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

Subjects (quem) será associado ao roleRef (o quê)

```bash
kubectl create -f admin-cluster-role-binding.yaml
kubectl get clusterrolebindings.rbac.authorization.k8s.io
kubectl describe clusterrolebindings.rbac.authorization.k8s.io admin-user
```

## Helm

Helm é uma forma de conseguir fazer a instalação de determinados softwares, como banco de dados. Ele usa um chart (tipo um playbook do ansible) para baixar o banco de acordo com as informações passadas. O Helm é um "apt install" da sua aplicação e das dependências.

Para instalar:

```bash
snap install helm --classic

# ou com wget
wget https://storage.googleapis.com/kubernetes-helm/helm-<versao>-linux-amd64.tar.gz
tar -xvzf helm-<versao>-linux-amd64.tar.gz
cd linux-amd64/
mv helm /usr/local/bin
mv tiller /usr/local/bin
helm version
```

```bash
helm init
kubectl create serviceaccount --namespace=kube-system tiller
kubectl create clusterrolebiding tiller-cluster-role --clusterrole cluster-admin --serviceaccount=kube-system:tiller
kubectl get clusterrolebidings.rbac.authorizarion.k8s.io

kubectl patch deployments -n kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
helm list # nada
helm search grafana
helm search nginx

kubectl create namespace monitoring

helm install --namespace=monitoring --name=prometheus --version=7.0.0 --set alertmanager.persistentVolume.enabled=false,server.persistentVolume.enabled=false stable/prometheus

```

Por padrão precisa do Persistent Volume. Está como falso para o exercício.
```bash
helm list
kubectl get pods
kubectl get pods -n monitoring
kubectl get services -n monitoring

kubectl edit service -n monitoring prometheus-server
```

Muda de `ClusterIP` para `NodePort`

```bash
helm install --namespace=monitoring --name=grafana --version=1.12.0 --set=adminUser=admin,adminPassword=admin,service.type=NodePort stable/grafana
kubectl get pods -n monitoring
kubectl get service -n monitoring

vim app-v1.yml
```

Annotation passa a informação para o cluster kubernetes e o serviço do prometheus fica procurando por ele em todos os deployments que estão em execução. Scrape é para pegar as informações do container automaticamente. Port é a porta que vai expor as métricas do serviço.

```bash
kubectl create -f app-v1.yml
kubectl get pods
```

No navegador, enra no `<ip>:<porta>/targets` para ver informação dos nods, pods, endpoints, etc. Se pegar a porta e o endpoint do pod (`<ip>:<porta_pod>/metrics`) em outra aba, será possível ver as métricas.

No prometheus, em graph: pesquisa a métrica `nginx_http_requests`. Ele já vai ter reconhecido, por ter capturado do container. Clica em `Execute` e clica em `Graph`.

No graphana, add source:

- name: Prometheus
- type: Prometheus
- URL: http://prometheus-server (nome do serviço do kubernetes)
- Access: Default
- Save & test

Home > dashboard > graph > panel title > edit

- `sum(rate(nginx_http_requests{app="giropops"}[5m])) by (version)`
- legend format: {{version}}
- display: bars

Salva como `nginx`

```bash
kubectl create -f app-v2-canary.yml
kubectl get deployments.
kubectl get pods

kubectl scale deployment --replicas=10 giropops-v2
kubectl get deployments.
```

```bash
helm list
helm --help
helm inspect stable/grafana
helm status grafana
helm delete grafana --purge
helm delete prometheus --purge

kubectl delete -f app-v1.yml
kubectl delete -f app-v2-canary.yml
```
