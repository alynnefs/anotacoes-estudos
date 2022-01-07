# Aula 2 - volumes e imagens

## Volumes

`mkdir /opt/giropops`

`docker container run -ti --mount type=bind,src=/opt/giropops,dst=/giropops debian`

temos dois tipos de volumes no docker: bind e volume. Bind é quando eu já tenho um diretório e quero montar este diretório no container. O tipo volume cria o volume automaticamente dependendo do dockerfile. Quando não tem a gestão de volumes, o diretório vai receber um nome aleatório

src ou source é quem eu quero montar

dst ou destination é onde eu quero montar no container

dentro do container:

- usando o `ls` dá pra ver a pasta giropops
- `cd giropops/`
- `touch teste`

depois que sair do container

- `docker container ls` para ver que o container não está mais sendo executado
- com `ls /opt/giropops/` dá para ver o arquivo criado através do docker


o parâmetro `ro` pode ser usado para deixar como "read only", apenas leitura

`docker container run -ti --mount type=bind,src=/opt/giropops,dst=/giropops,ro debian`

ao excutar `rm teste`, aparecerá o erro `rm: cannot remove 'teste': Read-only file system`

`docker volume ls` lista todos os volumes

`docker volume create giropops` cria um volume

`docker volume inspect giropops` mostra as especificações

todos os volumes ficam no `/var/lib/docker/volumes`

para criar arquivos, tem que acessar a pasta `_data`, que por sua vez está dentro da pasta que tem o mesmo nome do volume:

- `cd /var/lib/docker/volumes/giropops/_data`
- `touch <nome_do_arquivo>`

para executar, muda o type para volume e deixa o src apenas como `giropops`:

`docker container run -ti --mount type=volume,src=giropops,dst=/giropops debian`


`ls giropops/`

`touch giropops/teste`

fora do container:

`ls /var/lib/docker/volumes/giropops/_data`

entra de novo e sai com `ctrl+p+q` para deixar em execução e cria outro container igual

`touch giropops/<nome_do_container>` e sai

`docker container exec -ti 9df1106eb162 touch /giropops/9df1106eb162`, sendo que o valor em hexa é o nome do primeiro container que estava sendo executado.

Se der `ls` na pasta do host de novo, aparecerão os nomes dos dois arquivos. Mesmo finalizando o container com `docker rm -f <container_id>`, os arquivos permanecem.

`docker volume rm giropops` retorna erro, pois algum container ainda está usando esse volume

`docker container rm -f <id_do_container_que_usa_o_volume>` e tenta apagar de novo

Usando `ls /var/lib/docker/volumes` dá pra ver que o volume giropops não existe mais


--

`docker volume create giropops`

`docker container run -ti --mount type=volume,src=giropops,dst=/giropops debian`

`ctrl+p+q`

`docker container inspect <container_id>`

na parte de Mounts mostra o RW do volume, onde está o dado original, onde monta no container, tipo, etc

Se remover o container, o volume continua existindo:
- `docker container rm -f <container_id>`
- `docker volume ls`

`docker volume prune` apaga todos os volumes que não estão sendo usados por pelo menos um container. Da mesma forma, `docker container prune` remove todos os containers que não estão sendo usados.

--


O `create` apenas cria o container, o `run`também executa

`docker container create -v /data --name dbdados centos`

-v é a notação antiga de volume

o `/data` é onde o volume vai ser montado no container

`docker container run -d -p 5432:5432 --name pgsql1 --volumes-from dbdados -e POSTGRESQL_USER=docker -e POSTGRESQL_PASS=docker -e POSTGRESQL_DB=docker kamui/postgresql`

-d para rodar em daemon

-p para escolher a porta host:container

--volumes-from forma antiga de gerenciar os volumes do container

-e enviroment, variável de ambiente para que o postgres funcione

kamui/postgresql é a imagem que será utilizada


Executa de novo mudando o nome e a porta: `docker container run -d -p 5433:5432 --name pgsql2 --volumes-from dbdados -e POSTGRESQL_USER=docker -e POSTGRESQL_PASS=docker -e POSTGRESQL_DB=docker kamui/postgresql`

Quem quiser acessar esse postgres vai acessar a 54322. Quando a requisição chegar no host (5433), ele vai redirecionar para a porta do container (5432)

`cd /var/lib/docker/volumes`

`docker container inspect <container_id>`

procura o mount e copia o nome do volume

`cd <nome>/_data/` e `ls`

aparecerão os dados gerados pelo postgres. Os dois containers estão compartilhando o mesmo volume. Não usar em produção!!


--


desafio: criar um volume do posgres

`docker volume create dbdados`

`docker container run -d -p 5432:5432 --name pgsql1 --mount type=volume,src=dbdados,dst=/data -e POSTGRESQL_USER=docker -e POSTGRESQL_PASS=docker -e POSTGRESQL_DB=docker kamui/postgresql`

`docker container run -d -p 5433:5432 --name pgsql2 --mount type=volume,src=dbdados,dst=/data -e POSTGRESQL_USER=docker -e POSTGRESQL_PASS=docker -e POSTGRESQL_DB=docker kamui/postgresql`

`docker container ls`

`docker volume ls`

`docker volume inspect dbdados`

Para fazer backup do banco de dados do volume em outro docker

`sudo mkdir /opt/backup`

`docker container run -ti --mount type=volume,src=dbdados,dst=/data --mount type=bind,src=/opt/backup,dst=/backup debian tar -cvf <caminho/nome.tar> /data`

`docker container run -ti --mount type=volume,src=dbdados,dst=/data --mount type=bind,src=/opt/backup,dst=/backup debian tar -cvf /backup/bkp-banco.tar /data`

o primeiro dst pode ser qualquer coisa, usamos o mesmo da criação do volume
o primeiro mount é quem vai ter backup feito, o segundo é para onde vai.
`cvf` é para empacotar. O `z` seria para compactar, o que não queremos
depois vem caminho e nome do arquivo de backup
diretório que eu quero fazer backup

`sudo ls /opt/backup/`

`tar -xvf bkp-banco.tar`

`ls data`

--

## Dockerfiles

`mkdir dockerfiles`

`mkdir 1`

`cd 1/`

`vim Dockerfile`

arquivo:

```
FROM debian

RUN apt-get update && apt-get install -y apache2 && apt-get clean
ENV APACHE_LOCK_DIR="/var/lock"
ENV APACHE_PID_FILE="/var/run/apache2.pid"
ENV APACHE_RUN_USER="www-data"
ENV APACHE_RUN_GROUP="www-data"
ENV APACHE_LOG_DIR="/var/log/apache2"

LABEL description="Webserver"
LABEL version="1.0.0"

VOLUME /var/www/html

EXPOSE 80
```

Se colocar em outro RUN, ele cria uma nova camada. Se tirar o `clean`, não tem diferença de tamanho, pelo menos no exemplo. Aparentemente não precisa mais.

VOLUME serve para criar um volume que vá para o caminho definido, dentro do container

EXPOSE define a porta do container que estará em execução. É equivalente ao segundo valor do `-p`:

`docker container run -ti -p 8080:80`


O `-P` vai procurar no dockerfile se tem alguma porta exposta no container. Se tiver, ele vai "bindar" em uma porta aleatória do host

`docker container run -ti -P`

`docker image build -t meu_apache:1.0.0 .`

`-t` é a tag, o nome da imagem

`docker image ls`

`dpkg -l | grep apache` para ver se está executando

`cd .. && cp -R 1 2 && cd 2` para copiar a pasta 1 e entrar na 2

`vim Dockerfile`

adiciona no fim do arquivo:

```
ENTRYPOINT ["/usr/sbin/apachectl"]
CMD ["-D", "FOREGROUND"]
```

ENTRYPOINT é o principal processo do container. Se ele morrer, o container morre. Outra forma de executar:

`CMD /usr/sbin/apachectl -D FOREGROUND`

mas quando tem entrypoint, o cmd não tem mais a função de chamar qualquer comando. O CMD está passando parâmetros para o bash. Quando tem ENTRYPOINT, o CMD vai passar parâmetro para ele. Esse parâmetro diz para rodar em primeiro plano, como daemon.

`docker image build -t meu_apache:2.0.0 .`

se não quiser usar cache, acrescenta no final `--no-cache`

`docker image ls` para ver as imagens 

`docker container run -d -p 8080:80 meu_apache:2.0.0`

`-d` para rodar em primeiro plano

`curl localhost:8080`


