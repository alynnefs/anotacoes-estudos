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

`cp -R 2 3`

`cd 3/`

`vim Dockerfile`

Adiciona depois do ENV:

```
COPY index.html /var/www/html
```

Cria o index.html no mesmo nível que o Dockerfile, com um texto qualquer e builda

`docker image build -t meu_apache:3.0.0 .`

`docker container run -d -p 8000:80 meu_apache:3.0.0`

Abre o `localhost:8000` no navegador para ver o index.

`docker container exec -ti <container_id> bash`

`cat /var/www/html/index.html`

Para mudar a permissão de execução, adiciona abaixo do RUN:

```
RUN chown www-data:www-data /var/lock && chown www-data:www-data /var/run/ && chown www-data:www-data /var/log/
```

Como não tem permissão para usar porta "baixa" (80), o usuário voltou a ser `root`

Substitui o COPY pelo ADD. A diferença é que o COPY copia arquivos de diretórios e adiciona no caminho definido. O ADD faz isso e copia o arquivo tar extraído, passando os arquivos que ficam nele. Se tem um arquivo remoto, por exemplo em `www.qualquercoisa.arquivo.txt` ele consegue baixar o arquivo e adicionar no container.

Depois de LABEL adiciona a linha seguinte para rodar com o usuário específico, não apenas root:

```
USER root

WORDIR /var/www/html
```

WORKDIR é o diretório padrão. Quando o container subir, vamos "cair" direto no diretório definido

`docker image build -t meu_apache:4.0.0 .`

`docker container run -d -p 8080:80 meu_apache:4.0.0`

`docker container logs -f <container_id>`

`curl localhost:8080`

--

## MultiStage

`mkdir 4 && cd 4/`

`vim meu_go.go`

```
package main
import "fmt"

func main(){
    fmt.Println("Giropops Strigus Girus")
}
```

`vim Dockerfile`

```
FROM golang

WORKDIR /app
ADD . /app
RUN go build -o meugo

ENTRYPOINT ./meugo
```

`docker image build -t meugo:1.0 .`

`docker container run -ti meugo:1.0`

`docker image ls`

tamanho aproximado da imagem: 943MB

Deve retornar "Giropops Strigus Girus"

`cd .. && cp -R 4 5 && cd 5/`

`vim Dockerfile`

modifica o arquivo da seguinte forma, com explicação no comentário do arquivo

```
FROM golang AS buildando

WORKDIR /app
ADD . /app
RUN go build -o meugo

# o primeiro FROM é referente ao código até aqui

FROM alpine
WORKDIR /giropops
COPY --from=buildando /app/meugo /giropops/

# vai copiar o meugo da primeira camada para o giropops da segunda camada da imagem 

ENTRYPOINT ./meugo
```

`docker image build -t meugo:2.0 .`

`docker container run -ti meugo:2.0`

`docker image ls`

tamanho aproximado da imagem: 7.53MB

Isso acontece porque a imagem do alpine, que existe separadamente, tem aproximadamente 5.59MB. A imagem do golang tem aproximadamente 941MB. O primeiro FROM, a primeira fase da imagem, é utilizada para criar o "pré-container". A segunda fase usa a primeira.

## Palestra: Images Deep Dive, Healthcheck e Docker commit

- No dockerfile é possível usar multiline. Por exemplo:

```
RUN yum update -y && \
  yum intall -y apache2 \
    git \
    java \
    python
```

- Somente adicionar os pacotes necessários. Se precisar fazer troubleshooting, depois instala as dependências

- Limpe a casa

```
ENV MRAAVERSION v1.3.0
RUN git clone https://github.com/intel-iot-devkit/mraa.git && \
cd mraa && \
git checkout -b build ${MRAAVERSION} && \
<some build steps> make install && \
rm -rf /root/mraa
```

- Reduza o número de camadas
- Use o `.dockerignore`. O arquivo deve ficar no mesmo nível do `Dockerfile`
- Adicione um `non-root` user
- CMD: serve para executar um comando
```
# exec:
CMD ["giropops", "param1", "param2"]

#modo shell:
CMD giropops param1 param2
```
- ENTRYPOINT: principal processo na execução do container
```
# exec:
ENTRYPOINT ["giropops", "param1"]

#modo shell:
ENTRYPOINT giropops param1
```
- Se usar o ENTRYPOINT e o CMD no mesmo dockerfile é obrigatório usar no modo exec. O que tiver no CMD é apenas um parâmetro para o ENTRYPOINT.
```
# exec:
ENTRYPOINT ["giropops"]

#modo shell:
CMD ["param1"]
```
- Use ou não o cache, dependendo da intenção. Sem cache, vai instalar a última versão do que tem no `apt`
```
docker image build --no-cache -t opa:1.0 .
```
- Container é imutável e efêmero
- Use um script de start, se não tiver outra solução
```
ENTRYPOINT ["python", "start.py"]
```
- NÃO use um script de start, pois não é elegante
- Use variáveis
```
ENV alynne comy2n
```
- Use variáveis no dockerfile
```
ENV app_dir /opt/app
WORKDIR ${app_dir}
ADD . $app_dir
```
- SHELL
```
SHELL ["powershell", "-command"]
```
- LABELS
```
LABEL "com.example.vendor"="LINUXtips"
LABEL com.example.label-with-value="VAIIII"
LABEL version="1.0"
LABEL description="Aqui vc pode \
usar multi-line!"
```
- COPY ou ADD: os dois fazem a mesma coisa, copiar. O COPY copia arquivos e diretórios. O ADD, além disso, copia o conteúdo de arquivos da extensão tar e jar. Também pega arquivos remotos.

```
COPY . /app
COPY dir1 /app
COPY file /app

ADD . /app
ADD opa.tar /app
ADD http://abc.com/arquivo /app
```
- ARGS: argumentos que podem ser mudados no build

```
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER v1.0.0
RUN echo $CONT_IMG_VER
```
```
$ docker build --build-arg
CONT_IMG_VER=v2.0.0 Dockerfile
```
- HEALTHCHECK: faz uma verificação a cada x período de tempo
```
HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f http://localhost/ || exit 1
```
- Multi-stage: espécie de pipeline dentro do container

Demo:

`docker container rm -f $(docker ps -q)` para remover todos os containers. `-q` traz o id do container.

`cd ../ && cp -R 3 6 && cd 6/ && vim Dockerfile`

Adiciona o curl no RUN:

```
RUN apt-get update && apt-get install -y apache2 && apt-get clean curl
```

Adiciona o exemplo do HEALTHCHECK abaixo do ADD. Se demorar 3 segundos para responder, vai dar timeout e sair do container.

`docker image build -t meu_apache:5.0.0 .`

`docker container run -d -p 8080:80 meu_apache:5.0.0`

Quando executa `docker container ls`, no status aparece `Up X seconds (health: starting)`. Depois de aproximadamente um minuto, deve mudar para `Up About a minute (healthy)`

O ideal é colocar o HEALTHCHECK no docker-compose.

Para fazer falhar:

`docker container exec -ti <container_id> bash`

`echo "123.21.12.12 localhost" > /etc/hosts`

`ping localhost`

`curl -f http://localhost`

`docker container ls`

criar imagem de container a partir de um container em execução, sem dockerfile

`docker container run  -ti ubuntu`

`docker commit -m "ubuntu com vim e curl" <container_id>`

`docker image ls`

`docker image tag <image_id> ubuntu_vim_curl:1.0`

`docker container run -ti ubuntu_vim_curl:1.0`

## Dockerhub

`docker image ls`

`docker image tag <image_id> <id_do_dockerhub>/meu_apache:1.0.0`

Como vai ser o primeiro no DockerHub, pode ser chamado de `1.0.0`

Considerando que se tem uma conta no DockerHub:

`docker login`

`docker push <id_do_dockerhub>/meu_apache:1.0.0`

`docker image rm -f alynnefs/meu_apache:1.0.0`

`docker container run -d alynnefs/meu_apache:1.0.0`

Como executar a imagem padrão do registry

`docker container run -d -p 5000:5000 --restart=always --name registry registry:2`

`-d` daemon
`-p` porta
`--restart` caso tenha problema, reinicia o container
sempre pegar a versão 2, por ser a versão mais recente e compatível com as novas versões do docker

`docker logout`

`docker image tag <registry_image_id> localhost:5000/meu_apache:1.0.0`

sendo `endereço_registry/image`

`docker image push localhost:5000/meu_apache:1.0.0`

`docker container rm -f <id_do_container_em_execução>`

`docker image rm -f <id_da_imagem>`

`docker container run -d localhost:5000/meu_apache:1.0.0`

ele baixa novamente.

`curl localhost:5000/v2/_catalog`

`curl localhost:5000/v2/meu_apache/tags/list`

`docker exec -ti <id_registry> sh`

`ls /var/lib/registry/docker/registry/v2/repositories/meu_apache`
