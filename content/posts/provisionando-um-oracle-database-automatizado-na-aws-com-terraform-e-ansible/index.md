---
title: "Provisionando um Oracle Database em EC2 (AWS) com Terraform e Ansible"
date: 2025-11-10T07:00:00-03:00
draft: false
categories: ["Cloud", "Oracle DBA", "Scripts & Automação"]
tags: ["Cloud", "Oracle", "Script Oracle"]
cover:
  image: "ansible_terraform_aws.png"
---

O objetivo deste projeto foi criar um ambiente Oracle Database totalmente automatizado na AWS, utilizando **Terraform** para provisionar a infraestrutura e **Ansible** para configurar e instalar o banco de dados.  
A ideia foi entregar um banco Oracle pronto para uso **sem abrir o console da AWS**, **sem executar comandos manuais** e **com total reprodutibilidade**.

Esse tipo de automação é o que diferencia um DBA tradicional de um **Database Engineer**, capaz de reconstruir todo o ambiente de banco em minutos a partir de código versionado.

## Estrutura do projeto

A estrutura base do repositório foi dividida em duas camadas:

```text
projeto-oracle/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── ansible/
    ├── inventory.ini
    └── playbook-oracle.yml
```

A camada **Terraform** é responsável por criar a infraestrutura na AWS (EC2, Security Group, chave SSH).  
A camada **Ansible** conecta na instância criada e instala o Oracle Database em container Docker.

## Etapa 1 - Criando a chave pública para o Terraform

O primeiro passo foi gerar um par de chaves SSH para acesso à EC2.  
Essa chave será usada tanto pelo Terraform (para criar o recurso) quanto pelo Ansible (para conectar via SSH).

No Linux:

```text
ssh-keygen -t rsa -b 4096
```

Pressione **Enter** para aceitar os valores padrão.  
Isso criará dois arquivos:

```text
~/.ssh/id_rsa        # chave privada
~/.ssh/id_rsa.pub    # chave pública
```

O Terraform vai usar o conteúdo do `.pub` para criar o **Key Pair** dentro da AWS.

## Etapa 2 - Criando a infraestrutura com Terraform

Dentro da pasta `terraform/`, os arquivos foram definidos da seguinte forma.

### **main.tf**

```text
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.18.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Pega a VPC default
data "aws_vpc" "default" {
  default = true
}

# Pega as subnets da VPC default
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Cria o Key Pair usando a chave pública local
resource "aws_key_pair" "sshkey" {
  key_name   = "terraform-key"
  public_key = file(var.public_key_path)
}

# Cria o Security Group liberando SSH e Oracle
resource "aws_security_group" "oracle_sg" {
  name        = "seu-sg-oracle"
  description = "Acesso SSH e Oracle"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Oracle Listener"
    from_port   = 1521
    to_port     = 1521
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Cria a instância EC2
resource "aws_instance" "oracle_ec2" {
  ami                    = var.instance_ami
  instance_type          = var.instance_type
  subnet_id              = data.aws_subnets.default.ids[0]
  vpc_security_group_ids = [aws_security_group.oracle_sg.id]
  key_name               = aws_key_pair.sshkey.key_name

  tags = {
    Name = "oracle-db"
  }
}
```

### **variables.tf**

```text
variable "aws_region" {
  description = "Região AWS"
  default     = "sa-east-1"
}

variable "public_key_path" {
  description = "Caminho da chave pública SSH"
  type        = string
  default     = "~/.ssh/id_rsa.pub"
}

variable "instance_ami" {
  description = "AMI base (Amazon Linux 2023)"
  type        = string
  default     = "ami-00311490973bb9f7c"
}

variable "instance_type" {
  description = "Tipo da instância EC2"
  type        = string
  default     = "m7i-flex.large"
}
```

### **outputs.tf**

```text
output "ec2_public_ip" {
  description = "IP público da EC2"
  value       = aws_instance.oracle_ec2.public_ip
}
```

### **Executando o Terraform**

```text
cd terraform
terraform init
terraform apply
```

Em poucos minutos, o Terraform cria a EC2, a chave SSH e o Security Group.  
Ao final, o IP público da instância é exibido:

```text
ec2_public_ip = "18.230.xxx.xxx"
```

Esse IP será usado no Ansible.

## Etapa 3 - Preparando o Ansible

Na pasta `ansible/`, o primeiro arquivo define o inventário:

### **inventory.ini**

```text
[oracle]
18.230.xxx.xxx ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/id_rsa
```

O `ansible_user` é `ec2-user` porque o SO base é Amazon Linux.

## Etapa 4 - Instalando e configurando o Oracle Database

Como o Amazon Linux 2023 não aceita o pacote RPM nativo do Oracle XE, a instalação foi feita em **container Docker**, usando a imagem oficial `gvenzl/oracle-xe`.

Essa abordagem é prática, limpa e totalmente automatizável.

### **playbook-oracle.yml**

```text
---
- name: Instalar Oracle XE via container
  hosts: oracle
  become: yes
  tasks:
    - name: Instala Docker
      dnf:
        name: docker
        state: present

    - name: Habilita e inicia Docker
      systemd:
        name: docker
        state: started
        enabled: true

    - name: Baixa imagem Oracle XE
      command: docker pull gvenzl/oracle-xe

    - name: Sobe container Oracle XE
      command: >
        docker run -d --name oracle-xe
        -p 1521:1521 -p 5500:5500
        -e ORACLE_PASSWORD=SenhaForte123
        gvenzl/oracle-xe
```

### **Executando o Ansible**

```text
cd ansible
ansible-playbook -i inventory.ini playbook-oracle.yml
```

O Ansible conecta na EC2, instala o Docker, baixa a imagem do Oracle e sobe o container.  
Ao final, o Oracle XE estará rodando na porta 1521 e pronto para conexões externas.

## Etapa 5 - Testando o ambiente

Para testar a conexão, utilizei o **Oracle Instant Client** no meu ambiente local.  
Depois de instalado, a conexão é feita via SQL*Plus:

```text
sqlplus system/SenhaForte123@//18.230.xxx.xxx:1521/XEPDB1
```

O banco responde normalmente, com o listener configurado automaticamente pelo container.

## Entendendo o fluxo técnico completo

1. **Terraform**
   - Provisiona a infraestrutura na AWS.
   - Cria a EC2, Security Group e chave SSH.
   - Retorna o IP público.
2. **Ansible**
   - Conecta via SSH na EC2 criada.
   - Instala o Docker.
   - Baixa e inicia o container Oracle XE.
3. **Oracle Database**
   - Inicializa automaticamente dentro do container.
   - Listener ativo na porta 1521.
   - Banco acessível via SQL*Plus de qualquer cliente externo.
4. **Infraestrutura reversível**
   - `terraform destroy` remove toda a infraestrutura.
   - Nenhum resquício fica na conta AWS.

## Resultados e considerações

Com esse projeto, o ambiente completo - da criação da EC2 até o banco Oracle funcional - é entregue em aproximadamente 7 minutos.

Além de servir como laboratório, essa estrutura pode evoluir facilmente para:

- Ambientes multi-cloud (AWS + GCP).
- Criação automática de Data Guard.
- Ambientes de teste e CI/CD de banco de dados.
- Recriação automatizada de instâncias Oracle para desenvolvimento e treinamento.

Esse tipo de automação é o que garante **consistência, rastreabilidade e velocidade** em times modernos de banco de dados.

A principal lição aqui é que **tudo pode ser tratado como código**:  
infraestrutura, configuração e até mesmo o banco de dados.

Com duas ferramentas, Terraform e Ansible, foi possível criar, configurar e disponibilizar um Oracle Database totalmente funcional, sem qualquer interação manual.

![](exterminador-683x1024.png)
