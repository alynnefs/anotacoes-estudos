# Aula 4

## Secrets

Os secrets têm como finalidade armazenar dentro do container informações sensíveis, como senhas. Sua utilização aconte sempre em um cluster swarm.

`echo -n "GIROPOPS STRIGUS GIRUS" | docker secret create alynne -`

`-n` para não ter quebra de linha
`-` para mandar o retorno do echo para o docker

`docker secret ls`

`docker secret inspect alynne`

teste.txt:
```
GIROPOPS STRIGUS GIRUS
```

`docker secret create alynne-arquivo teste.txt`

`docker secret rm alynne`

`docker service create --name nginx -p 8080:80 --secret alynne-arquivo nginx`

`docker service ls`

`docker service inspect nginx` para olhar o secret

`docker service scale nginx=3`

`docker exec -ti <container_id> bash`

`cat /run/secrets/alynne-arquivo`

`docker service update --secret-add alynne nginx` atualiza as 3 instâncias

`docker secret ls`

`docker container ls`

`docker exec -ti <container_id> bash`

`cat /run/secrets/alynne`

`docker service create --name2 nginx -p 8081:80 --secret src=alynne-arquivo,target=meu-secret,uid=200,gid=200,mode=0400 nginx`
- uid: dono
- gid: usuário específico do container
- mode: permissões

`docker service scale nginx2=3`

`docker exec -ti <container_id> bash`

`cat /run/secrets/meu-secret`

Para os containers e `docker secret rm <secrets>`, apagando os dois

## Compose

É uma forma de passar para um arquivo como que você deseja que a imagem e o ambiente seja deployado. Pode fazer um compose para a aplicação web e uma para o banco, mas também pode criar no mesmo arquivo.

primeiro docker-compose.yml (colocar link)

`docker stack deploy -c <nome_arquivo> <nome_stack>` é usado para fazer deploy do docker compose
`-c` é usado por ser um compose

`docker stack deploy -c docker-compose.yml giropops`

`docker network ls`

`docker network inspect giropops-webserver`

`docker stack ls`

`docker stack ps giropops`

`docker stack services giropops`

`docker stack rm giropops` remove o service e a rede

Docker compose é usado quando se está pensando na imagem pronta para fazer deploy. Dockerfile é usado para criar a imagem.

(link para o segundo docker compose)

- db_data é o nome do volume que vai ser criado e `/var/lib/mysql` é o caminho
- Environment são as variáveis de ambiente
- depends_on: antes de iniciar o wordpress, precisa terminar o mysql
- db_data na última linha é o volume criado acima

`docker stack deploy -c docker-compose.yml wp`

`docker stack ps wp`

`docker service logs -f wp_db`

No navegador: `<ip_wordpress>:8000` 
- idioma: português
- título: GIROPOPS
- usuário: strigus
- senha: teste
- e-mail: teste@teste.com

se olhar o ip de novo, vai retornar o site

(link do terceiro arquivo)

`node.labels.dc` onde vai ser o data center

`/var/run/docker.sock` interface visual para ver onde estão os containers

`node.rolle == manager` só vai rodar quando o node for manager

`docker stack deploy -c docker-compose.yml giropops`

`docker stack ls` mostra que tem dois serviços

`docker stack ps giropops` mostra que deu falha na hora de agendar (scheduling)

`docker node update --label-add <chave-valor>`

`docker node update --label-add dc=UK ip-172-31-55-78` ip da máquina 1 

`docker stack ps giropops`

no navegador: <ip_wp:8888>

`docker service scale giropops_web=10`

`docker stack ps giropops`

`docker stack rm giropops`

(link do quarto arquivo)

- a porta definida no redis é para uso interno
- parallelism: se fizer algum update no serviço, faz de acordo com a quantidade definida. Por exemplo, de duas em duas
- dentro de deploy dá pra adicionar definições rollback.
    - failure_action pode ser "continue" ou "pause"
    - start_first primeiro inicia os outros para depois parar
    - está disponível a partir da versão 3.7
- worker
    - mode: global é um container para cada nó. Replicated permite ter um container para mais de uma réplica
    - window: janela de tempo
- visualizer
    - stop_grace_period: espera x segundos para "morrer", depois disso dá um "kill -9"

`docker stack deploy -c docker-compose.yml vote`

`docker service ls`

`docker service ps vote_db`

No navegador:
- <ip:5000> para votar
- <ip:5001> para ver o resultado

`docker stack rm vote`

(link para o quinto compose)

- `{{.Node.ID}}` id que pode ser encontrado pelo inspect
- cadvisor: pega métrica dos containers
- grafana.config é onde vai passar a senha do usuário


--

aula ao vivo

`curl -fsSL https://get.docker.com | bash` para instalar o docker


Protocolo | porta |
 --| --|
TCP | 2377|
TCP/UDP | 7946 |
UDP | 4789

`docker swarm init`

copia o comando `docker swarm join --token SWMTKN-1-<hash> <ip>:<porta>` e cola nas máquinas 2 e 3

Webhook é a forma que o alert manager vai mandar mensagem para o slack
