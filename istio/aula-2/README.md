# Istio - Aula 2

## Traffic Management

Virtual Service é um arquivo com todas as rotas e configurações de como lidar com as requisições.

Destination Rules configura as policies após o roteamento ser iniciado.


Nas máquina 1:

```bash
kubectl get virtualservices.networing.istio.io bookinfo -o yaml
kubectl get gateways.networking.istio.io

kubectl delete gateways.networking.istio.io bookinfo-gateway
kubectl delete virtualservices.networing.istio.io bookinfo

cd istio-1.2.2/
vim samples/bookinfo/networking/destination-rule-all.yaml

kubectl get deployments -o wide
kubectl get pods -o wide
kubectl get svc -o wide

kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
kubectl get destinationrules.networking.istio.io reviews -o yaml

vim samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

kubectl get virtualservices.networking.istio.io
kubectl get virtualservices.networking.istio.io reviews -o yaml
kubectl describe virtualservices.networking.istio.io reviews 

kubectl port-forward svc/productpage 9080:9080 -n default --address=0.0.0.0
```

No navegador: `<link_aws>:9080/productpage` e atualiza algumas vezes.

Muda o review de 1 para 2.

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl port-forward svc/productpage 9080:9080 -n default --address=0.0.0.0
```
No navegador: `<link_aws>:9080/productpage` e atualiza algumas vezes.

Muda o review de 2 para 3.

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl port-forward svc/productpage 9080:9080 -n default --address=0.0.0.0
```

Muda o review de 3 para 1.

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl port-forward svc/productpage 9080:9080 -n default --address=0.0.0.0
```

Para ver no Kiali:

```bash
kubectl port-forward svc/kiali 20001:20001 -n istio-system --address=0.0.0.0
```

Muda para a versão 3, faz apply e port-forward novamente. Verifica no Kiali.

Muda para a versão 1 e faz apply.

```bash
jobs
kill %1
kill %2
jobs
```

```bash
vim samples/bookinfo/networking/virtual-service-test-v2.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-test-v2.yaml
kubectl get virtualservices.networking.istio.io
kubectl get virtualservices.networking.istio.io reviews -o yaml

kubectl port-forward svc/productpage 9080:9080 -n default --address=0.0.0.0
kubectl port-forward svc/kiali 20001:20001 -n istio-system --address=0.0.0.0
```

Faz login com o usuário definido no yaml

### Traffic Shifting

```bash
kubectl delete -f samples/booktinfo/networking/virtual-service-all-v1.yaml
kubectl get destinationrules.networking.istio.io # mantém
kubectl apply -f samples/bookinfo/virtual-service-all-v1.yaml

vim samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

Weigth é o "peso" das requisições. No exemplo, vai 50% para cada versão.

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
kubectl get virtualservices.networking.istio.io
kubectl get virtualservices.networking.istio.io reviews -o yaml

kubectl port-forward svc/kiali 20001:20001 -n istio-system --address=0.0.0.0
jobs
```

Coloca o kiali para mostrar por porcentagem e atualiza a página do productpage várias vezes.

Muda no yaml para 90% na versão 1 e 10% na versão 3.

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

Aumenta o peso da versão 3 gradualmente, não de uma vez.

```bash
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### Fault Injection 

Serve para injetar falhas e testar a resiliência da aplicação.

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
vim samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

Se for "giropops" vai pra v2. Se for strigus vai pra v3. Se não for nenhum, vai pra v1

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
kubectl get virtualservices.networking.istio.io
kubectl get virtualservices.networking.istio.io reviews -o yaml

vim samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

Quando for "giropops", vai fazer o fault. Value é a porcentagem de vezes em que isso vai acontecer. Vai mandar o que for "giropops" pra v2 e o resto pro v1.

```bash
kubeclt apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
kubectl get virtualservices.networking.istio.io ratings -o yaml
```

Se mudar o fixedDelay para 3 segundos, o erro vai parar de acontecer.

```bash
kubectl get services -n istio-system
kubectl port-forward svc/jaeger 16686:16686 -n default --address=0.0.0.0
```

O Jaeger serve para mostrar os traces. Uma possibilidade de tag é "error=True"

```bash
vim samples/networking/virtual-service-ratings-test-abort.yaml
```

Muda o "exact" para giropops.

```bash
kubectl apply -f samples/networking/virtual-service-ratings-test-abort.yaml

kubectl get virtualservices.networking.istio.io ratings -o yaml
```
