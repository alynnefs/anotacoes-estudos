Aula 1

instalação Docker CE

https://docs.docker.com/install

EE: empresa - paga
CE: comunidade - gratuita

Baixar a versão estável (stable) de acordo com a plataforma (supported platforms)

`curl -fsSL https://get.docker.com | bash` -> pega o script e executa no bash

`docker container ls` verifica os containers que estão em execução no momento. Substitui o `docker ps`, que ainda funciona.

`docker container run -ti hello-world`

Docker client fala com docker daemon. Docker daemon faz o download do hello-world se ela não existir no computador. Daemon cria um container novo de acordo com a imagem baixada. Daemon diz para client a mensagem de saída.

`docker container run -ti ubuntu`
-ti : terminal e interatividade. Quando o container subir, já vou estar conectada.
`cat /etc/issue` mostra a versão do ubuntu
`exit` pra sair ou ctrl+d

`docker container ls -a` mostra todos os containers, inclusive os que já finalizaram
Entry point é o principal processo do container. No caso do ubuntu é o bash. Se o bash é finalizado, o container também é

`docker container run -ti centos`
`cat /etc/redhat-release` mostra a versão do centos
`ctrl+p+q` pra sair sem matar

`docker container attach <container_id ou nome_container>` volta para um container em execução

`docker container -d nginx` faz rodar como daemon e não de forma interativa. Roda em primeiro plano
`docker container attach <container_id>` em um daemon não vai funcionar por não ter terminal interativo.
`docker container exec -ti <container_id> <comando>`
`docker container exec -ti <container_id> ls /usr/share/nginx/html`

`docker container stop <container_id ou nome_container>` para o container
`docker container start <container_id ou nome_container>` inicia o container
`docker container restart <container_id ou nome_container>` reinicia o container
`docker container inspect <container_id ou nome_container>` mostra todas as informações do container

No container no nginx: `echo "GIROPOPS STRIGUS GIRUS" > /usr/share/nginx/html/index.html` e `exit`. Se fizer `curl 172.17.0.3` ele retorna "giropops ...".
`docker container pause <container_id ou nome_container>` pausa o container, se der curl no ip dele, não retorna nada
`docker container unpause <container_id ou nome_container>` tira o container da pausa
com o curl, volta a retornar "giropops"

`docker container rm <container_id ou nome_container>` usado para remover o container. Usar a tag -f para forçar a remoção. `docker container rm -f <container_id ou nome_container>`.

`docker container stats <container_id ou nome_container>` diz percentual de CPI, total e percentual de memória usada, I/O de rede e I/O de disco
`while true; do curl 172.17.0.3; done` em outro terminal para testar o uso de CPU e I/O de rede subindo
`docker container exec -ti <container_id ou nome_container> bash`
`apt-get update && apt-get install stress` para fazer teste de estresse na máquina
exemplo`stress --cpu 1 --vm-bytes 128M --vm 1`. É possível colocar apenas parte de um core de CPU, usando --cpu 0.5 por exemplo
`docker container top <container_id ou nome_container>` mostra os processos que estão em execução dentro do container
`docker container run -d -m 128M nginx` ou --memory - máximo de memória que pode ser usado
`cat /proc/cpuinfo` mostra informações de CPU

`docker image ls` mostra um resumo das imagens

`mkdir tosko_dockerfile`
```
FROM debian  # a partir de quem quero criar a imagem

LABEL app="Giropops"  # criar informação no metadado da imagem do container
ENV ALYNNE="COMYE2N"  # para variável de ambiente

RUN apt-get update && apt-get install -y stress && apt-get clean # executar algum comando em tempo de build

CMD stress --cpu 1 --vm-bytes 64M --vm 1  # comando que vai ser executado
```
um CMD por dockerfile

`docker image build -t toskeira:1.0 .` 
nome_da_imagem:versão. O ponto simboliza que o dockerfile está neste nível.

`docker container run -d -m 64M toskeira:1.0`
`docker container ls`
`docker container logs -f <container_id>`
`docker container stats <container_id>`
`docker container update --cpu 0.2`

`docker rmi -f <imagem_id>` remove a imagem

