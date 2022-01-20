Aula 03

## Docker Machine

Serve para criar e gerenciar hosts com o docker instalado. Para o docker-machine, drivers são as plataformas suportadas, como por exemplo, AWS, Azure e VirtualBox.

https://docs.docker.com/machine/drives/

- Seguir o passos da documentação no host, não pode ser em VM

https://github.com/docker/machine/releases/

`docker-machine --version`

`docker-machine --help`

`docker-machine create --driver virtualbox giropops` cria uma máquina virtual no virtualbox

`docker-machine ls` mostra as máquinas

`docker-machine ip giropops` mostra o ip da máquina escolhida

`docker-machine ssh giropops` conecta com a máquina via SSH

`cat /etc/issue` mostra a distribuição que está em execução

No desktop

`docker-machine env giropops` mostra as variáveis de ambiente

- DOCKER_TLS_VERIFY: serve para verificar o TLS
- DOCKER_HOST: caminho do host
- DOCKER_CERT_PATH: onde estão os certificados
- DOCKER_MACHINE_NAME: nome da máquina
- o eval executa as variáveis

`docker container run -d -p 8080:80 nginx`

`docker container ls` mostra o container do nginx

`curl <ip_docker_machine>:8080` mostra o html do nginx. No navegador também funciona.

`docker-machine inspect giropops` retorna os detalhes da máquina virtual

`docker-machine stop giropops`

`docker-machine ls giropops`

`docker-machine status giropops`

`docker-machine start giropops`

Para remover a máquina:

`docker-machine env -u`

`docker-machine rm giropops`

## Docker Swarm

Um orquestrador serve para integrar os containers com outras ferramentas. Os mais conhecidos atualmente são o Swarm e o Kubernetes. Com ele é possível fechar um cluster Swarm para fazer a orquestração dos containers, tem um ambiente seguro, em alta disponibilidade, distribuição e balanceamento de carga.

Duas características importantes dentro do Swarm: manager e worker. A principal função do worker é "carregar o piano", ter containers em execução. O manager conhece os detalhes do cluster, nele está a administração dos nós e containers.

Resiliência
- Pelo menos 51% dos managers funcionando. Compensa ter um número ímpar.
- Quando um manager cai, acontece uma eleição para escolher o próximo manager ativo. Quanto mais nodes, mais tempo levará para escolher.

`docker swarm --help`

`docker swarm init`

Nas máquinas 2 e 3:

- `curl -fsSL https://get.docker.com | bash`

- Copia o `docker swarm join --token <token> <ip:porta>` e cola nas máquinas

Na máquina principal:

- `docker node ls`. Vão aparecer as 3 máquinas, sendo a primeira como "Leader"
- `docker node promote maquina-2`
- `docker node ls`: a máquina 2 agora está como "Reachable" (alcançável)
- `docker node promote maquina-3` para promover a 3
- `docker node ls`: máquinas 2 e 3 estão como "Reachable"
- `docker node demote maquina-3` para remover a máquina 3

Na máquina 2:

- `docker swarm leave` vai dar errado, por se tratar de um manager. Para sair, precisa usar o `-f`
- `docker swarm leave -f`

Caso tenha mais de uma interface `eth`, usar: 
`docker swarm init --advertise-addr <ip_eth0_ou_1>`

`docker node demote maquina-3` novamente para ter resiliência, com um número ímpar de managers

`docker node demote maquina-2`

`docker node -f rm maquina-2`

`docker swarm join-token worker` para pegar o token de worker

`docker swarm join-token manager` para pegar o token de manager

Na máquina 2, usa o comando do manager. Ele volta a aparecer na máquina 1, já como manager

`docker swarm join-token --rotate manager` gera outro token de manager. Também serve com o worker

### Node

Na máquina 1
- `docker service create --name webserver --replicas 3 -p 8080:80 nginx`
- `docker service ps webserver`
- `docker node update --availability [drain|pause|active] maquina-2` vamos usar o 
- usa o `docker node inspect maquina-2` para olhar o campo "Availability"
- `docker service scale webserver=10`
    - o nó 2 não pode mais receber nenhum container
- `docker service ps webserver` para ver onde está rodando cada container. Os novos foram para o 1 ou 3
- `docker node update --availability active maquina-2`
- `docker service scale webserver=10`, a máquina 2 volta a receber containers
- `docker node update --availability drain maquina-2` vai desligar os containers que estão na máquina 2 e vão subir novos nas máquinas 1 e 3
- Depois de ativar, usa o scale para diminuir para a quantidade de máquinas e depois para 10 novamente

### Services

Forma de ter diversos containers respondendo a determinado serviço

`docker service create --name giropops --replicas 3 -p 8080:80 nginx`

8080 é referente à porta do cluster e vai redirecionar a todas as réplicas de container

`docker service ls`
- modo: "replicated"
- réplicas: disponíveis/total

`docker service ps giropops` mostra todas as réplicas

`docker service inspect giropops --pretty`

`docker service scale giropops=10`

#### Services e volume

`docker volume crete giropops`

`cd /var/lib/docker/volumes/giropops/_data/`

`vim index.html`

```
GIROPOPS STRIGUS GIRUS
```

`docker service create --name giropops --replicas 3 -p 8080:80 --mount type=volume,src=giropops,dst=/usr/share/nginx/html/ nginx`

Executar algumas vezes para ver em réplicas diferentes
- `docker service ls`
- `curl localhost:8080` 

Só aparece a mensagem em uma das réplicas, porque não é o mesmo index. O docker não sincroniza o conteúdo dos volumes por padrão.

Na máquina 1. Exemplo não recomendado para compartilhar:
- `sudo apt install nfs-server`
- `mkdir /opt/site`
- `vim /etc/exports`
- coloca as opções:
    - `/var/lib/docker/volumes/giropops *(rw,sync,subtree_check)`
    - `/opt/site *(rw,sync,subtree_check)`
- exportfs -ar
- `chmod 777 /var/lib/docker/volumes/giropops -R`, também não recomendado, é apenas demonstração

Na máquina 2 (client):
- `sudo apt install nfs-common`
- `showmount -e <ip_maquina_1>` para ver se ele "enxerga" a máquina
- `mount -t nfs <ip_maquina_1>:/opt/site /var/lib/docker/volumes/giropops`, sendo origem e destino do que vai ser compartilhado


Para verificar
- Máquina 1:
    - `mkdir /opt/site/_data`
- Máquina 2:
    - `ls /var/lib/docker/volumes/giropops`
- Máquina 1:
    - `docker volume rm giropops`
    - `docker volume create giropops`
    - `ln -s /opt/site/ /var/lib/docker/volumes/giropops/`
    - `echo "GIROPOPS STRIGUS GIRUS" > /opt/site/_data/index.html`
- Máquina 2:
    - `cat _data/index.html`
- Máquina 1:
    - `docker service create --name giropops --replicas 3 -p 8080:80 --mount type=volume,src=giropops,dst=/usr/share/nginx/html/ nginx`
    - `curl localhost:8080` algumas vezes
