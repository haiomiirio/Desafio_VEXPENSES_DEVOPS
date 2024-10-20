# Desafio_VEXPENSES_DEVOPS
DESAFIO ONLINE
  1° Descrição Técnica:<br><br>

1 Provedor AWS: Define a região us-east-1 para a criação dos recursos.

2 Variáveis: Duas variáveis (projeto e candidato) permitem personalizar os nomes dos recursos, facilitando a identificação.

3 Chave Privada TLS: Uma chave privada é gerada para segurança e autenticação.

4 IAM Role: Cria um papel de IAM que a instância EC2 poderá assumir, garantindo permissões específicas.

5 IAM Instance Profile: Associa o papel de IAM à instância EC2 para que ela utilize essas permissões.

6 VPC: Cria uma VPC, que é uma rede privada, para isolar os recursos e melhorar a segurança.

7 Sub-rede: Dentro da VPC, é criada uma subnet onde a instância EC2 será alocada.

8 Gateway de Internet: Permite que a VPC se conecte à Internet, permitindo acesso externo aos recursos.

9 Tabela de Roteamento: Direciona o tráfego da subnet para o gateway de Internet, garantindo conectividade.

10 Grupo de Segurança: Controla o acesso à instância EC2, permitindo conexões SSH apenas de um IP confiável.

11 AMI: Busca a imagem mais recente do Debian 12 para garantir que a instância seja provisionada com uma versão atualizada do sistema operacional.

12 Instância EC2: Cria uma instância EC2 do tipo t2.micro, equipada com Nginx, que é instalado automaticamente ao iniciar a instância.

13 Logs de Fluxo da VPC: Ativa logs para monitorar o tráfego de rede, permitindo análises detalhadas sobre o que acontece na VPC.

14 Grupo de Logs do CloudWatch: Cria um grupo de logs no CloudWatch para armazenar os logs de fluxo da VPC <br> <br>

2° Descrição Técnica:<br><br>

1.	Restringir Acesso SSH<br>
	O código atualmente permite acesso SSH de qualquer lugar. Vamos restringir esse acesso a um IP confiável.<br>
2.	Habilitar Criptografia no Volume EBS<br>
ativar a criptografia no volume principal da instância EC2 para proteger os dados.<br>
3.	Configurar Logs de Tráfego (VPC Flow Logs)<br>
adicionar logs de tráfego para monitorar o fluxo de rede dentro da VPC.<br>
4.	Automatizar a Instalação do Nginx<br>
o	Adicionar um script para instalar e iniciar o Nginx automaticamente quando a instância EC2 for criada.<br>
5.	Usar IAM Role em vez de Key Pair<br>
o	Configurar uma IAM Role para a instância EC2, removendo o uso de chaves SSH.<br>





Instruções para Usar o Terraform com a AWS<br>
Pré-requisitos:<br>

Instale o Terraform: Baixe e instale a versão mais recente do site do Terraform.<br>
Crie uma Conta na AWS: Se ainda não tiver, crie uma conta na AWS e obtenha suas credenciais de acesso (Access Key e Secret Key).<br>
Configure o Ambiente: Configure suas credenciais da AWS em ~/.aws/credentials ou defina variáveis de ambiente.<br>
Dica de Segurança:<br>

Evite Usar a Conta Root da AWS: Sempre crie usuários com permissões específicas em vez de usar a conta root. Utilize o AWS Identity and Access Management (IAM) para gerenciar permissões e atribuir apenas as permissões necessárias a cada usuário.<br>
Passos para Inicializar e Aplicar a Configuração<br>
Crie uma Pasta do Projeto:<br>

Digite o seguinte comando:<br>
bash<br>

mkdir meu-projeto-terraform<br>
cd meu-projeto-terraform<br>
Crie um Arquivo de Configuração:<br>

Crie um arquivo chamado main.tf e adicione o seguinte código:<br>
<br><br><br>



hcl

provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string  
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_iam_role" "ec2_role" {
  name = "${var.projeto}-${var.candidato}-role"

  assume_role_policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Action" : "sts:AssumeRole",
        "Principal" : {
          "Service" : "ec2.amazonaws.com"
        },
        "Effect" : "Allow",
        "Sid" : ""
      }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2_instance_profile" {
  name = "${var.projeto}-${var.candidato}-instance-profile"
  role = aws_iam_role.ec2_role.name
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH e tráfego necessário"
  vpc_id      = aws_vpc.main_vpc.id

  
  ingress {
    description      = "Allow SSH from trusted IP"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["seu_IP/32"]  # Substituir pelo IP confiável
    ipv6_cidr_blocks = []
  }


  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true
  iam_instance_profile        = aws_iam_instance_profile.ec2_instance_profile.name

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
    encrypted             = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install nginx -y
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}

resource "aws_flow_log" "vpc_flow_log" {
  log_destination      = aws_cloudwatch_log_group.vpc_flow_logs.arn
  traffic_type         = "ALL"
  vpc_id               = aws_vpc.main_vpc.id
}

resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name = "vpc-flow-logs"
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}

<br><br><br>




Inicialize o Projeto:

Digite o comando:
bash

terraform init
Verifique o Plano de Execução:

Digite o comando:
bash

terraform plan
Aplique a Configuração:

Digite o comando:
bash

terraform apply
Confirme digitando yes.

Verifique os Recursos Criados:

Digite o comando:
bash

terraform show
Remova Recursos (se necessário):

Digite o comando:
bash

terraform destroy
Confirme digitando yes.


Observações Finais

Lembre-se de substituir seu_IP/32 pelo IP que deseja permitir o acesso SSH.
Ajuste o default das variáveis projeto e candidato conforme necessário.



