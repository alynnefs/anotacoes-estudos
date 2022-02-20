# Kubernetes - aula 6

## EKS

Binário que vai interpretar os Cloud Formation já existentes.

```yaml
# vim cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: LINUXtips-labs
  region: eu-central-1
  
nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 2
  - name: ng-2
    instanceType: m5.xlarge
    desiredCapacity: 2
```

Tem que ter o config e o credentials da AWS.

Cria usuário com permissões: S3, EKSClusterPolicy, etc.

```bash
# TO-DO procurar instalação
# mover pro local/bin
eksctl create cluster -f cluster.yaml

kubectl get nodes
kubectl get pods -A
kubectl get svc
kubectl get svc -A
```

## External DNS

Route 53 é um serviço da AWS que gerencia esquemas de DNS

```bash
aws route53 create-hosted-zone --name "eks.strigus.com.br." --caller-reference "external-dns-test-$(date +%s)"
```

IAM Policy é onde você consegue listar as hosted zones que estão na AWS. Exemplo de json:

iam_policy.json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSet"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

Copia o json acima. Na AWS: IAM > Policies > Create Policy > JSON e cola > next e adiciona o nome

```bash
eksctl utils associate-iam-oidc-provider --region=eu-central-1 --cluster-LINUXtips-lab --approve

eksctl create iamserviceaccount --name external-dns --namespace default --cluster LINUXtips-lab --attach-policy-arn <policy_arn_da_aws> --approve

kubectl get sa # serviceaccount
kubectl describe sa external-dns
```


```yaml
# vim dns-external.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  annotations:
    # substituir
    eks.amazonaws.com/role-arn: arn:aws:iam::371371653782:role/giropops
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
      annotations:
        iam.amazonaws.com/role: arn:aws:iam::371371653782:role/giropops
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.6
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=eks.strigus.com.br
        - --provider=aws
        - --policy=upsert-only
        - --aws-zone-type=public
        - --registry=txt
        - --txt-owner-id=<id>
      securityContext:
        fsGroup: 65534
```

```bash
kubectl create -f dns-external.yaml

kubectl get deploy
kubectl get pods
kubectl logs -f external-dns-<hash>
```

## Cert-manager

Segue a instalação via helm.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

```yaml
# vim cert-manager-values.yml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::371371653782:role/giropops
    
installCRDs: true

securityContext:
  enabled: true
  fsGroup: 1001
```

```bash
helm upgrade cert-manager jetstack/cert-manager --install --namespace cert-manager --create-namespace --values "cert-manager-values.yml" --wait
```

```yaml
# vim eks-dns-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
  external-dns.alpha.kubernetes.io/hostname: nginx.eks.strigus.com.br
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app:nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: http
```

```bash
kubectl create -f eks-dns-service.yaml

kubectl get deploy
kubectl get svc
```

Confere na AWS se apareceu um Record. Acessa no navegador `http://nginx.eks.strigus.com.br` para ver que deu certo. Não vai ter https por não ter criado um cert issuer

```yaml
# vim cert-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: email@email.com
    privateKeySecretRef:
      name: letsencrypt
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
        route53:
          region: eu-central-1
          hostedZoneID: <id>
```

```bash
kubectl create -f cert-issuer.yaml

kubectl get pods
kubectl get certificate
kubectl get clusterissuers.cert-manager.io
kubectl get pods -A
```

## Deploy do Gitlab usando o Helm 

```bash
helm repo add gitlab https://charts.gitlab.io/
helm repo update
helm upgrade --install gitlab gitlab/gitlab --timeout 600s --set global.hosts.domain=eks.strigus.com.br
helm uninstall cert-manager -n cert-manager
helm upgrade --install gitlab gitlab/gitlab --timeout 600s --set global.hosts.domain=eks.strigus.com.br --set certmanager-issuer.email=email@email.com

kubectl get deploy
kubectl get pods
kubectl get certificates
kubectl describe certificates gitlab-gitlab-tls
kubectl get CertificateRequest
kubectl describe CertificateRequest gitlab-gitlab-tls-dckmg
```

Agora já é possível acessar a URL. No navegador: `gitlab.eks.strigus.com.br/users/sign_in`

```bash
kubectl get pods
kubectl get secrets

kubectl get secrets <name>-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo

kubectl get pv # criou automaticamente
kubectl get pvc # criou automaticamente
```

