# Kubernetes - aula 5

## Ingress

É um conjunto de regras para abstrair os services, fazendo com que sejam expostos para além dos clusters sem NodePort ou LoadBalancer.

```yaml
# vim app1.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: dockersamples/static-site
        env:
        - name: AUTHOR
          value: GIROPOPS
        ports:
        - containerPort: 80
```

```bash
cp app1.yaml app2.yaml
```

```yaml
# vim app2.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: dockersamples/static-site
        env:
        - name: AUTHOR
          value: STRIGUS
        ports:
        - containerPort: 80
```

```bash
kubectl create -f app1.yaml
kubectl create -f app2.yaml
kubectl get deployments
kubectl get pods
```

```yaml
# vim svc-app1.yaml
apiVersion: v1
kind: Service
metadata:
  name: appsvc1
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: app1
```

```bash
cp svc-app1.yaml svc-app2.yaml
```

```yaml
# vim svc-app2.yaml
apiVersion: v1
kind: Service
metadata:
  name: appsvc2
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: app2
```

```bash
kubectl create -f svc-app1.yaml
kubectl create -f svc-app2.yaml
kubectl get deploy,svc,po
kubectl get ep # endpoints

curl <endpoint_svc1>
curl <endpoint_svc2>
```

```yaml
# vim default-backend.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-backend
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: default-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-backend
        image: gcr.io/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
```

Liveness Probe verifica se está em pé. Se não estiver, vai reiniciar o container. Readiness Probe verifica quando o container está pronto para receber as definições.

Initia Delay Seconds: tempo que aguarda até que faça a primeira checagem do liveness probe

```bash
kubectl create namespace ingress
kubectl create -f default-backend.yaml -n ingress
```

```yaml
# vim default-backend-service.yaml
apiVersion: extensions/v1beta1
kind: Service
metadata:
  name: default-backend
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: default-backend
```

```bash
kubectl create -f default-backend-service.yaml -n ingress

kubectl get deployments.
kubectl get deployments. -n ingress
kubectl get services
kubectl get services -n ingress
kubectl get ep
kubectl get ep -n ingress

curl <ip>:<port> # default-backend - 404
```

```yaml
# vim nginx-ingress-controller-config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller-conf
  labels:
    app: nginx-ingress-lb
data:
  enable-vts-status: 'true'
```

VTS (Vhost Traffic Status) traz detalhes sobre o nginx, como está sendo o acesso, etc

```bash
kubectl create -f nginx-ingress-controller-config-map.yaml -n ingress

kubectl get configmaps -n ingress
kubectl describe configmaps -n ingress nginx-ingress-controller-conf
```

Criar um service account, que é um usuário para "bater" na API, e um cluster role, que são regras para definir permissões do usuário.

```yaml
# vim nginx-ingress-controller-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx
  namespace: ingress
```

```yaml
# vim nginx-ingress-controller-clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-role
rules:
- apiGroups:
  - ""
  - "extensions"
  resources:
  - configmaps
  - secrets
  - endpoints
  - ingresses
  - nodes
  - pods
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - watch
  - get
  - update
- apiGroups:
  - "extensions"
  resources:
  - ingresses
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
- apiGroups:
  - "extensions"
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
```

Aspas vazias: v1

precisa criar um cluster role binding para associar um cluster role com um usuário

```yaml
# vim nginx-ingress-controller-clusterrolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-role
  namespace: ingress
roleRef:
  apiGroup: rbac.authorization.k8s.io/v1beta1
  kind: ClusterRole
  name: nginx-role
subjects:
- kind: ServiceAccount
  name: nginx
  namespace: ingress
```

roleRef: a qual role ele se referencia
subjects: com quem vai associar

```bash
kubectl create -f nginx-ingress-controller-service-account.yaml
kubectl create -f nginx-ingress-controller-clusterrole.yaml
kubectl create -f nginx-ingress-controller-clusterrolebinding.yaml

kubectl get serviceaccounts -n ingress
kubectl get clusterrole -n ingress
kubectl get clusterrolebindings.rbac.authorization.k8s.io -n ingress
```

Tudo isso será usado pelo nginx para criar rotas, estabelecer conexões e expor os services. O deployment a  seguir é do nginx ingress controller

```yaml
# vim nginx-ingress-controller-deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  revisionHistoryLimit: 3
  template:
    metadata:
      labels:
       app: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      serviceAccount: nginx
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.9.0
          imagePullPolicy: Always
          readinessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
          livenessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 5
          args:
            - /nginx-ingress-controller
            - --default-backend-service=ingress/default-backend
            - --configmap=ingress/nginx-ingress-controller-conf
            - v=2
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 80
            - containerPort: 18080
```

```bash
kubectl create -f nginx-ingress-controller-deployment.yaml -n ingress

kubectl get deployments. -n ingress
kubectl get pods --all-namespaces
```

A partir do momento que o deployment foi feito, é como se tivesse extendido o kubernetes, foi dado um novo componente a ele. A partir de agora é possível criar objetos do tipo ingress. Para criar uma regra de ingress:

```yaml
# vim nginx-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  # mude para seu endereço
  - host: ec2-18-212-4-47.compute-1.amazonaws.com
    http:
      paths:
      - backend:
          serviceName: nginx-ingress
          servicePort: 18080
        path: /nginx_status
```

```yaml
# vim app-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: app-ingress
spec:
  rules:
  # mude para seu endereço
  - host: ec2-18-212-4-47.compute-1.amazonaws.com
    http:
      paths:
      - backend:
          serviceName: appsvc1
          servicePort: 80
        path: /app1
      - backend:
          serviceName: appsvc2
          servicePort: 80
        path: /app2
```

```bash
kubectl create -f nginx-ingress.yaml -n ingress # porque o deployment tá no ingress
kubectl create -f app-ingress.yaml

kubectl get ingresses.
kubectl get ingresses. -n ingress
kubectl describe ingresses. nginx-ingress -n ingress # verificar rules
kubectl describe ingresses. app-ingress -n ingress # verificar rules
```

Para criar o service:

```yaml
# vim nginx-ingress-controller-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30000
    name: http
  - port: 18080
    nodePort: 32000
    name: http-mgmt
  selector:
    app: nginx-ingress-lb
```

```bash
kubectl create -f nginx-ingress-controller-service.yaml -n ingress

kubectl get service -n ingress
```

No navegador, coloque o endereço da AWS e a porta 30000. De acordo com os exemplos: `ec2-18-212-4-47.compute-1.amazonaws.com:30000`. O retorno será o default backend. `ec2-18-212-4-47.compute-1.amazonaws.com:30000/app1` retorna o Hello Giropops. Já o `ec2-18-212-4-47.compute-1.amazonaws.com:30000/app2` retorna o Hello Strigus.

Acessando `ec2-18-212-4-47.compute-1.amazonaws.com:32000/nginx_status` retorna o status do VTS. Para ver mudar a quantidade de acesso com status 200, coloque no terminal: `curl http://ec2-18-212-4-47.compute-1.amazonaws.com:30000/app2` e `curl http://ec2-18-212-4-47.compute-1.amazonaws.com:30000/app1`. O valor no Response 2xx deverá ter mudado.

### Load Balancer

Na AWS: Load Balancers > HTTP/HTTPS
- nome: giropops
- scheme: internet-facing
- IP address type: ipv4
- Protocol: HTTP
- Port: 80
- Availability Zones: us-east-1a, us-east-1b, us-east-1d, us-east-1e

Security group: launch-wizard-126 (no exemplo)

Target group:
- Name: giropops
- Protocol: HTTP
- Port: 30000

Adiciona os 3 nós, revisa e cria. Copia o DNS name e cola no host do app-ingress.yaml

```bash
kubectl delete -f app-ingress.yaml
kubectl create -f app-ingress.yaml

kubectl get ingress
kubectl describe ingress app-ingress
```

No navegador: `<DNS_load_balancer>/app1` e `<DNS_load_balancer>/app2`
