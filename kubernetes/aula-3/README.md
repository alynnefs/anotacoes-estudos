# Kubernetes - Aula 3

## Deployment

Deployment é o controller do replicaset. Quando eu crio um deployment, ele cria um replicaset com a quantidade de pods solicitada.

Quando altera a versão do pod, ficam dois replicasets, um parado e outro em execução. Isso é importante para o caso de precisar fazer um rollback.

`vim primeiro-deployment.yaml`

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
    app: giropops
  name: primeiro-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
        dc: UK
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx2
        ports:
        - containterPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

`restartPolicy: Always`: se parar, vai reiniciar, a menos que peça para encerrar. Mas se "matar" um pod, o replicaset vai perceber e colocar outro no lugar.

```
kubectl create -f primeiro_deployment.yaml
kubectl get deployments.
cp primeiro_deployment.yaml segundo_deployment.yaml
vim segundo_deployment.yaml
```

Muda o nome e o dc do label
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
    app: giropops
  name: segundo-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
        dc: NL
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx2
        ports:
        - containterPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

```
kubectl create -f segundo_deployment.yaml
kubectl get deployments.
```

## Labels

`kubectl describe deployments primeiro-deployment` para verificar as labels

`kubectl get replicasets.` para ver os nomes

`kubectl describe replicasets primeiro-deployment-<hash>`

```
kubectl get pods
kubectl describe pods primeiro-deployment-<hash>
```

`kubectl get pods -l dc=NL` mostra todos os pods onde a label dc é igual a "NL"

`kubectl get pods -L dc` adiciona a coluna dc

`kubectl get replicasets. -l dc=NL`

```
kubectl label nodes maquina-2 disk=SSD
kubectl label nodes maquina-3 disk=HDD

kubectl describe nodes maquina-2

kubectl label nodes maquina-2 --list

kubectl label nodes maquina-3 disk=SSD --overwrite
kubectl label nodes maquina-3 --list
kubectl label nodes maquina-3 disk=HDD --overwrite
```

Só atualiza com "overwrite"

```
kubectl label nodes maquina-3 dc=NL
kubectl label nodes maquina-2 dc=UK
```

```
cp primeiro_deployment.yaml terceiro_deployment.yaml
vim terceiro_deployment.yaml
```

Modifica abaixo do terminationGracePeriodSeconds

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
    app: giropops
  name: terceiro-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
        dc: UK
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx2
        ports:
        - containterPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      nodeSelector:
        disk: SSD
```

```
kubectl create deployment -f terceiro_deployment.yaml
kubectl get deployments.
kubectl label nodes maquina-3 --list
kubectl label nodes maquina-2 --list
```

SSD aparece na maquina-2. Para conferir se subiu o terceiro deployment lá:

`kubectl get pods -o wide`

`kubectl edit deployments. terceiro-deployment` e muda SSD para HDD

```
kubectl get pods -o wide
kubectl describe deployments. terceiro-deployment
kubectl get replicasets.
```

`kubectl label nodes dc- --all` remove as labels dc de todos os nós

```
kubectl label nodes maquina-2 --list
kubectl label nodes disk- --all
kubectl label nodes maquina-2 --list
kubectl label nodes maquina-3 --list
```

Apenas para fins didáticos, não aconselhado em produção:

`vim primeiro-replicaset.yaml`

```
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: replica-set-primeiro
spec:
  replicas: 3
  template:
    metadata:
      labels:
        system: Giropops
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```
kubectl create -f primeiro-replicaset.yaml
kubectl get replicasets
kubectl get deployments.
```

Ele não criou nenhum deployment, já que deployment é o controller do replicaset 

Nomenclaturas para usar com `kubectl get`:

- rs: replicaset
- po: pod
- deploy: deployment
- svc: service
- ep: endpoint

```
kubectl describe rs replica-set-primeiro
kubectl get pods
kubectl delete pod replica-set-primeiro-<hash>
kubectl get pods
```

Aparece outro pod no lugar do que foi apagado, já que a função do replicaset é manter o controle dos pods

`kubectl describe rs replica-set-primeiro` para ver que um pod tem poucos segundos de existência

`kubectl edit replicasets. replica-set-primeiro`

Muda as réplicas para 4

```
kubectl get rs
kubectl describe rs replica-set-primeiro
kubectl get pods -L system
```

Muda a versão do nginx para 1.15.0

```
kubectl describe rs replica-set-primeiro
kubectl get rs
kubectl get pods -L system
```

Ainda está a versão antiga do nginx.

```
kubectl delete pod replica-set-primeiro-<hash>
kubectl get pods -L system
```

Outro container está sendo criado

`kubectl describe pods replica-set-primeiro-<hash>` para ver a diferença de versão do nginx entre os pods. Apenas um dos pods está com a versão certa. É por isso que devemos criar replicasets pelo deployment. Ele teria atualizado todas as réplicas.

`kubectl delete -f primeiro-replicaset.yaml` para remover todos ao mesmo tempo

```
kubectl delete -f primeiro_deployment.yaml
kubectl delete -f segundo_deployment.yaml
kubectl delete -f terceiro_deployment.yaml
```

## Daemonset

Daemonset é praticamente um replicaset, mas não se escolhe quantas réplicas dele vai ter. Ele garante que vai rodar um pod em cada um dos nós.

`vim primeiro-daemonset.yaml`

```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: daemon-set-primeiro
spec:
  template:
    metadata:
      labels:
        system: Strigus
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```
kubectl create -f primeiro-daemonset.yaml
kubectl get damonsets # ou ds
get pods -o wide
```

Não sobe no master por causa do taint

Para tirar o taint da maquina-1: `kubectl taint node maquina-1 node-role.kubernetes.io/master-`. Mas isso não é recomendado. Para colocar de volta: `kubectl taint node maquina-1 node-role.kubernetes.io/master-=value1:NoSchedule`

```
kubectl set image ds daemon-set-primeiro nginx=nginx:1.15.0
kubectl get pods -o wide

kubectl describe ds daemon-set-primeiro # 1.15.0
kubectl describe pod daemon-set-primeiro-<hash> # 1.7.9

kubectl delete pods daemon-set-primeiro-<hash> # cria outro depois que apaga
```

## Rollouts e Rollbacks 

```
kubectl rollout history daemonset daemon-set-primeiro
```
