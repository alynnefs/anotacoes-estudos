# Istio - Aula 1

## Mesh e Istio

Service mesh é uma solução que dá a possibilidade de controlar os microserviços, e que dá a opção de dar um "observability", dando acesso às métricas, logs, fazer deployment, etc. Istio é um service mesh muito utilizado pela comunidade.

Nas 3 máquinas:

```bash
curl -fsSL https://get.docker.com | bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/k8s.list
cat /etc/apt/sources.list.d/k8s.list
apt update
apt install -y kubeadm kubelet kubectl
```

Na máquina 1:

```bash
kubeadm config images pull
kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

kubectl get nodes
kubectl get pods --all-namespaces
```

Cola o comando `kubeadm join` nas máquinas 2 e 3. Na máquina 1:

```bash
kubectl get nodes

curl -L https://git.io/getLatestIstio |ISTIO_VERSION=1.2.2 sh -
```

No navegador: https://istio.io/docs/setup/kubernetes/install

```bash
cd istio-1.2.2/
export PATH=$PWD/bin:$PATH
istioctl get gateway
```

https://istio.io/docs/setup/kubernetes/install/kubernetes/

```bash
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```

CRD (Custom Resources Definition) cria resources e objetos personalizados para o kubernetes.

```bash
kubectl api-resources | grep istio
kubectl get crd | grep istio
kubectl apply -f install/kubernetes/istio-demo.yaml

kubectl get service -n istio-system
kubectl get deploy -n istio-system
kubectl get pods -n istio-system

# kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

## Deployment

```bash
echo "source <(kubectl completion bash)" >> /root/.bashrc # para dar autocomplete

kubectl create namespace apps
kubectl get namespace
kubectl label namespaces default istio-injection=enabled
vim samples/bookinfo/platform/kube/bookinfo.yaml
```

Explicação do exemplo: https://istio.io/docs/examples/bookinfo

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get deployments.
kubectl get pods
kubectl describe pods reviews-v3-<hash>
kubectl get services
kubectl edit service productpage

kubectl port-forward svc/productpage 9080:9080 -n default --address:0.0.0.0
```

No navegador: `<link_aws>/9080/productpage`

```bash
kubectl get pods
vim samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

kubectl get gateways.networking.istio.io
kubectl describe gateways.networking.istio.io bookinfo-gateway
kubectl get pods -n istio-system

kubectl get virtualservices.networking.istio.io
kubectl describe virtualservices.networking.istio.io bookinfo
kubectl get virtualservices.networking.istio.io bookinfo -o yaml

kubectl port-forward svc/productpage 9080:9080 -n default --address=0.0.0.0
```

No navegador: `<link_aws>/9080/productpage`

## Kiali e Grafana 

```bash
kubectl get pods -n istio-system
kubectl get services -n istio-system

kubectl edit service kiali -n istio-system
kubeadm port-forward svc/kiali 20001:20001 -n istio-system --address=0.0.0.0
```

No navegador: `<url_aws>:20001/kiali/`. O padrão é "admin" para usuário e senha. Em outro terminal:

```bash
kubeadm port-forward svc/productpage 9080:9080 -n default --address=0.0.0.0
```

Em outra aba do navegador:`<url_aws>:9080/productpage` e atualiza algumas vezes.

No kiali: graph > namespace: default > workload graph, no edge labels, display

```bash
kubeadm port-forward svc/grafana 3000:3000 -n default --address=0.0.0.0
```

Em outra aba do navegador: `<url_aws>:3000`
