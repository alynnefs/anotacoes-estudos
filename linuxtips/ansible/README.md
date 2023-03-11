# Ansible

## Aula 1

### Criando instâncias e instalando o Ansible

https://docs.ansible.com/ansible/latest/installation_guide/index.html

Criar 3 `t2.small` na AWS. Uma para o Ansible e duas para webserver

Máquina 1 (root)

```
mkdir descomplicando-o-ansible
cd descomplicando-o-ansible
cp ../Downloads/ansible-class.pem .
chmod 400 ansible-class.pem
ssh -i "ansible-class.pem" ubuntu@<root>
```

Máquina 2 (elliot-01)

```
ssh -i "ansible-class.pem" ubuntu@<elliot-01>
```

Máquina 3

```
ssh -i "ansible-class.pem" ubuntu@<elliot-02>
```

Máquina 1, como usuário comum

```
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible

cat /etc/apt/sources.list.d/ansible-ubuntu-ansible-bionic.list

ansible --version

ls /etc/ansible/roles

touch hosts

vim /etc/hosts # adiciona o ip das máquinas 2 e 3 no formato
# ip nome
# x.x.x.x elliot-0X
```

Adiciona o security group para que as máquinas se comuniquem.

### Setup inicial e primeiro ping

`scp -i ansible-class.pem ansible-class.pem <maquina1>:/tmp`

```
mv ansible-class.pem /home/ubuntu/descompliando-o-ansible
ssh -i <chave> <ip_elliot-01>
ssh -i <chave> <ip_elliot-02>

ssh-keygen
cd /home/seu_user/.ssh
ssh-copy-id id_rsa <elliot-01>

ssh -i ansible-class.pem <ip_elliot-01>
ssh-agent bash
ssh-add ansible-class.pem
ssh <elliot-01>
```

`vim hosts`

```

# ip maquina 2
# ip maquina 3
# ou
elliot-01
elliot-02
```

```
ansible -i hosts all -m ping
```

Lembre-se de instalar o python nas máquinas-alvo

### Brincando com o Ansible, seus módulos e comandos ad-hoc

https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html#intro-adhoc

```
ansible -i hosts <maquina2> -m apt -a "name=cmatrix state=present -b"
# -b é de become user, ou root
```

Testa o `cmatrix` no elliot-01

```
ansible -i hosts <elliot-02> -m apt -a "update_cache=yes name=cmatrix state=present -b -k"
# -k serve para pedir a senha
```

```
ansible -i hosts <elliot-01> -m shell -a "ifconfig eth0"

ansible -i hosts <elliot-02> -m copy -a "src=giropops dest=/tmp"

ansible -i hosts <elliot-02> -m git -a "repo=https://github.com/badtuxx/giropops-monitoring.git dest=/home/ubuntu/teste-git"

ansible -i hosts <elliot-01> -m setup # traz todos os facts

ansible -i hosts all -m setup -a "filter=ansible_distribution"

ansible -i hosts all -m setup -a "filter=ansible_nodename"
```

### Criando o primeiro playbook

`vim hosts`

```
[webservers]
elliot-01

[db]
elliot-02
```

`ansible -i hosts webservers -m ping`

`vim meu_primeiro_playbook.yml`

```
---
- hosts: webservers
  become: yes
  remote_user: ubuntu
  tasks:
  - name: Instalando o Nginx
    apt: latest
    update_cache: yes
```

`ansible-playbook -i hosts meu_primeiro_playbook.yml`

```
---
- hosts: webservers
  become: yes
  remote_user: ubuntu
  tasks:
  - name: Instalando o Nginx
    apt: latest
    update_cache: yes

  - name: Habilitando o start do Nginx no boot
    service:
      name: nginx
      enabled: yes

  - name: Iniciando o Nginx
    service:
      name: nginx
      state: started
```

`ansible-playbook -i hosts meu_primeiro_playbook.yml`

Elliot-01:

`vim /var/www/html/index.nginx-debian.html`

"giropops strigus girus"

mrRobot

`vim ~/descomplicando-o-ansible/index.html`

```
DESCOMPLICANDO O ANSIBLE
GIROPOPS STRIGUS GIRUS
```

```
---
- hosts: webservers
  become: yes
  remote_user: ubuntu
  tasks:
  - name: Instalando o Nginx
    apt: latest
    update_cache: yes

  - name: Habilitando o start do Nginx no boot
    service:
      name: nginx
      enabled: yes

  - name: Iniciando o Nginx
    service:
      name: nginx
      state: started
  - name: Copiando html personalizado
    copy:
      src: index.html
      dest: /var/www/html/index.html
```

`ansible-playbook -i hosts meu_primeiro_playbook.yml`

Elliot-01

`cat /var/www/html/index.nginx-debian.html`

```
<html>
<head>
  <title>GIROPOPS STRIGUS GIRUS!</title>
<body>
  <h1>DESCOMPLICANDO O ANSIBLE!</h1>
</body>
</head>
</html>
```

Copia o `/etc/nginx/nginx.conf` para `~/descomplicando-o-ansible/nginx.conf` e adiciona um comentário na primeira linha

```
---
- hosts: webservers
  become: yes
  remote_user: ubuntu
  tasks:
  - name: Instalando o Nginx
    apt: latest
    update_cache: yes

  - name: Habilitando o start do Nginx no boot
    service:
      name: nginx
      enabled: yes

  - name: Iniciando o Nginx
    service:
      name: nginx
      state: started

  - name: Copiando html personalizado
    copy:
      src: index.html
      dest: /var/www/html/index.html

  - name: Copiando nginx.conf
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf

  handlers:
  - name: Restartando o Nginx
    service:
      name: nginx
      state: restarted
```

# Dia 2

### Criando o primeiro template

```
<!-- index.html -->
<html>
<head>
  <title>Giropops Strigus Girus</title>
<body>
  <h1>DESCOMPLICANDO O ANSIBLE - 2022</h1>
  <h2>Esse aqui é o meu IP: {{ ansible_facts.eth0.ipv4.address }}</h2>
</body>
</head>
</html>
```

```
# playbook
---
- hosts: webservers
  become: yes
  remote_user: ubuntu
  tasks:
  - name: Instalando o Nginx
    apt: latest
    update_cache: yes

  - name: Habilitando o start do Nginx no boot
    service:
      name: nginx
      enabled: yes

  - name: Iniciando o Nginx
    service:
      name: nginx
      state: started

  - name: Copiando html personalizado
    template:
      src: index.html.j2
      dest: /var/www/html/index.html

  - name: Copiando nginx.conf
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf

  handlers:
  - name: Restartando o Nginx
    service:
      name: nginx
      state: restarted
```

```
[webservers]
elliot-01
elliot-02
```

`mv index.html index.html.j2`

### Projeto Ansible + AWS + Kubernetes


```
mv projeto projeto-k8s-cluster
cd projeto-k8s-cluster
mkdir provisioning
mkdir install_k8s
mkdir deploy_app
mkdir extra
vim README.md
```

```
# Treinamento Descomplicando o Ansible

Projeto do treinamento para a instalação de um cluster Kubernetes utilizando o Ansible + AWS.

## Fases do projeto
- Provisioning => Criar as instâncias/vms para o nosso cluster
- Install_k8s => Criação do cluster, etc
- Deploy_app => Deploy de uma aplicação exemplo
- Extra
```

```
cd provisioning
touch hosts
touch main.yml

mkdir roles && cd roles
ansible-galaxy init criando-instancias
```

### Provisioning - Criando o nosso primeiro playbook do projeto

```
# main.yml
- hosts: local
  roles:
  - criando-instancias
```

```
# vim hosts
[local]
localhost ansible_connection=local ansible_python_interpreter=python gather_facts=false

[kubernetes]
```

```
cd roles/criando-instancias/tasks
vim main.yml
```

```
---
# tasks file for criando-instancias
- include: provisioning.yml
```

```
# vim provisioning.yml
- name: Criando o Security Group
  local_action:
    module: ec2_group
    name: "{{ sec_group_name }}"
    description: sg giropops
    profile: "{{ profile }}"
    region: {{ region }}
    rules:
    - proto: tcp
      from_port: 22
      to_port: 22
      cidr_ip: 0.0.0.0/0
      rule_desc: SSH
    rules_egress:
    - proto: all
      cidr_ip: 0.0.0.0/0
  register: basic_firewall
  
- name: Criando a instancia EC2
  local_action: ec2
    group={{ sec_group_name }}
    instance_type={{ instance_type }}
    image={{ image }}
    profile={{ profile }}
    wait=true
    region={{ region }}
    key_pair={{ keypair }}
    count={{ count }}
  register: ec2
    
- name: Adicionando a instancia ao inventario temp
  add_host: name={{ item.public_ip }} groups=giropops-new
  with_items: "{{ ec2.instances }}"
  
- name: Adicionando o IP publico da instancia criada ao arquivo hosts
  local_action: lineinfile
    dest="./hosts"
    regexp={{ item.public_ip }}
    insertafter="[kubernetes]" line={{ item.public_ip }}
  with_items: "{{ ec2.instances }}"
  
- name: Adicionando o IP privado da instancia criada ao arquivo hosts
  local_action: lineinfile
    dest="./hosts"
    regexp={{ item.private_ip }}
    insertafter="[kubernetes]" line={{ item.private_ip }}
  with_items: "{{ ec2.instances }}"
  
- name: Esperando o SSH
  local_action: wait_for
    host={{ item.public_ip }}
    port=22
    state=started
  with_items: "{{ ec2.instances }}"
  
- name: Adicionando uma tag na instancia
  local_action: ec2_tag resource={{ item.id }} region={{ region }} profile={{ profile }} state=present
  with_items: "{{ ec2.instances }}"
  args:
    tags:
      Name: ansible-{{ item.ami_launch_index |int +1 }}
```

```
cd ../vars
vim main.yml
```

```
---
# vars file for criando-intancias
instance_type: t2.medium
sec_group_name: giropops
image: ami-<id_ubuntu>
keypair: ansible-class
region: eu-central-1
count: 3
profile: giropops
```

### Provisioning - Executando o nosso primeiro playbook do projeto

Cria um usuário com o nome `ansible-class` e marca o `acesso programático`. Adiciona no grupo admin e pega a senha para fazer o export.

```
aws configure

pip install boto

cd ~/descomplicando-o-ansible-projeto-k8s-cluster/provisioning
ansible-playbook -i hosts main.yml

ssh -i "ansible-class.pem <maquina-nova>"
```

Remove os ips do hosts

#### Links importantes:

##### Ansible + AWS

https://www.ansible.com/integrations/cloud/amazon-web-services

https://docs.ansible.com/ansible/latest/scenario_guides/guide_aws.html

https://docs.ansible.com/ansible/latest/modules/list_of_all_modules.html

https://docs.ansible.com/ansible/latest/modules/list_of_cloud_modules.html

https://docs.ansible.com/ansible/latest/modules/ec2_module.html

https://docs.ansible.com/ansible/latest/modules/ec2_group_module.html

https://galaxy.ansible.com/docs/contributing/creating_role.html


##### Para criar readme.md

https://www.makeareadme.com/

## Dia 3

### Install k8s - criando o nosso playbook

Limpa o "kubernetes" do arquivo hosts para criar de novo.

```
cd install_k8s
vim hosts
```

```
[k8s-master]


[k8s-workers]

[k8s-workers:vars]
K8S_MASTER_NODE_IP=
K8S_API_SECURE_PORT=6443
```

```
# vim main.yml
- hosts: all
  become: yes
  user: ubuntu
  gather_gacts: no
  pre_tasks:
  - name: 'Atualizando o repo'
    raw: 'apt-get update'
  - name: 'Intalando o Python'
    raw: 'apt-get install -y python'
  roles:
  - { role: install-k8s, tags: ["install_k8s_role"] }

- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - { role: create-cluster, tags: ["create_cluster_role"] }

- hosts: k8s-workers
  become: yes
  user: ubuntu
  roles:
  - { role: join-workers, tags: ["join_workers_role"] }
```

```
ansible-galaxy init install-k8s
ansible-galaxy init create-cluster
ansible-galaxy init join-workers
```

### Install_k8s - Criando a role para instalar Docker e K8s

```
cd install-k8s/tasks
vim main.yml
```

```
---
# tasks file for install-k8s
- include: install.yml
```

```
# vim install.yml
- name: Instalando o Docker
  shell: curl -fsSL https://get.docker.com | bash

- name: Adicionando as chaves do repo apt do k8s
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Adicionando o repo do k8s
  apt_repository:
    repo: deb https://apt.kubernetes.io/ kubernetes-xenial main
    state: present

- name: Instalando os pacotes kubeadm, kubelet e kubectl
  apt:
    name: "{{ packages }}"
  vars:
    packages:
    - kubelet
    - kubeadm
    - kubectl
```

### Install_k8s - Criando o nosso cluster k8s

```
cd ../../create_cluster/tasks
vim main.yml
```

```
---
# tasks file for create-cluster
- include: init-cluster.yml
```

```
# vim init-cluster.yml
# forçar reset no cluster
- name: Removendo cluster antigo
  command:
    kubeadm reset --force
  register: kubeadm_reset

- name: Inicializando o cluster k8s
  command:
    kubeadm init
  register: kubeadm_init

- name: Criando o diretorio .kube
  file:
    path: ~/.kube
    state: directory

- name: Linkando o arquivo admin.conf para o ~/.kube/config
  file:
    src: /etc/kubernetes/admin.conf
    dst: ~/.kube/config
  state: link

- name: Configurando o pod network Weavenet
  shell: kubectl apply -f {{ default_url_weavenet }}
  register: weavenet_result

- name: Pegando o token para adicionar os workers no cluster
  shell: kubeadm token list | cut -d ' ' -f1 | sed -n '2p'
  register: k8s_token

- name: CA Hash
  shell: openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
  register: k8s_master_ca_hash

- name: Adicionando o token e o hash em um dummy host
  add_host:
    name: "K8S_TOKEN_HOLDER"
    token: "{{ k8s_token.stdout }}"
    hash: "{{ k8s_master_ca_hash.stdout }}"

- name:
  degub:
    msg: "[MASTER] K8S_TOKEN_HOLDER - O token e {{ hostvars['K8S_TOKEN_HOLDER']['token'] }}"

- name:
  degub:
    msg: "[MASTER] K8S_TOKEN_HOLDER - O hash e {{ hostvars['K8S_TOKEN_HOLDER']['hash'] }}"
```

```
# cd ../vars
# vim main.yml
```

```
---
# vars file for create-cluster
default_url_weavenet: "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

### Install_k8s - Adicionando outros nodes ao nosso cluster k8s

```
cd ../../join-workers/tasks
vim main.yml
```

```
---
# tasks file for join-workers
- include: join-cluster.yml
```

```
# vim join-cluster.yml
- name:
  debug:
    msg: "[WORKER] K8S_TOKEN_HOLDER - O token e {{ hostvars['K8S_TOKEN_HOLDER']['token'] }}"

- name:
  debug:
    msg: "[WORKER] K8S_TOKEN_HOLDER - O hash e {{ hostvars['K8S_TOKEN_HOLDER']['hash'] }}"

- name: Removendo o cluster
  command:
    kubeadm reset --force
  register: kubeadm_reset

- name: Adicionando o worker ao cluster k8s
  shell:
    kubeadm join --token={{ hostvars['K8S_TOKEN_HOLDER']['token'] }}
    --discovery-token-ca-cert-hash sha256:{{ hostvars['K8S_TOKEN_HOLDER']['hash'] }}
    {{K8S_MASTER_NODE_IP}}:{{K8S_API_SECURE_PORT}}
```

### Install_k8s - Executando o nosso playbook

Coloca os IPs do provisioning no hosts do projeto-k8s-cluster [k8s-master] e [k8s-workers].

No install_k8s:


```
ssh-agent zsh
ssh-add ../../ansible-class.pem

ansible-playbook -i hosts main.yml
cat hosts

ssh ubuntu@ip-maquina1
sudo su -
kubectl get nodes
```

```
# vim deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 50
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      container:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

```
kubectl apply -f deploy-nginx.yaml
kubectl get deploy
kubectl get deploy nginx
kubectl delete deploy nginx-deployment
```

```
cd provisioning
ansible-playbook -i hosts main.yml

cat ../../../.aws/credentials
# vim no arquivo e deixa o default no giropops
```

## Dia 4

```
cd install_k8s/roles
ansible-galaxy init install-helm
vim roles/install-helm/tasks/main.yml
```

```
---
# tasks file for install-helm
- include: install-helm.yml
```

```
# vim roles/install-helm/tasks/install-helm.yml
---
- name: Download helm
  get_url:
    url: "{{ helm_url }}"
    dest: /tmp/get_helm.sh
    mode: 0775
  ignore_errors: true
  register: download

- name: Instalando helm
  shell:
    /tmp/get_helm.sh
  when:
    - download.failed|bool == false
  register: install_helm

- name:
  debug: var=install_helm
```

```
# vim roles/install-helm/vars/main.yml
---
# vars file for install-helm
helm_url: https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
```

```
# vim install_k8s/main.yml
- hosts: all
  become: yes
  user: ubuntu
  gather_gacts: no
  pre_tasks:
  - name: 'Atualizando o repo'
    raw: 'apt-get update'
  - name: 'Intalando o Python'
    raw: 'apt-get install -y python'
  roles:
  - { role: install-k8s, tags: ["install_k8s_role"] }

- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - { role: create-cluster, tags: ["create_cluster_role"] }

- hosts: k8s-workers
  become: yes
  user: ubuntu
  roles:
  - { role: join-workers, tags: ["join_workers_role"] }

- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - { role: install-helm, tags: ["install_helm3_role"]}
  
- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - { role: install-monit-tools, tags: ["install_monit_tools_role"]}
```

### Criando Role para instalar o Prometheus Operator


```
cd roles
ansible-galaxy init install-monit-tools
vim  install-monit-tools/tasks/main.yml
```

```
---
# tasks file for intall-monit-tools
- include: install-monit-tools.yml
```

```
# vim install-monit-tools.yml
- name: helm add repo
  shell: helm repo add stable {{ url_repo_helm }}
  register: prometheus_add_repo

- name: helm update
  shell: helm repo update
  register: prometheus_repo_update

- name: Instalando o Prometheus Operator
  shell: helm install {{ deploy_prometheus }}
  register: prometheus_install
```

```
# vim roles/install-monit-tools/vars/main.yml
---
# vars file for install-monit-tools
url_repo_helm: https://kubernetes-charts.storage.googleapis.com
deploy_prometheus: "--set prometheusOperator.createCustomResource=false meu-querido-prometeus stable/prometheus-operator"
```

```
ansible-playbook -i hosts main.yml --tags  "install_helm3_role, install_monit_tools_role" --list-tasks
# --skip-tags não executa as tags

ssh ubuntu@ip_master
sudo su -

kubectl get nodes
kubectl get pods
kubectl get svc
kubectl edit svc meu-querido-prometheus-grafana
kubectl get svc
```

Muda o type de ClusterIp para NodePort. No navegador, entra em `<ip>:<porta>/login`

```
kubectl create deployment nginx --image nginx
kubectl scale deploy --replicas 50 nginx
kubectl get deploy

kubectl delete deploy nginx
```

### Criando um novo playbook para o deploy da App Giropops

```
mkdir deploy-app-v1
cd deploy-app-v1
cp ../install_k8s/hosts .
vim main.yml
```

```
- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - { role: deploy-app, tags: ["deploy_app_role"] }
```

```
mkdir roles && cd roles
ansible-galaxy init deploy-app
cd ..
vim roles/deploy-app/tasks/main.yml
```

```
---
# tasks file for deploy-app
- include: deploy-app.yml
```

```
# vim roles/deploy-app/tasks/deploy-app.yml
- name: Criando o diretorio da app Giropops
  file: path={{item}} state=directory
  with_items:
    - /opt/giropops
    - /opt/giropops/logs
    - /opt/giropops/conf
  register: criando_diretorios

- name: Copiando o arquivo de deployment da app para o host
  template:
    src: app-v1.yml.j2
    dest: /opt/giropops/app-v1.yml
    owner: root
    group: root
    mode: 0644
  register: copiando_template

- name: Copiando o arquivo de service da app para o host
  copy: src={{ item.src }} dest={{ item.dest }}
  with_items:
    - {src: 'service-app.yml', dest: '/opt/giropops/service-app.yml'}
  register: copiando_service_file

- name: Criando o deployment da App Giropops
  shell: kubectl apply -f /opt/giropops/app-v1.yml
  register: deploy_app

- name: Criando o Service da App Giropops
  shell: kubectl apply -f /opt/giropops/service-app.yml
  register: deploy_svc_app
```

```
# vim roles/deploy_app/templates/app-v1.yml.j2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: giropops-v1
spec:
  replicas: {{ numeros_replicas }}
  selector:
    matchLabels:
      app: giropops
  template:
    metadata:
      labels:
        app: giropops
        version: {{ version }}
      annotations:
        prometheus.io/scrape: "{{ prometheus_scrape }}"
        prometheus.io/port: "{{ prometheus_port }}"
    spec:
      containers:
      - name: giropops
        image: linuxtips/nginx-prometheus-exporter:{{ version }}
        env:
        - name: VERSION
          value: {{ version }}
        ports:
        - containerPort: {{ nginx_port }}
        - containerPort: {{ prometheus_port }}
```

```
# vim roles/deploy_app/files/service-app.yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: giropops
    run: nginx
    track: stable
  name: giropops
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32222
    name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - nodePort: 32111
    name: prometheus
    port: 32111
    protocol: TCP
    targetPort: 32111
  selector:
    app: giropops
  sessionAffinity: None
  type: NodePort
```

```
# vim roles/deploy-app/vars/main.yml
---
# vars file for deploy-app
numeros_replicas: 10
version: 1.0.0
prometheus_scrape: "true"
prometheus_port: 32111
nginx_port: 80
```

```
ansible-playbook -i hosts main.yml

kubectl get pods
kubectl get svc
kubectl get deploy

cd ../
vim provisioning/roles/criando-instancias/tasks/provisioning.yml
```

```
# adiciona depois do - proto: udp
  - proto: tcp
    from_port: 32222
    to_port: 32222
    cidr_ip: 0.0.0.0/0
    rule_desc: GiropopsApp
  - proto: tcp
    from_port: 32111
    to_port: 32111
    cidr_ip: 0.0.0.0/0
    rule_desc: GiropopsAppProm
```

Abre no navegador o `<ip>:32222` e o `<ip>:32111/metrics`

## Dia 5

### Utilizando Handlers em nosso playbook


```
# install_k8s/roles/install-k8s/tasks/install.yml
- name: Instalando o Docker
  shell: curl -fsSL https://get.docker.com | bash
  notify: Restart Docker
  
# ...
- name: Instalando os pacotes kubeadm, kubelet e kubectl
  apt:
    name: "{{ packages }}"
  vars:
    packages:
    - kubelet
    - kubeadm
    - kubectl
  notify: Restart Kubelet
```

```
# install_k8s/roles/install-k8s/handlers/main.yml
---
# handlers files for install-k8s
- name: Restart Docker
  service:
    name: docker
    state: restarted

- name: Restart Kubelet
  service:
    name: kubelet
    state: restarted
```

Abre o hosts e apaga os IPs de [kubernetes]

```
ansible-playbook -i hosts main.yml
```

```
cd projeto-k8s-cluster/install_k8s
```

Atualiza os IPs

### Criando o playbook para o deploy da App Giropops V2

```
cd ..
mkdir deploy-app-v2
cp ../deploy-app-v1/hosts .
cp ../deploy-app-v1/main.yml .
mkdir roles
cd roles

ansible-galaxy init deploy-app-v2
cd ..
vim main.yml
```

Muda de "app" para "app-v2"

```
# tasks/main.yml
---
# tasks file for deploy-app-v2
- include: deploy-app-v2.yml
```

```
# tasks/deploy-app-v2.yml
- name: Instalando o pip
  apt:
    name:
      - python-pip

- name: Instalando dependencias dos modulos do k8s
  pip:
    name:
      - openshift
      - PyYAML

- name: Copiando o arquivo de deployment da app v1
  template:
    src: app-v1.yml.j2
    dest: /opt/giropops/app-v1.yml
    owner: root
    group: root
    mode: 0644
  register: copiando_deploy_app_v1

- name: Copiando o arquivo de deployment da app v2
  template:
    src: app-v2.yml.j2
    dest: /opt/giropops/app-v2.yml
    owner: root
    group: root
    mode: 0644
  register: copiando_deploy_app_v2

- name: Deploy da app-v2
  k8s:
    state: present
    namespace: default
    src: /opt/giropops/app-v2.yml

- name: Scale down da app-v1
  k8s:
    state: present
    namespace: default
    src: /opt/giropops/app-v1.yml

- name: Dentro de 2 minutos o app-v1 sera removido. Pressione ctrl+c para cancelar
  pause:
    minutes: 2

- name: Removendo o app-v1
  k8s:
    state: absent
    namespace: default
    src: /opt/giropops/app-v1.yml
```

```
# templates/app-v1.yml.j2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: giropops-v1
spec:
  replicas: {{ number_replicas_old_version }}
  selector:
    matchLabels:
      app: giropops
    template:
      metadata:
        labels:
          app: giropops
          version: {{ old_version }}
        annotations:
          prometheus.io/scrape: "{{ prometheus_scrape }}"
          prometheus.io/port: "{{ prometheus_port }}"
      spec:
        containers:
        - name: giropops
          image: linuxtips/nginx-prometheus-exporter:{{ old_version }}
          env:
          - name: VERSION
            value: {{ old_version }}
          ports:
          - containerPort: {{ nginx_port }}
          - containerPort: {{ prometheus_port }}
```

```
# templates/app-v1.yml.j2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: giropops-v2
spec:
  replicas: {{ number_replicas_new_version }}
  selector:
    matchLabels:
      app: giropops
    template:
      metadata:
        labels:
          app: giropops
          version: {{ new_version }}
        annotations:
          prometheus.io/scrape: "{{ prometheus_scrape }}"
          prometheus.io/port: "{{ prometheus_port }}"
      spec:
        containers:
        - name: giropops
          image: linuxtips/nginx-prometheus-exporter:{{ new_version }}
          env:
          - name: VERSION
            value: {{ new_version }}
          ports:
          - containerPort: {{ nginx_port }}
          - containerPort: {{ prometheus_port }}
```

```
# vars/main.yml
---
# vars file for deploy-app-v2
number_replicas_old_version: 1
number_replicas_new_version: 10
old_version: 1.0.0
new_version: 2.0.0
prometheus_scrape: "true"
prometheus_port: 32111
nginx_port: 80
environment: production
```

```
cd deploy-app-v1
cp ../install_k8s/hosts .

ansible-playbook -i hosts main.yml
```

```
# terminal 1
watch -n1 kubectl get pods
```

## Dia 6

### AWX - intro

AWX é uma forma de executar os playbooks de uma forma gráfica

Criar uma instância AWS EC2 e instalar docker, ansible, pip

```
curl -fsSL https://get.docker.com | bash
docker version

sudo apt-add-repository --yes --update ppa:ansible/ansible
apt install ansible
ansible --version

apt install python-pip
pip install docker-compose==1.9.0 docker-py

apt get install npm nodejs
npm install npm --global

apt install -y tree

git clone git@github.com:ansible/awx.git
```

```
cd awx/installer
vim install.yml

cd roles/

# arquivos legais de ver:
vim local_docker/tasks/compose.yml
vim local_docker/templates/docker-compose.yml.j2
vim local_docker/defaults/main.yml
vim local_docker/tasks/set_image.yml
vim inventory # onde acontece a mágica
```

```
openssl rand -hex 32 # cola no secret_key do inventory
```

Descomenta ou modifica cada linha como o bloco a seguir:

```
postgres_data_dir="~/var/lib/pgdocker"
docker_compose_dir="/var/lib/awx"
project_data_dir=/var/lib/awx/projects
```

Para pegar o arquivo sem os comentários e colocar em outro arquivo:

```
cat inventory | grep -v "^#" | grep -v "^$" > inventory_clean
mv inventory_clean inventory # para substituir
```

```
ansible-playbook -i inventory install.yml

docker ps
docker logs -f <id_web>
```

Acessa o IP público pelo navegador

### AWX - Mais alguns detalhes

Máquina local:

```
cd projeto-k8s-cluster/provisioning
vim hosts
```

Apaga os IPs do kubernetes. Cria uma IAM e configura as credenciais.

```
ansible-playbook -i hosts main.yml
cat hosts

cd .. && rm -rf extra
```

### Criando nosso inventário e o nosso projeto

Na página do AWX:

#### Inventories > + > Inventory

Name: hosts
description: meu inventario
organization: Default
variables:

```
---
K8S_MASTER_NODE_IP=<IP>
K8S_API_SECURE_PORT=6443
```

#### Hosts > +

Host name: <ip_1>
Description: k8s master
    
#### Hosts > +

Host name: <ip_2>
Description: k8s worker

#### Hosts > +

Host name: <ip_3>
Description: k8s worker
    
#### Projects > +

Name: Kubernetes
Description: Configure k8s cluster
Organization: Default
SCM Type: Git
SCM URL: https://github.com/badtuxx/descomplicando-ansible-2020.git
SCM branch/tag/commit: master
SCM update options: clean, delete on update, update revision on launch


### AWX - Criando e executando o nosso primeiro template

#### Credentials > create credential

Name: Ansible class
Description: minha chave
Organization: Default
Credential type: Machine
SSH Private key: conteúdo da chave ansible-class.pem
Privilege escalation method: sudo
    
#### Templates > +
    
Name: Configurando o K8s
Description: Configuracao do k8s com 3 nodes
Job type: Run
Inventory: hosts
Project: kubernetes
Playbook: install_k8s/main.yml
Credentials: ansible-class
Verbosity: 1 (Verbose)
Options: enable privilege escalation, enable concurrent jobs
    
#### Templates > Configurando o K8s > foguetinho
    
Espera subir
    
#### Inventories > hosts > groups > +
    
Name: k8s-master
    
#### Inventories > hosts > hosts > +

Seleciona o primeiro IP público

#### Inventories > hosts > groups > +
    
Name: k8s-worker
Variables:
    
```
---
K8S_MASTER_NODE_IP: <IP>
K8S_API_SECURE_PORT: 6443
```
    
#### Inventories > hosts > hosts > +

Seleciona os outros dois IPs públicos
    

Máquina onde está rodando o kubernetes:
    
```
ssh ubuntu@<ip>
sudo su -
kubectl get nodes
kubectl get all --all-namespaces
```

### AWX - Criando e executando o nosso workflow

Copia o `Configurando o K8s`

Name: Deploy Giropops App v1
Description: Deployando a versao 1 do giropops app
Playbook: deploy-app-v1/main.yml
Verbosity: 0 (Normal)

Copia o Deploy Giropops App v1

Name: Deploy Giropops App v2
Description: Deployando a versao 2 do giropops app
Playbook: deploy-app-v2/main.yml

#### Template > + > workflow template

Name: Giropops App
Description: Deploys do K8s, App v1 e App v2
Organization: Default
Inventory: hosts

#### Templates > Giropops App > Workflow Visualizer

Clica em Start

Configurando o K8s
Run: Always
Convergence: Any

Clica em +

Deploy Giropops App v1
Run: On Success
Converge: Any

Clica em +

Deploy Giropops App v2
Run: On Success
Converge: Any

Clica em +

Deploy Giropops App v1
Run: On Failure
Converge: Any

Caso falhe, ele volta pro v1


Executa o `Giropops App`

```
curl htttp://<ip>:32222/
```

### Ansible-Vault - como utilizar

```
cd deploy-app-v1
vim roles/deploy-app/vars/main.yml
```

```
---
# vars file for deploy-app
numeros_replicas: 10
version: 1.0.0
prometheus_scrape: "true"
prometheus_port: 32111
ngix_port: 80
db_root_password: Giropops123Mudar
db_admin_user: godofredo
password_tosko: asfafedde
```

Obs: não é seguro deixar senha em arquivos

```
ansible-vault --help
ansible-vault encrypt roles/deploy-app/vars/main.yml # coloca a senha
cat roles/deploy-app/vars/main.yml
ansible-vault decrypt roles/deploy-app/vars/main.yml # coloca a senha

ansible-vault encrypt roles/deploy-app/vars/main.yml --output teste.yml
# apenas o teste fica encryptado

ansible-vault encrypt roles/deploy-app/vars/main.yml

ansible-playbook -i hosts main.yml --ask-vault-pass

touch ~/.vault_password
vim ~/.vault_password # senha
ansible-playbook -i hosts main.yml --vault-password-file ~/.vault_password
```

Sobe todas as máquinas da AWS e possivelmente mudar os IPs públicos.

```
ansible-playbook -i hosts main.yml --vault-password-file ~/.vault_password

ansible-vault create opa # coloca a senha e o texto do arquivo
ansible-vault decrypt opa

ansible-vault encrypt opa
ansible-vault rekey opa # para mudar de senha

ansible-vault edit roles/deploy-app/vars/main.yml
ansible-vault view roles/deploy-app/vars/main.yml
```

Caso precise colocar senhas diferentes

```
# vim ~/.ansible.cfg
[defaults]
host_key_checking = False
vault_identity_list = opa@~/.vault_password, ipa@prompt
```

```
ansible-vault create --vault-id opa teste.yaml # pede a senha do ipa

ansible-vault decrypt roles/deploy-app/vars/main.yml
ansible-vault encrypt --vault-id opa roles/deploy-app/vars/main.yml

ansible-playbook -i hosts main.yml
```

Abre o `~/.ansible.cfg`, tira o ipa e deixa o opa como prompt

```
ansible-playbook -i hosts main.yml # pede a senha do opa
```

### Ansible-Vault - Como configurar no AWX

#### Deploy Giropops App v1

Atualiza os IPs. Se pedir pra executar de novo, vai dar erro porque não tem o vault secrets.

#### Credentials > create

Name: uoa
description: vault
organization: default
credential type: vault
vault password: giropops
vault identifier: opa

#### Templates > Deploy Giropops App v1

Credentials: adiciona o opa
