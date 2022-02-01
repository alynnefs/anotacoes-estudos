# Kubernetes - Aula 2

## Services

Service é uma forma de fazer com que o deployment possa ser acessado de fora do cluster.

Controller que verifica se os pods morreram: replicaset
Controller do pod: replicaset
Controller do replicaset: deployment

Quando o arquivo é modificado, o deployment cria um novo replicaset. Caso precise de um rollback, ele fecha o novo e sobe o antigo

Pode ter mais de um container dentro de um pod, desde que tenha o mesmo ip, já que compartilham o namespace

Kube API Server é o "cérebro" do kubernetes. A comunicação dos nós é feita por ele. Somente ele tem acesso ao ETCD.

Kube Scheduler é quem determina em qual nó determinado pod vai ser hospedado.

Kube Controller Manager é o controller principal. É ele quem vai interagir com o Kube API Server para saber qual é o estado de cada um dos seus componentes.

Kube Proxy é responsável por gerenciar a rede dos containers. É ele quem vai fazer com que o nó exponha/libere as portas para que a comunicação funcione.

Supervisor D (?) é responsável por ver ser o kubelet e o docker estão rodando ou não. Se não estiverem, ele vai tentar reestabelecer por algumas vezes.

O kubernetes não utiliza nat.

Container Network Interface (CNI) é uma definição de uma biblioteca para conseguir fazer um plugin para que a rede funcione e para que tenha um bom gerenciamento das redes do container.

O container d (?) é o container runtime do docker.

Para verificar

```
kubectl get nodes
kubectl get deployments.
kubectl get services
kubectl get replicasets.
```

Para apagar

```
kubectl delete delpoyments. meu-nginx
kubectl delete delpoyments. nginx
kubectl delete service meu-nginx
```

`kubectl get pods` para confirmar que todos foram apagados

`systemctl restart kubelet` pra reinciar o serviço

`kubectl run nginx --image nginx --port=80`: se não colocar a tag de port, não será possível subir o service nele. O comando run será obsoleto nas versões futuras.

`kubectl get deployments.`

`kubectl expose deployment nginx` para criar o serviço. 

`kubectl get service` O tipo dele é ClusterIP. Significa que é um ip que só será usado dentro do cluster. Não precisa de ip para conversar dentro do cluster, ele já tem um DNS integrado

`kubectl get deployments. --all-namespaces`: namespace - kube-system, name - coredns

`kubectl get endpoints`: o endpoint é o ip do pod.

`kubectl scale --replicas=10 deployment nginx`

`kubectl get endpoints` vai mostrar vários ips

Se fizer `curl <ip_cluster_nginx>` ele retorna a página do nginx

`kubectl get pods`

`curl <ip_cluster_nginx>` algumas vezes para ver alternando no log

`kubectl logs -f <nome>`

### ClusterIP

`kubectl get services nginx -o yaml > meu_primeiro_service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  clusterIP: 10.105.180.162 # pode apagar se quiser que seja dinâmico
  ports:
  - port: 80 # container
    protocol: TCP
    targetPort: 80 # pod
  selector: # pra qual deployment este serviço está rodando
    run: nginx # que tem esse run
  sessionAffinity: None
  type: ClusterIP # pode mudar para NodePort ou LoadBalancer
```

```
kubectl delete service nginx
kubectl create -f meu_primeiro_service.yaml
kubectl get services
kubectl delete service nginx
```

### NodePort

```
kubectl expose deployment nginx --type=NodePort
kubectl get services
```

Mesmo sendo NodePort, ele trouxe o ClusterIP.

`curl <endereço_externo>` traz o html do nginx

`kubectl describe service nginx`

`kubectl get services nginx -o yaml > meu_primeiro_service_nodeport.yaml`

```
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32222
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: NodePort
```

O range disponível para o node port incia em 30000 até 32000 (e pouco)

```
kubectl delete service nginx
kubectl create -f meu_primeiro_service_nodeport.yaml
kubectl get services
```

### Load Balancer

```
kubectl delete service nginx
kubectl expose deployment nginx --type=LoadBalancer
kubectl get services
kubectl get endpoints
kubectl create -f meu_primeiro_service_loadbalancer.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32333
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: LoadBalancer
```

```
kubectl delete service nginx
kubectl create -f meu_primeiro_service_loadbalancer.yaml
kubectl get services
kubectl describe endpoints nginx
```

### Deployment e service no mesmo arquivo

`vim meu_primeiro_deployment.yaml`

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
  labels:
    run: nginx
  name: meu-nginx
  namespace: default
spec:
  replicas: 10
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
    dnsPolicy: Always
    terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32222
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: NodePort
```

```
kubectl delete deployments. nginx
kubectl create -f meu_primeiro_deployment.yaml
```

Ele cria o deployment e o serviço.

### Comandos

```
Add-ons:
https://kubernetes.io/docs/concepts/cluster-administration/networking/
https://chrislovecnm.com/kubernetes/cni/choosing-a-cni-provider/
https://kubernetes.io/docs/concepts/cluster-administration/addons/
```

```
# vim primeiro-service-clusterip.yaml:

apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx-clusterip
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: ClusterIP
```

```
# vim primeiro-service-nodeip.yaml:

apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx-nodeport
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32548
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: ClientIP
  type: NodePort
```

```
# vim primeiro-service-loadbalancer.yaml:

apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx-loadbalancer
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32548
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: LoadBalancer
```

```
# kubectl create -f primeiro-service-clusterip.yaml

# kubectl create -f primeiro-service-nodeip.yaml

# kubectl create -f primeiro-service-loadbalancer.yaml

# kubectl edit -f primeiro-service-nodeip.yaml

# kubectl get deploy nginx -o yaml > primeiro-deployment-limitado.yaml

# kubectl exec -ti nginx-limitado-8d767cd5f-lg5xs -- /bin/bash

# kubectl replace -f deployment_limitado.yaml

# kubectl get endpoints

# kubectl describe endpoints nginx-ddswrgb
```

```
# vim deployment-limitado.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-limitado
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          limits:
            memory: 128Mi
            cpu: 1
          requests:
            memory: 96Mi
            cpu: 1
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

## Limitando recursos

Copia o `meu_primeiro_deployment.yaml` para `deployment_limitado.yaml` e apaga o service. Atenção para os campos `dnsPolicy`, `retartPolicy` e resources.

500m equivale à metade de um core. Também poderia ser 0.5, mas é preferência "do kubernetes". 250m equivale a 0.25.

"Mi" é a forma que preferem que a memória seja definida.

Limits é o total de recurso que o kubernetes vai liberar e requests é o quanto ele vai garantir.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
  labels:
    run: nginx
  name: meu-nginx
  namespace: default
spec:
  replicas: 10
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          limits:
            memory: 512Mi
            cpu: "500m"
          requests:
            memory: 256Mi
            cpu: "250m"
    dnsPolicy: ClusterFirst
    restartPolicy: Always
    terminationGracePeriodSeconds: 30
```

```
kubectl delete deployments. meu-nginx
kubectl create -f deployment_limitado.yaml
kubectl get deployments.
kubectl describe deployments. meu-nginx
kubectl delete service nginx
kubectl create -f meu_primeiro_service.yaml
kubectl get services
kubectl edit service nginx
```

muda o type de ClusterIP para NodePort

`kubectl get services` para confirmar que foi editado

`kubectl edit deployments. meu-nginx`

```
kubectl get deployments.
kubectl get pods
kubectl get pods -o wide
```

`kubectl exec -ti <nome_pod> -- bash` para entrar no pod

```
apt update && apt install -y stress
stress --vm 1 --vm-bytes 128M
stress --vm 1 --vm-bytes 300M
stress --vm 1 --vm-bytes 500M
# ...
stress --vm 1 --vm-bytes 510M # estourou
stress --vm 1 --vm-bytes 507M
# ...
```

## Limit Range

```
kubectl create namespace giropops
kubectl get namespaces
kubectl describe namespaces giropops
kubectl get namespaces giropops -o yaml > meu_primeiro_namespace.yaml
```

```
apiVersion: v1
kind: Namespace
metadata:
  name: strigus
spec:
  finalizers:
  - kubernetes
```

```
kubectl create -f meu_primeiro_namespace.yaml
kubectl get namespaces
```

```
# vim namespace_limitado.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limitando-recursos
spec:
  limits:
  - default:
      cpu: 1
      memory: 256Mi
    defaultRequest:
      cpu: 0.5
      memory: 128Mi
    type: Container
```

```
kubectl create -f namespace_limitado.yaml -n giropops
kubectl get limitranges # No resources found
kubectl get limitranges -n giropops

kubectl describe limitranges
kubectl describe limitranges -n giropops limitando-recursos
```

Tudo o que criar neste namespace já vai estar limitado. Para criar um pod:

```
# vim pod-limitado.yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit-pod
  namespace: giropops
spec:
  containers:
  - name: meu-container
    image: nginx
```

```
kubectl create -f pod-limitado.yaml -n giropops # se tá no yaml, não precisa do -n
kubectl get pods # no resources found
kubectl get pods -n giropops
kubectl describe pods -n giropops
kubectl delete -f pod-limitado.yaml
```

### Comandos

```
# vim limitando-recursos.yaml:

apiVersion: v1
kind: LimitRange
metadata:
  name: limitando-recursos
spec:
  limits:
  - default:
      cpu: 1
      memory: 100Mi
    defaultRequest:
      cpu: 0.5
      memory: 80Mi
    type: Container


# vim pod-limitrange.yml

apiVersion: v1
kind: Pod
metadata:
  name: pod-limitrange
spec:
  containers:
    - name: limitado-pod
      image: nginx
```

```
# kubectl create namespace primeiro-namespace

# kubectl get ns

# kubectl describe primeiro-namespace

# kubectl describe namespace primeiro-namespace

# kubectl create -f limitando-recursos.yaml -n primeiro-namespace

# kubectl get limitranges

# kubectl get limitranges -n primeiro-namespace

# kubectl get limitranges --all-namespace

# kubectl describe limitranges -n primeiro-namespace

# kubectl create -f pod-limitrange.yaml

# kubectl create -f pod-limitrange.yaml -n primeiro-namespace

# kubectl get pods --all-namespaces

#kubectl describe pod limit-pod

# kubectl describe pod limit-pod -n primeiro-namespace
```

## Taints

`kubectl describe nodes maquina-2` para ver `Taints: <none>`

`kubectl describe nodes maquina-1` para ver `Taints: node-role.kubernetes.io/master:NoSchedule`. Isso indica que não é para agendar mais nenhum pod. Ele vai manter os pods que estão executando, mas não vai ter mais novos deploys.

```
kubectl create -f deployment_limitado.yaml
kubectl get deployments. --all-namespaces # para ver quantidade de pods
kubectl get pods -o wide # estão nas máquinas 2 e 3
kubectl scale --replicas=6 deployment meu-nginx
kubectl taint node maquina-2 key1=value1:NoSchedule
```

`kubectl describe nodes maquina-2` para ver `Taints: key1=value1:NoSchedule`

```
kubectl edit deployments. meu-nginx
```
Apaga os limites de resources

```
kubectl scale --replicas=10 deployment meu-nginx
kubectl get deployments.
kubectl get pods -o wide
```

```
kubectl delete deployments. meu-nginx
kubectl taint node maquina-2 key1:NoSchedule-

kubectl create -f deployment_limitado.yaml
kubectl get pods -o wide
kubectl taint node maquina-2 key1=value1:NoExecute
kubectl describe nodes maquina-2
kubectl get pods -o wide
```

Todos os pods que estavam no 2 foram movidos para o 3.

```
kubectl taint node maquina-2 key1:NoExecute-
kubectl get pods -o wide
```

Não volta a balancear o cluster.

`kubectl taint nodes --all key1=value1:NoExecute` iria deixar TODOS os nós sem execução.

### Comandos

```
# kubectl describe nodes elliot-01 | grep -i taint
Taints:             node-role.kubernetes.io/master:NoSchedule

# kubectl describe nodes  | grep -i taints

# kubectl taint node elliot-01 node-role.kubernetes.io/master:NoSchedule-

# kubectl taint node elliot-01 node-role.kubernetes.io/master:NoSchedule

# kubectl taint node elliot-02 key1=value1:NoSchedule

# kubectl taint node elliot-02 key1=value1:NoSchedule-

# kubectl get pods -o wide

# kubectl taint node elliot-02 key1=value1:NoExecute-

# kubectl taint node elliot-02 key1=value1:NoExecute

# kubectl get pods -o wide

# kubectl taint node all key1=value1:NoExecute
```

Quando cria um ClusterIP, cria só um ClusterIP. Quando cria um NodePort, cria um NodePort e um ClusterIP. Quando cria um LoadBalancer, cria um LoadBalancer, um NodePort e um ClusterIP.

`get deployments.apps nginx-tosko -o wide`

`kubectl cordon <nome_nó>` não agenda mais coisas nele (SchedulingDisabled)

`kubectl cordon <nome_nó>` volta a agendar coisas nele
