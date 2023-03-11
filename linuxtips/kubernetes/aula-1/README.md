# Kubernetes - aula 1

[play with kubernetes](https://labs.play-with-k8s.com/)

https://kubernetes.io

https://github.com/kubernetes/kubernetes

https://github.com/badtuxx/DescomplicandoKubernetes

Kubernetes é um orquestrador de containeres. Significa piloto ou timoneiro.

Toda ação que acontece no cluster passa pela API. ETCD é onde tem todas as informações do cluster. O scheduler é quem vai ter o "olhar crítico" para saber onde é melhor criar o pod, etc.

Pod é a menor unidade do kubernetes. O pod divide o namespace e pode ter mais de um container, tendo apenas um ip. Caso tenha mais de um container, eles compartilham o volume.

Controller é responsável por fazer a orquestração do cluster. É ele quem se comunica com o API server para verificar o status dos pods. Deployment é um dos principais controllers.

## Minikube

Usar apenas para testes, não em projetos reais. Ele "sobe" um ambiente com kubernetes, com tudo dentro de uma máquina virtual. Deverá ser usado na máquina local

Kubectl é quem vai gerenciar os deployments, services e etc dentro do cluster. Para instalar o kubectl no linux:

`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`

É possível instalar o minikube usando o .deb do site

https://kubernetes.io/docs/tasks/tools/install-minikube/

https://github.com/kubernetes/minikube/release


Todos os serviços estão rodando na mesma máquina, que é virtual

`minikube start` para iniciar o minikube

`kubectl get nodes` para verificar os nós

`kubectl get pods` para verificar os pods

`kubectl get deployment` para verificar os deployments

`kubectl get deployment --all-namespaces` para verificar todos os namespaces



`minikube ip` para ver o ip do minikube

`minikube ssh` para acessar a máquina por ssh

`minikube delete`

## Kubernetes

### versão 1.19.3

3 máquinas EC2 médias da AWS

Instala o docker nas 3 máquinas, sendo a máquina 1 a principal e as outras duas auxiliares.

Nas 3 máquinas:

vim /etc/docker/daemon.json

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

`mkdir -p /etc/systemd/system/docker.sercive.d`

`systemctl daemon-reload`

`systemctl restart docker`

`docker info | grep -i cgroup` 

para pegar a chave e "marcar" como seguro:

`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`

para adicionar o endereço do repositório no apt:

`echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list`

`sudo apt update`

para instalar os pacotes necessários para criar o cluster. O kubeadm vai formar o cluster, o kubelet vai interagir e o kubectl vai controlar

`sudo apt install -y kubeadm kubelet kubectl`

para inicializar o cluster:

Na máquina 1:

`kubeadm config images pull`

- API Server é quem sabe tudo o que está acontecendo com o cluster
- Controller Manager controla em qual nó vai jogar os deployments
- Scheduler vai escolher quem é o melhor nó
- Proxy é responsável por expor os serviços para o mundo
- ETCD banco que contém todo o status do cluster. Quem conversa com ele é apenas o API Server. As informações de tudo o que acontece são gravadas nele
- Core DNS serve para ter um DNS interno


Para começar a usar o cluster, precisa dos seguintes comandos (que foram informados pelo kubeadm):

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Para subir módulos do kernel, apenas para garantir:

`modprobe br_netfilter ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4 ip_vs`

`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

`kubectl get nodes` traz o primeiro, que está funcionando

`kubetcl get pods -n kube-system` traz todos os pods do namespace (-n) "kube-system"

para adicionar outros nós no cluster:

`kubeadm join <ip>:<porta> --token <token> --discovery-token-ca-cert-hash sha256:<hash>`

esse comando aparece na saída do kubeadm que foi feito anteriormente. Ele deve ser colado nas máquinas 2 e 3. Caso não tenha mais no histórico, o comando pode ser recuperado com `kubeadm token create --print-join-command`

com `kubectl get nodes` as outras máquinas vão aparecer

`kubectl get pods -n kube-system -o wide` traz em qual nó está rodando cada pod


### versão 1.14

`cat /proc/cpuinfo` para ver a quantidade de CPUs

`free -m` para ver a quantidade de memória

Por default não se executa nenhum container no master, mas é possível.


`sudo su -`

`hostname maquina-N`, sendo N a identificação de cada uma das máquinas

`bash` para ver o nome, logout e entra de novo com `ssh -o ServerAliveInterval=30 -i "chave.pem" ubuntu@ec2-<código_específico>.compute-1.amazonaws.com`, apenas para "reiniciar" e conferir o hostname. Entra em sudo de novo.

Não obrigatório, mas para adicionar módulos de kernel:

`vim /etc/modules-load.d/k8s.conf`

```vim
br_netfilter
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
ip_vs
```

`apt update && apt upgrade` e reinicia as 3 máquinas.

Instala o docker com `curl -fsSL https://get.docker.com | bash`

`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`

`echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/k8s.list`

`cat /etc/apt/sources.list.d/k8s.list` para conferir

`apt update && apt install -y kubelet kubeadm kubectl`

Na máquina 1:

`kubeadm config images pull` para baixar as imagens antes do init e poupar tempo.

`kubeadm init` para inicializar o cluster

Aparece um warning sobre o systemd ser o driver recomendado, ao invés do cgroupfs, que está sendo utilizado. Mas é apenas uma recomendação, ambos podem ser usados.

- RBAC significa Roled Based Access Control
- Control Plane são todas as imagens que pegamos no config images pull

Copia os comandos de inicialização do cluster

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Como está como root, não é necessário.

`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"` para aplicar o arquivo que vai pegar e executar

`kubectl get node` só irá mostrar um.

`kubectl get pods --all-namespaces` para trazer todos os namespaces. Se tiver "2/2", então tem 2 containers rodando no pod.

Nas máquinas 2 e 3:

Copia o código do kubeadm join e cola nas máquinas

`kubeadm join <ip>:<porta> --token <token> --discovery-token-ca-cert-hash sha256:<hash>`

`kubectl get nodes` na máquina 1 para ver que as 3 aparecem

Nas 3 máquinas:

`apt install -y bash-completion`

`kubectl completion bash > /etc/bash_completion.d/kubectl` para ter o autocomplete

`source <(kubectl completion bash)` para forçar a leitura 

### Primeiros passos - versão  1.19.3

Na máquina 1:

`kubectl describe [nodes | pod | services]`

`kubectl describe nodes <maquina-1>` traz todos os detalhes desde quando subiu a máquina.

Faz os passos do completion listados acima

`kubectl describe pods -n kube-system weave-net-glj59`

opções para ver os pods:

`kubectl get pods` quando não especifica nada, ele faz a busca no default

`kubectl get pods --all-namespaces`

`kubectl get pods --all-namespaces -o wide`

para ver os namespaces:

`kubectl get namespaces`

Kube-node-lease, kube-public e kube-system são usados pelo kubernetes. Para criar uma nova namespace para sua aplicação:

`kubectl create namespace giropops`

`kube get namespaces` para ver o giropops

`kubectl run nginx --image nginx`

`kubectl get pods`

`kubectl describe pods nginx`

`kubectl describe pods nginx -o yaml` para ver o objeto do pod no formato yaml

`kubectl get pods -o yaml > meu_primeiro_pod.yaml` 

Nem todas as informações são necessárias para criar um novo pod

- remove managedFields
- muda o namespace para giropops
- apaga o resourceVersion, selfLink e uid
- em spec, apaga resources, terminationMessagePath, terminationMessagePolicy, volumeMounts, enableServiceLinks, nodeName (não vai subir um, é o status do dump), priority, schedulerName, securityContext, serviceAccount, serviceAccountName, terminationGracePeriod, tolerations, volumes, status, containerStatuses, hostIP, phase, podIp, podIps, qosClass, startTime
- apaga 

Basicamente, o arquivo fica assim:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: giropops
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: ngix
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

`kubectl get pods -n default` vai mostrar o nginx

`kubectl get pods -n giropops` não vai ter nada

`kubectl create -f meu_primeiro_pod.yaml`

-f não é de force, é de file name 

`kubectl get pods -n giropops` agora vai mostrar o nginx

`kubectl delete -f meu_primeiro_pod.yaml`

`kubectl get pods -n giropops` o nginx não vai mais aparecer

`kubectl run nginx --image=nginx --dry-run=client` vai "fingir" que criou

`kubectl run nginx --image=nginx --dry-run=client -o yaml` vai retornar o que precisa para criar um novo. Isso é melhor do que apagar as linhas não utilizadas

`kubectl run nginx --image=nginx --dry-run=client -o yaml > meu_segundo_pod.yaml` para salvar no arquivo

`kubectl create -f meu_segundo_pod.yaml`

`kubectl expose pod nginx` dá erro por não ter encontrado porta

`kubectl delete pods nginx`

`kubectl create -f meu_segundo_pod.yaml`

`kubectl describe pods nginx`

o Host Port está 0 por não ter sido vinculado

`kubectl expose pod nginx`

`kubectl get service` para ver as portas

CLUSTER-IP é um IP que só vale dentro cluster e EXTERNAL-IP seria o IP que daria pra acessar de fora, caso tivesse um load balancer

Se fizer um curl no cluster-ip, aparecerá a página de boas vindas do nginx

`kubectl describe services nginx` para pegar a descrição formatada

`kubectl edit service nginx` para editar. Se mudar o type para `NodePort`, ele vai adicionar uma porta em todo o cluster. Ou seja, pode "bater" em qualquer IP do cluster nessa porta e vai funcionar. Se colocar <ip>:<porta> no navegador, vai aparecer o html
	
`kubectl explain pod` mostra a documentação do que é, qual a versão, etc. Também funciona com services e os outros valores.
	
comandos equivalentes:
- `kubectl get all` e `kubectl get pods,services,endpoins`, com a diferença que os endpoints não aparecem no all
- `kubectl get pods` e `kubectl get po`
- `kubectl get services` e `kubectl get svc`
- `kubectl get deployments.apps` e `kubectl get deploy`
- `kubectl get namespaces` e `kubectl get ns`

	
Para apagar:

`kubectl delete pods nginx`
	
`kubectl delete service nginx`
	
`kubectl get all` para verificar se apagou
	
	
## Comandos
	
```bash
# vim /etc/modules-load.d/k8s.conf
br_netfilter
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
ip_vs


# apt-get update -y && apt-get upgrade -y

# curl -fsSL https://get.docker.com | bash

# apt-get update && apt-get install -y apt-transport-https

# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

# echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

# apt-get update

# apt-get install -y kubelet kubeadm kubectl

# swapoff -a

# vim /etc/fstab

# kubeadm config images pull

# kubeadm init

# mkdir -p $HOME/.kube

# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# sudo chown $(id -u):$(id -g) $HOME/.kube/config

# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# kubectl get pods -n kube-system

# kubeadm join --token 39c341.a3bc3c4dd49758d5 IP_DO_MASTER:6443 --discovery-token-ca-cert-hash sha256:37092 

# kubectl get nodes

# kubectl describe node elliot-03

# kubeadm token create --print-join-command

# echo "source <(kubectl completion bash)" >> ~/.bashrc

# kubectl get namespace

# kubectl get pods -n kube-system

# kubectl get pods --all-namespaces

# kubectl run nginx --image nginx

# kubectl get deployments

# kubectl describe deployment nginx

# kubectl get events

# kubectl get deployment nginx -o yaml

# kubectl get deployment nginx -o yaml
	> meu_primeiro.yaml

# kubectl delete deployment nginx

# kubectl create -f meu_primeiro.yaml

# kubectl delete -f meu_primeiro.yaml

# kubectl get deploy,pod

# kubectl expose deployment/nginx

# kubectl get svc nginx 

# kubectl describe pod nginx-6f858d4d45-qxjlh

# kubectl get pods -o wide

# kubectl delete pods nginx-6f858d4d45-qxjlh	
```
	
Exercício:
	
- Como eu posso criar um service para um deployment em execução?
	`kubectl expose deployment/nome_do_deployment`
- Eu posso ter a swap habilitada em meu cluster?
	Não
	
## Cluster multi-master
	
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
	
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology
	
haproxy.com para o load balancer
	
O que precisa no load balancer: um DNS e um IP que aponte para os nós. A porta é 6443/TCP. Os workers conversam com o load balancer, que vai fazer a distribuição no control plane node. Se estiver separado, o ETCD estará em outros nós.

Para criar o setup, serão considerados 3 workers, 3 masters e um load balancer. Serão 7 máquinas. Para wokers e masters, o aconselhado são 2 cores de CPU e 2gb de memória.
	
- Cria 6 instâncias de ubuntu na AWS. Sevurity Group Name: k8s-multi-master. Sendo load balancer (ha-proxy) micro e o restante medium.
- No security group: inbound > edit > add rule > All Traffic, source k8s-multi-master
- Connect > copia o ssh de cada master e cola nos terminais, sem apertar enter. Faz o mesmo com o ha-proxy, mas conecta.
- Na máquina ha-proxy:
	```bash
	sudo su -
	hostname k8s-haproxy-01
	echo "hostname k8s-haproxy-01" > /etc/hostname
	bash
	apt update
	apt install -y haproxy
	vim /etc/haproxy/haproxy.cfg
	```

Adiciona no fim do arquivo:

```vim
frontend kubernetes
	mode tcp
	bind <ip_da_maquina>:6443
	option tcplog
	default_backend k8s-masters

backend k8s-masters
	mode tcp
	balance roundrobin
	option tcp-check
	server k8s-master-0 <ip_privado>:6443 check fall 3 rise 2
	server k8s-master-1 <ip_privado>:6443 check fall 3 rise 2
	server k8s-master-2 <ip_privado>:6443 check fall 3 rise 2	
```
	

Frontend é ip que vai redirecionar para os 3 nós master. É o VirtualIP.
Roundrobin "joga" uma conexão por vez para cada servidor.
Check fall são quantas falhas serão consideradas para ter o host como down.
Rise são quantos health check positivos vai considerar para ter o host como up.
	
```bash
systemctl restart haproxy
systemctl status haproxy
tail -f /var/log/haproxy.log
ntstat -atunp
```

O netstat serve para ver o status da porta 6443.
	
- Conecta nas 3 masters e segue os passos:

```bash
sudo su -
apt update
hostname k8s-master-N
echo k8s-master-N > /etc/hostname
bash
curl -fsSL https://get.docker.com | sh
docker version
```

N é 1, 2, 3, um para cada máquina.
	
`vim /etc/hosts`
	
adiciona
	
```bash
<load_balancer_ip> k8s-haproxy
```
	
Com isso, quando fizer `ping k8s-haproxy`, ele vai alcançar o haproxy/load balancer
	
Copia o trecho de instalação na [documentação do kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) e cola nos terminais.
	
```bash
sudo apt update $$ sudo apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
	
- Copia o código para inicializar o control plane, encontrado [na documentação](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) e cola apenas na máquina 1, modificando as variáveis 
	
`sudo kubeadm init --control-plane-endpoint "<load_balancer_dns>:<load_balancer_ports>" --upload-certs`

Como o DNS foi definido nas máquinas, fica

`sudo kubeadm init --control-plane-endpoint "k8s-haproxy:6443" --upload-certs`
	
- Copia as seguintes linhas, para configurar o kubectl, e cola na máquina 1
	
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
kubectl get nodes
kubectl get all --all-namespaces
```	

Pode demorar um pouco para terminar e aparecer com os comandos acima. Copia o comando completo do `kubeadm join` e cola nas máquinas 2 e 3.
	
- Adiciona nas máquinas 2 e 3 a configuração para usar o kubectl, porque os 3 nós são master
	
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
	
- Nas 3 máquinas worker:

Copia os comandos ssh da AWS, colando cada um em seu respectivo terminal
	
```bash
sudo su -
hostname k8s-worker-N
echo k8s-worker-N > /etc/hostname
bash
vim /etc/hosts
```
	
e adiciona o ip do haproxy:

```bash
<load_balancer_ip> k8s-haproxy
```
	
`ping k8s-haproxy`
	
Instala o docker e o kubernetes. Depois disso, copia o join de worker que está no master 1 e cola nos 3 workers. O comando é parecido com o seguinte:

`kubeadm join k8s-haproxy:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>`
	
- Na master 1:
	
`kubectl get all --all-namespaces` para ver tudo do cluster

```bash
kubectl run nginx --image nginx --replicas 10
kubectl get deploy
kubectl get pods
kubectl get pods -o wide
```
