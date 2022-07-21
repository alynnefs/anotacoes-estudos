# Terraform

## Aula 1

O binário terraform lê o arquivo de configuração. Ele tem um outro arquivo de estado, que salva o estado que você quer chegar. O binário vai na cloud e pede para criar as máquinas de acordo com o que tem no arquivo de estado.

```
# main.tf
provider "aws" {
  region  = "us-east-1"
  version = "~> 2.0"
}

terraform {
  backend "s3" {
    # Lembre de trocar o bucket para o seu, não pode ser o mesmo nome
    bucket = "descomplicando-terraform-gomex-tfstates"
    key    = "terraform-test.tfstate"
    region = "us-east-1"
  }
}
```

```bash
docker run -it -v $PWD:/app -w /app --entrypoint "" hashicorp/terraform:light sh
terraform version
terraform init # erro

export AWS_ACCESS_KEY_ID=<id_usuario_aws>
export AWS_SECRET_ACCESS_KEY=<chave_usuario_aws>
terraform init
```

```
# main.tf
provider "aws" {
  region  = "us-east-1"
  version = "~> 2.0"
}

terraform {
  backend "s3" {
    # Lembre de trocar o bucket para o seu, não pode ser o mesmo nome
    bucket = "descomplicando-terraform-gomex-tfstates"
    key    = "terraform-test.tfstate"
    region = "us-east-1"
  }
}
```

```
# ec2.tf
resource "aws_instance" "web" {
  ami           = "ami-0885b1f6bd170450c"
  instance_type = "t2.micro"

  tags = {
    Name = "HelloWorld"
  }
}
```

```bash
terraform init -upgrade # atualiza plugins
terraform plan # sem o out salva em memória
terraform plan -out <nome_do_arquivo>
terraform apply <nome_do_arquivo>
head <nome_do_arquivo>

terraform plan -destroy # para ver o id e outras informações
terraform destroy # para excluir. É aconselhado usar sempre que parar de executar as coisas
```

```
# main.tf
provider "aws" {
  region  = "us-east-1"
  version = "~> 3.0"
}

terraform {
  backend "s3" {
    # Lembre de trocar o bucket para o seu, não pode ser o mesmo nome
    bucket = "descomplicando-terraform-gomex-tfstates"
    key    = "terraform-test.tfstate"
    region = "us-east-1"
  }
}
```

```
# ec2.tf
resource "aws_instance" "web" {
  ami           = "ami-0885b1f6bd170450c"
  instance_type = "t2.micro"

  tags = {
    Name = "HelloWorld"
  }
}
```

```
# output.tf
output "ip_address" {
  value = aws_instance.web.public_ip
}
```

### Expression e console

```
terraform console
> data.aws_ami.ubuntu
```

```
# main.tf
provider "aws" {
  region  = "us-east-1"
  version = "~> 3.0"
}

terraform {
  backend "s3" {
    # Lembre de trocar o bucket para o seu, não pode ser o mesmo nome
    bucket = "descomplicando-terraform-gomex-tfstates"
    key    = "terraform-test.tfstate"
    region = "us-east-1"
  }
}
```

```
# output.tf
output "dns_name" {
  value = aws_instance.web.public_dns
}
```

```
# ec2.tf
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  owners = ["099720109477"] # Ubuntu
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name = "HelloWorld"
  }
}
```

### Providers

Região é um conjunto de data centers. Ex: us-east-1. Dentro da região tem Availability Zones (AZ). A máquina EC2 é colocada dentro de uma AZ. Quando seu serviço está sendo executado em mais de uma AZ diferente, ele é chamado de Multi AZ, mas o ideal é que ele seja Multi Region

#### Provider configuration

Isso varia de acordo com o cloud

```
provider "google" {
  project = "acme-app"
  region = "us-central1"
}
```

O binário do terraform não sabe chegar em nenhum provider, ele precisa de um plugin da AWS, do GCP, etc

alias: permite trabalhar com diferentes providers

```
provider "aws" {
  alias = "west"
  region = "us-west-2"
  version = "~> 3.0"
}

...

resource "aqs_instance" "web_west" {
  provider = aws.west
  ami = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  
  tags = {
    Name = "HelloWorld"
  }
}
```

`terraform plan -out plano`

`terraform apply "plano"` não conseguiu subir a EC2 pq a AMI não existe

adiciona o provider `aws.west` no `data "aws_ami" "ubuntu_west"` e roda os dois comandos de novo

## Variables

type constraint é o tipo da variável

sensitive: permite que o valor não apareça no console. É um booleano

```
# variables.tf

variable "image_id" {
  default = "ami-12345678" # tira caso queira forçar a pessoa a setar um valor
  type = string
  description = "The id of the machine image (AMI) to use for the server."
  validation {
    condition = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI id, starting with \"ami-\"."
  }
}
```

`terraform plan -var image_id="ami-<hash>" -out plano`

Executa primeiro com o valor errado para verificar a validação de nome

```
# variables.tfvars
image_id = "ami-abc123"
availability_zone_names = [
  "us-east-1a",
  "us-west-1c",
]
```

também tem como fazer com `tfvars.json`

`terraform plan -var-file="variables.tfvars -out plano"`

qualquer arquivo terminado com `.auto.tfvars` ou `.auto.tfvars.json` vai funcionar automaticamente.

`export TF_VAR_image_id=ami-abc456`
`terraform plan -out plano`

### Precedência de definição de variáveis

Ordem de prioridade:

1. -var e -var-file na linha de comando
2. *.auto.tfvars ou *.auto.tfvars.json
3. terraform.tfvars.json
4. terraform.tfvars
5. Variáveis de ambiente

## Aula 2

### Módulos

Módulo é uma forma de reunir as configurações do terraform, como recursos, data, output e afins, em um lugar. Facilita o compartihamento com outros times.

Caso um arquivo não esteja definido para utilização de módulos, por padrão ele é `root module`

Quando usa o módulo filho, você coloca inputs do módulo raiz nele e ele retorna output. O módulo filho instancia recursos no provider. O módulo raiz envia outputs para o operador.

Chamando um módulo filho

```
// terrafile.tf
// tudo o que tá dentro desse móduo é input para o módulo filho
module "servers" { // nome da pasta/módulo
  source = "./app-cluster"
  servers = 2
}
```

```
// variable.tf
variable "servers" { // não especifica default, vai obrigar a pessoa a especificar
  
}
```

```
// ec2.tf
// adiciona no resource "aws_instance" "web" {
  count = var.servers
}
```


`terraform init` # para inicializar os módulos
`terraform plan -out plano`
`terraform apply plano`

```
// adiciona no terrafile.tf
output "ip_address" {
  value = module.servers.ip_address
}
```

`servers` é o input do módulo raiz para o módulo filho. `ip_address` é o output do módulo filho para o módulo raiz

`terraform destroy`
`terraform plan -out plano`
`terraform apply plano`

Route53 é um serviço da AWS para gerenciamento de serviços DNS

### Backend

terraform.io/docs/backends/index.html

As configurações do backend são opcionais. Ele é um sub-bloco dentro do bloco terraform. A primeira configuração sempre começa com `terraform init`. Você não precisa especificar no arquivo tudo o que for necessário sobre o backend precisa, por exemplo, dados sensíveis, passar por arquivos, ou passar por linha de comando etc

```
// main.tf
terraform {
  backend "s3"{
    bucket = "iaasweek-tfstates-terraform"
    key    = "terraform-test.tfstate"
    region = "us-east-1"
  }
}
```

`terraform state pull >> aula-backend.tfstate`
`terraform state push aula-backend.tfstate` precisa deixar o serial maior que o anterior para funcionar

### State Locking

É a habilidade que o backend pode oferecer para quando estivermos trabalhando com o state line (?).

No caso do S3, se quiser usar o State Locking, precisa usar o Dynamo DB.

```
// dynamodb.tf
resource "aws_dynamodb_table" "dynamodb-terraform-state-lock" {
  name = "terraform-state-lock-dynamo"
  hash-key = "LockID"
  read_capacity = 20
  write_capacity = 20
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name = "DynamoDB Terraform State Lock Table"
  }
}
```

O comando de destroy também vai apagar o DynamoDB, podendo causar erro com o lock. É possível resolver apagando o `dynamodb_table` ou usando o comando `terraform destroy -lock=false`. Para executar de novo, `terraform plan -out plan -lock=false`, porque ele ainda não existe. `terraform apply -lock=false "plano"`

### State

https://www.terraform.io/docs/state/index.html

State é a forma que o Terraform encontra de armazenar informações sobre a estrutura. O padrão é usar um arquivo json local ou remoto. O Terraform não funciona sem state. Ele funciona para mapear o mundo real, metadata, performance, sincronia

`-refresh=false`
`terraform refresh` sem o plan
`terraform state pull > stateaula.tfstate`

### State avançado

`terraform state <subcommand> [options] [args]`

subcommads: list, mv, pull, push, rm, show

Caso gere modificação, é criado backup e isso não pode ser desabilitado.

`terraform state list`

`terraform state mv aws_instance.web module.app.aws_instance.web`

`terraform state rm module.app.aws_instance.web2`

rm não apaga o recurso, ele tira o recurso do state. Se quiser apagar, tem que entrar no console e apagar manualmente.

### Import

terraform.io/docs/import/usage.html

O comando `terraform import` é usado para importar uma infraestrutura existente

`terraform import <instancia>.<nome> <identificador>`


```
resource "aws_instance" "test" {
  ami = "ami-<hash>"
  instance_type = "t2.micro"
}
```

Security groups podem ser importados usando:

`terraform import aws_security_group.elb_sg sg-<hash>`

### Workspaces

terraform.io/docs/state/workspaces.html

`new` cria um novo, `select` utiliza um já existente

```
provider "aws" {
  region  = "${terraform.workspace == 'production' ? 'us-east-1' : 'us-east-2'}"
  version = "~> 2.0"
}

terraform {
  backend "s3" {
    bucket = "iaasweek=tfstates-terraform"
    key    = "terraform-test.tfstate"
    region = "us-east-1"
  }
}
```

`terraform workspace new production`

`terraform workspace new staging`

possíveis comandos: new, list, show, select e delete

`terraform plan` e `apply`

`terraform workspace select production`

O ideal é colocar staging e produção em regiões diferentes para evitar possíveis conflitos de nome e de produto

### Dados sensíveis no state

terraform.io/docs/state/sensitive-data.html

O S3 suporta o `encrypt`

```
terraform {
  backend "s3" {
    bucket  = "iaasweek=tfstates-terraform"
    key     = "terraform-test.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}
```

`terraform init`, seguido de plan e apply

## Dia 3

### Conseguindo ajuda

`terraform -install-autocomplete`

### Dependências entre recursos

`EIP` depende do `ec2 instance`, então cria primeiro o ec2. Para apagar, tem que apagar primeiro o `EIP` para depois apagar o `ec2`, de quem ele depende. Isso é a dependência implícita, a AWS já sabe como agir sem que a gente diga.

No caso de dependência explícita, usamos o `depends_on`

```
resource "aws_instance" "web2" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  
  tags = {
    Name = "HelloWorld"
  }
  depends_on = [aws_instance.web]
}
```

### Introdução a comandos e mudando o verbose deles

terraform.io/docs/internals/debugging.html

`TF_LOG` pode ter os níveis `TRACE`, `DEBUG`, `INFO`, `WARN` e `ERROR`

`TF_LOG=DEBUG terraform plan -out plan`

### Comando taint

terraform.io/docs/commands/taint.html

O taint serve para marcar que um recurso precisa ser destruído e recriado quando usar plan e apply novamente.

`terraform state list`

`terraform taint [options] address` # sendo address o que aparece no list
    
exemplo: `terraform taint aws_instance.web[0]`

`terraform plan -out plan`

`untaint` desmarca o recurso

### Comando graph

terraform.io/docs/commands/graph.html

Pode ser usado pelo GraphViz

`terraform graph`

`apk -U add graphviz`

`terraform graph | dot -Tsvg > graph.svg`

### Comando fmt (format)

terraform.io/docs/commands/fmt.html

Ele reescreve os arquivos para colocar na formatação correta. 

`terraform fmt [option] [dir_com_os_tf]`

`echo $?` mostra o código de saída do último comando executado. Serve para ver se deu certo. 0 significa que deu certo.

`terraform fmt -check` # `-diff` mostra a diferença

`echo $?` # saída 0

### Validate

terraform.io/docs/commands/validate.html

Serve para CI

`terraform validate -json`

## Aula 4

### Condições

https://www.terraform.io/language/expressions/operators

Ordem de como vão ser executados:

1. !, - (multiplication by -1)
2. *, /, %
3. +, - (subtraction)
4. >, >=, <, <=
5. ==, !=
6. &&
7. ||

A condição é escrita com operador ternário

`condição ? verdadeiro : falso`

```
valiable "environment"{
  default = "staging"
}

variable "plus" {
  default = 2
}

variable "production" {
  default = true
}
```

```
resource "aws_instance" "web" {
  # count = var.environment == "production" ? 2 + var.plus : 1
  count = var.production ? 2 : 1
  instance_type = count.index < 1 ? "t2.micro" : "t3.medium" # 1a micro e segunda medium
  # ...
}
```

### Type constraints | restrições por tipo

terraform.io/docs/configurations/types.html#collection-types

Tipos primitivos: string, number, bool.

O terraform converte automaticamente para string e de sring para os outros tipos caso seja necessário. Isso funciona apenas para tipos primitivos.

Tipos complexos:

- Collection types: list(string|number|bool), object, tuple

```
resource "aws_instance" "web" {
 # ...
 vpc_security_group_ids = var.sg
 tags = {
   Name = "HelloWorld"
   Env = var.environment
 }
}
```

```
variable "environment" {
  type        = string
  default     = "staging"
  description = "The environment of instance"
}

variable "sg" {
  type        = list(number)
  default     = [1,2,3,4]
  description = "The list of sg for this instance"
}
```

`export TF_VAR_sg=[10,20,30]` 

### Laço for_each

Ele só aceita map e set de strings

```
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
  #for_each = {
  #  dev = "t2.micro"
  #  staging = "t3.medium"
  #}
  for_each = toset(var.instance_type)
  instance_type = each.value
}

variable "instance_type" {
  type = list(string)
  default = ["t2.micro", "t3.medium"]
  description = "The list of instance type"
}

output "ip_address" {
  value = {
    for instance in aws_instance.web:
    instance.id => instance.private_ip
  }
}
```

## Dia 05

### Bloco dinâmico

É definido com um set, no caso o `ebs_block_device`. A ideia é pegar um bloco aninhado para iterar

```
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  
  dynamic "ebs_block_device" {
    for_each = var.blocks
    content {
      device_name = ebs_block_device.value["device_name"]
      volume_size = ebs_block_device.value["volume_size"]
      volume_type = ebs_block_device.value["volume_type"]
    }
  }

  tags = {
    Name = "HelloWorld"
  }
}
```

```
# terraform.tfvars
blocks = [
{
  device_name = "/dev/sdg"
  volume_size = 5
  volume_type = "gp2"
},
{
  device_name = "/dev/sdh"
  volume_size = 10
  volume_type = "gp2"
}
]
```

```
# variable.tf
variable "blocks" {
  type = list(object({
    device_name = string
    volume_size = string
    volume_type = string
  }))
  description = "List of EBS Block"
}
```

### String template

```
variable "name" {
  type = string
  default = "testando interpolação"
  description = "Nome do helloworld"
}
```

```
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name = "HelloWorld %{ if var.name == "Alynne"}${var.name}%{ else }não valeu%{ endif }!"
  }
}
```

