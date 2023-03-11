# Kubernetes - Aula 3

## Deployment

Deployment é o controller do replicaset. Quando eu crio um deployment, ele cria um replicaset com a quantidade de pods solicitada.

Quando altera a versão do pod, ficam dois replicasets, um parado e outro em execução. Isso é importante para o caso de precisar fazer um rollback.

`vim primeiro-deployment.yaml`

```yaml
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

```bash
kubectl create -f primeiro_deployment.yaml
kubectl get deployments.
cp primeiro_deployment.yaml segundo_deployment.yaml
vim segundo_deployment.yaml
```

Muda o nome e o dc do label
```yaml
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

```bash
kubectl create -f segundo_deployment.yaml
kubectl get deployments.
```

## Labels

`kubectl describe deployments primeiro-deployment` para verificar as labels

`kubectl get replicasets.` para ver os nomes

`kubectl describe replicasets primeiro-deployment-<hash>`

```bash
kubectl get pods
kubectl describe pods primeiro-deployment-<hash>
```

`kubectl get pods -l dc=NL` mostra todos os pods onde a label dc é igual a "NL"

`kubectl get pods -L dc` adiciona a coluna dc

`kubectl get replicasets. -l dc=NL`

```bash
kubectl label nodes maquina-2 disk=SSD
kubectl label nodes maquina-3 disk=HDD

kubectl describe nodes maquina-2

kubectl label nodes maquina-2 --list

kubectl label nodes maquina-3 disk=SSD --overwrite
kubectl label nodes maquina-3 --list
kubectl label nodes maquina-3 disk=HDD --overwrite
```

Só atualiza com "overwrite"

```bash
kubectl label nodes maquina-3 dc=NL
kubectl label nodes maquina-2 dc=UK
```

```bash
cp primeiro_deployment.yaml terceiro_deployment.yaml
vim terceiro_deployment.yaml
```

Modifica abaixo do terminationGracePeriodSeconds

```yaml
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

```bash
kubectl create deployment -f terceiro_deployment.yaml
kubectl get deployments.
kubectl label nodes maquina-3 --list
kubectl label nodes maquina-2 --list
```

SSD aparece na maquina-2. Para conferir se subiu o terceiro deployment lá:

`kubectl get pods -o wide`

`kubectl edit deployments. terceiro-deployment` e muda SSD para HDD

```bash
kubectl get pods -o wide
kubectl describe deployments. terceiro-deployment
kubectl get replicasets.
```

`kubectl label nodes dc- --all` remove as labels dc de todos os nós

```bash
kubectl label nodes maquina-2 --list
kubectl label nodes disk- --all
kubectl label nodes maquina-2 --list
kubectl label nodes maquina-3 --list
```

Apenas para fins didáticos, não aconselhado em produção:

`vim primeiro-replicaset.yaml`

```yaml
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

```bash
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

```bash
kubectl describe rs replica-set-primeiro
kubectl get pods
kubectl delete pod replica-set-primeiro-<hash>
kubectl get pods
```

Aparece outro pod no lugar do que foi apagado, já que a função do replicaset é manter o controle dos pods

`kubectl describe rs replica-set-primeiro` para ver que um pod tem poucos segundos de existência

`kubectl edit replicasets. replica-set-primeiro`

Muda as réplicas para 4

```bash
kubectl get rs
kubectl describe rs replica-set-primeiro
kubectl get pods -L system
```

Muda a versão do nginx para 1.15.0

```bash
kubectl describe rs replica-set-primeiro
kubectl get rs
kubectl get pods -L system
```

Ainda está a versão antiga do nginx.

```bash
kubectl delete pod replica-set-primeiro-<hash>
kubectl get pods -L system
```

Outro container está sendo criado

`kubectl describe pods replica-set-primeiro-<hash>` para ver a diferença de versão do nginx entre os pods. Apenas um dos pods está com a versão certa. É por isso que devemos criar replicasets pelo deployment. Ele teria atualizado todas as réplicas.

`kubectl delete -f primeiro-replicaset.yaml` para remover todos ao mesmo tempo

```bash
kubectl delete -f primeiro_deployment.yaml
kubectl delete -f segundo_deployment.yaml
kubectl delete -f terceiro_deployment.yaml
```

## Daemonset

Daemonset é praticamente um replicaset, mas não se escolhe quantas réplicas dele vai ter. Ele garante que vai rodar um pod em cada um dos nós.

`vim primeiro-daemonset.yaml`

```yaml
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

```bash
kubectl create -f primeiro-daemonset.yaml
kubectl get damonsets # ou ds
get pods -o wide
```

Não sobe no master por causa do taint

Para tirar o taint da maquina-1: `kubectl taint node maquina-1 node-role.kubernetes.io/master-`. Mas isso não é recomendado. Para colocar de volta: `kubectl taint node maquina-1 node-role.kubernetes.io/master-=value1:NoSchedule`

```bash
kubectl set image ds daemon-set-primeiro nginx=nginx:1.15.0
kubectl get pods -o wide

kubectl describe ds daemon-set-primeiro # 1.15.0
kubectl describe pod daemon-set-primeiro-<hash> # 1.7.9

kubectl delete pods daemon-set-primeiro-<hash> # cria outro depois que apaga
```

## Rollouts e Rollbacks 

```bash
kubectl rollout history daemonset daemon-set-primeiro
kubectl rollout history daemonset daemon-set-primeiro --revision=1
kubectl rollout history daemonset daemon-set-primeiro --revision=2

kubectl rollout history undo daemonset daemon-set-primeiro --to-revision=1

kubectl describe pods daemon-set-primeiro-<hash> | grep Image
```

O comando que tem `undo` serve para voltar para a primeira versão, mas não volta porque não estava definido no yaml. Adiciona no primeiro nível, depois do template:

```yaml
updateStrategy:
  type: RollingUpdate
```

`kubectl describe ds daemon-set-primeiro` para ver quantos mós estáo atualizados de acordo com a versão corrente.

`kubectl get daemonsets` mostra 2

`kubectl get pods -o wide` mostra 3 pods

Para ver funcionando com o updateStrategy:

```bash
kubectl delete -f primeiro-daemonset.yaml

kubectl create -f primeiro-daemonset.yaml

kubectl get daemonsets
kubectl describe ds daemon-set-primeiro
kubectl get daemonsets daemon-set-primeiro -o yaml

kubectl set image ds daemon-set-primeiro nginx=nginx:1.15.0

kubectl get daemonsets
kubectl pods -o wide
```

Ele já atualiza os pods no momento da edição. 

```bash
kubectl rollout history daemonset daemon-set-primeiro
kubectl rollout history daemonset daemon-set-primeiro --revision=1
kubectl rollout history daemonset daemon-set-primeiro --revision=2

kubectl rollout history undo daemonset daemon-set-primeiro --to-revision=1

kubectl get pods -o wide

kubectl delete daemonsets daemon-set-primeiro
```

## Canary Deploy

```bash
git clone https://github.com/badtuxx/k8s-canary-deploy-example.git
cd k8s-canary-deploy-example/
vim roles/create-cluster/tasks/init-cluster.yml

cp deploy-app-v1-playbook/roles/common/files/* .
vim app-v1.yml

kubectl create -f app-v1.yml
kubectl get deploy


vim service-app.yml
kubectl create -f service-app.yml
kubectl get services
kubectl get pods -o wide

cp k8s-canary-deploy-example/canary-deploy-app-v2-playbook/roles/common/files/* .
vim app-v2-canary.yml
kubectl create -f app-v2-canary.yml
kubectl get deploy
kubectl get pods -o wide
```

Como o selector app do service é giropops e o label app dos deployments também são giropops, é possível rodar duas versões da aplicação no mesmo service.

`curl http://<ip>:32222` algumas vezes para ver a versão mudar. A versão 2 aparece menos, por estar definida para apenas 10%.

```bash
cp k8s-canary-deploy-example/deploy-app-v2-playbook/roles/common/files/* .
vim app-v2.yml
kubectl apply -f app-v2.yml # porque já existe o giropops
kubectl get pods -o wide
```

`curl http://<ip>:32222` algumas vezes para ver que agora está balanceado.

```bash
kubectl scale deployment --replicas=3 giropops-v1
kubectl get pods -o wide
kubectl scale deployment --replicas=1 giropops-v1
kubectl get pods -o wide
```

`curl http://<ip>:32222` algumas vezes para ver que dificilmente aparece a versão 1.

```bash
kubectl delete deployments. giropops-v1
```

`curl http://<ip>:32222` agora só traz a versão 2.

```bash
kubectl rollout history deployment giropops-v2
kubectl rollout history deployment giropops-v2 --revision=1
kubectl rollout history deployment giropops-v2 --revision=2
```

### RollingUpdateStrategy

`vim app-v2.yml`

abaixo de replicas, adiciona:

```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
```

maxSurge: durante o processo de atualização, passa no máximo 2 pods do máximo definido. Se forem 10 pods, pode chegar em 12, para que "desligue" pods e já atualize, não perdendo o valor original.

maxUnavailable: durante o processo de atualização, esse é o número máximo de réplicas que podem ser "desligadas" ao mesmo tempo.

Ao invés do número, também é possível colocar a porcentagem.

```bash
kubectl apply -f app-v2.yml
kubectl get pods -o wide

kubectl delete -f app-v2.yml
kubectl get pods -o wide

kubectl edit deployment giropops-v2
```

muda a versão para 1.0.0 em todas as flags. Ele vai usar a estratégia definida.

`kubectl rollout status deployment giropops-v2` para ver o status

```bash
kubectl delete -f app-v2.yml
vim app-v2.yml
```

```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 5
```

Muda os valores para 1 e 5, para que seja mais rápido.

```bash
kubectl apply -f app-v2.yml
kubectl edit deployments giropops-v2
```

muda a versão para 1.0.0

`kubectl rollout status deployment giropops-v2`

### Comandos

```bash
# kubectl rollout history ds daemon-set-primeiro

# kubectl rollout history ds daemon-set-primeiro --revision=1

# kubectl rollout history ds daemon-set-primeiro --revision=2

# kubectl rollout undo ds daemon-set-primeiro --to-revision=1

# kubectl rollout status ds daemon-set-primeiro 

# kubectl describe daemon-set-primeiro-hp4qc | grep -i image:

# kubectl delete -f primeiro-daemonset.yaml
```

```yaml
# vim primeiro-daemonset.yaml

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: daemon-set-primeiro
spec:
  template:
    metadata:
      labels:
        system: DaemonOne
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
  updateStrategy:
    type: RollingUpdate
    
    
# kubectl create -f primeiro-daemonset.yaml

# kubectl get daemonset

# kubectl describe ds daemon-set-primeiro

# kubectl get ds daemon-set-primeiro -o yaml | grep -A 2 Strategy

# kubectl set image ds daemon-set-primeiro nginx=nginx:1.15.0

# kubectl get daemonset

# kubectl get pods -o wide

# kubectl describe ds daemon-set-primeiro

# kubectl rollout history ds daemon-set-primeiro

# kubectl rollout history ds daemon-set-primeiro --revision=1

# kubectl rollout history ds daemon-set-primeiro --revision=2

# kubectl rollout undo ds daemon-set-primeiro --to-revision=1

# kubectl rollout status ds daemon-set-primeiro

# kubectl delete ds daemon-set-primeiro
```
