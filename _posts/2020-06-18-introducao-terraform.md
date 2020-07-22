---
title: Introdução ao Terraform
tags: [introdução terraform, terraform, aws, pt-br]
style: fill
color: primary
description: Neste primeiro post da série Terraform vamos entrar de cabeça no mundo de infrastrutura como código e entender para que serve o Terraform, onde ele habita, do que ele se alimenta, etc.
---

{% include elements/figure.html image="/assets/public/introducao-terraform-logo.svg" %}

Antes de começar propriamente a falar de Terraform, gostaria primeiro de ressaltar que eu pretendo escrever uma série de artigos sobre este tema, desde instação e configuração do Terraform CLI até tópicos mais avançados como criação e publicação de módulos usando o Terraform Registry. Por isso, se você está lendo este post e se interessa por este assunto, deixa uma mensagem nos comentários. 

## Motivação

Para ser sincero, a [documentação disponível](https://www.terraform.io/docs/index.html) de Terraform é muito ampla e fácil de utilizar, por isso não seria útil somente traduzir o material disponível para o português, mas sim trazer exemplos de como podemos resolver problemas reais usando essa ferramenta.

Terraform, como veremos mais a frente, é uma ferramenta que usamos para definir infrastrutura de forma declarativa, ou seja, através de código e que nós, humanos, entendemos. Atualmente existe uma série de [provedores](https://www.terraform.io/docs/providers/index.html) suportados pela ferramenta, a lista é realmente longa e dentre eles destaco: AWS, GCP, Microsoft Azure etc. 

Como possuo experiência trabalhando com Amazon Web Services (AWS), daqui por diante os exemplos e cenários serão voltados para este provedor, e talvez aqui vale a primeira dica importante: {% include elements/highlight.html text="Terraform nos ajuda a criar, gerenciar e atualizar infrastrutura usando código, porém cabe a nós conhecermos os detalhes e recursos do provedor onde estamos criando essa infrastrutura" %}, por exemplo, uma VM (Virtual Machine) na GCP `google_compute_instance` terá parâmetros e ciclo de vida diferente de uma VM na AWS `ec2_instance`, este tipo de particularidade que temos que entender para trabalhar de forma efetiva com Terraform.

Por fim, outra parte que considero importante no ciclo de aprendizagem é compartilhar o que sabemos e aprendemos, registrar soluções que aplicamos de forma sucessitiva no passado, assim como acessar essas informações rapidamente no futuro.

Vamos para o artigo!

## Introdução

Terraform é uma ferramenta de código aberto que nos ajuda a criar e manter infraestrutura através de código, pertence a empresa HashiCorp, no qual também oferece Terraform como serviço na Cloud (SaaS). O core é mantido por uma grande comunidade. Para escrever código em Terraform utilizamos uma linguagem declarativa chamada HCL (HashCorp Language).

Abaixo temos um diagrama de alto nível demonstrando de forma didática os componentes envolvidos quando trabalhamos com Terraform e AWS.

{% include elements/figure.html image="/assets/public/introducao-terraform-basic-diagram.svg" %}

### Usuários

Terraform provê uma linguagem de alto nível e de fácil aprendizado, os desenvolvedores escrevem a configuração necessária para criar e manter infraestrutura em diferentes provedores (AWS, Google, Kubernetes) e isso fica armazenado em um arquivo de estado da infraestrutura, veremos em outro post detalhadamente como estado funciona e como manipulá-lo. Times que trabalham em um mesmo projeto geralmente irão utilizar uma ferramenta de controle de versão, como o github, para compartilhar o código através de um repositório.

### Terraform

A partir dos arquivos de configuração criados pelos usuários, Terraform saberá quais componentes deverão ser criados ou alterados, para isso, Terraform cria um grafo de todos os componentes de infraestrutura e armazena em um arquivo de estado, dessa forma Terraform consegue construir a infraestrutura da maneira mais eficiente possível uma vez que, recursos que não possuem dependência em comum serão criados ou modificados de forma paralela, na prática se estamos criando uma VPC na AWS e associando alguns Security groups, Terraform saberá que a configuração necessária para criar os security groups dependem da VPC e, dessa forma, criará a VPC em primeiro lugar e depois os security groups.

### AWS CLI

Terraforma, por padrão, não sabe como criar recursos em todos os provedores, por isso utiliza as APIs disponibilizadas pelos próprios provedores para interagir com eles, tomando como exemplo a AWS, ela provê uma CLI (Command line interface) de código aberto, que pode ser acessado aqui, na qual podemos, via linha de comando, interagir com os serviços disponíveis sem precisar utilizar a interface Web, é dessa forma que Terraform sabe como criar os recursos que precisamos e possui inteligência de criar esses recursos de forma paralela quando não existe dependência entre eles, veremos nos capítulos seguintes como configuramos este acesso utilizando o recurso provider do Terraform. 

### IAM Roles

Na AWS utilizamos o serviço IAM (Identity and Access Management), como próprio nome sugere, para gerenciar o que usuários e serviços e podem fazer, com IAM podemos criar usuários e associar políticas (Policy) a eles, podemos criar funções (Roles) e atribuir à serviços. Para que o Terraform funcione de forma correta, precisamos conceder acesso a todos os serviços da AWS que serão gerenciados por ele, veremos na prática as abordagens que podemos utilizar para essa configuração e boas práticas nessa utilização.

### AWS Services

Provedor de recursos em Núvem da Amazon, chamada AWS (Amazon Web Services), é o pioneiro neste seguimento e um dos mais reelevantes player no mercado, na data que escrevo este post, AWS possui mais de 200 serviços disponíveis para uso, utilizaremos Terraform para provisionar alguns deles nesta série de posts. 

## Terraform x Outras Abordagens

Atualmente temos diversas abordagens para criar código como infrastrutura, por exemplo, usando o próprio serviço da AWS chamado [Cloudformation](https://aws.amazon.com/pt/cloudformation/), uma desvantagem de se utilizar Cloudformation é que este é um serviço da Amazon, se no futuro migrarmos nossa implementação para outro provedor teremos que reescrever o código novamente. Se utilizarmos Terraform e pensarmos em nossa infrastrutura em forma de módulos (Veremos sobre módulos em outro post) é possível com pouco esforço migrar a nossa infrastrutura para outro provedor sem maiores esforços. 

Outra forma é utilizar a biblioteca [Boto](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) disponível para python, esta biblioteca disponibiliza APIs de baixo nível para gerenciamento de recursos da AWS, contúdo, também ficamos presos ao provedor AWS.

Quando buscamos por infrastrutura como código ainda é possível encontrar referências de outras ferramentas como por exemplo Puppet, Chef e Ansible, aqui vale lembrar uma diferença importante: {% include elements/highlight.html text="Puppet, Chef, Ansible e outras ferramentas deste seguimento são conhecidas como Configuration Management tools e são utilizadas para instalar e gerenciar software em servidores existentes, quando falamos de Terraform e Cloudformation, estamos nos referindo a Provisioning tools, ou seja, ferramentas para provisionar servidores e recursos de infrastrutura em geral." %}

## Vantagens 

Algumas vantagens que eu percebo no dia-a-dia ao trabalhar com Terraform:

- A linguagem HCL é fácil de se entender e utilizar, por exemplo, o script abaixo cria um Bucket S3:

```tf
resource "aws_s3_bucket" "b" {
  bucket = "my-tf-test-bucket"
  acl    = "private"

  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}
```

- Workflow do Terraform é simples, escrever configuração, planejar e aplicar e o ciclo se repete.
- A documentação da infrastrutura passa ser o próprio código, e o código por sua vez é versionado, dessa forma podemos traçar todas as iterações criadas em nossa infrastrutura;
- Utilizando Terraform também é possível provisionar e gerenciar clusteres de Kubernetes (Utilizando Amazon EKS ou próprio projeto Kops com EC2) ou até mesmo Puppet, como falado previamente neste post, a lista de provedores é imensa, dessa forma, conseguimos de forma centralizada provisionar e gerenciar nossa infrastrutura utilizando uma só ferramenta;
- É possível armazenar o estado da infrastrutura em backends em Buckets (AWS S3), banco de dados, etc. Ainda é possível criar mecanismos de Lock para evitar que dois membros do time alterem o mesmo recurso de infrastrutura em um mesmo momento.

## Conclusão

Vejo o Terraform como uma ferramenta indispensável quando o assunto é infrastrutura como código, o projeto tem uma comunidade bem ativa e novos recursos são lançados com frequência, com a adoção do Terraform é possível documentar toda a infrastrutura através do código e se torna uma tarefa fácil trabalhar em time uma vez que o código fica em um repositório remoto e o estado da infrastrutura fica separado deste repositório. 

Bom pessoal, é isso, no próximo post vamos começar a botar a mão na massa e configurar nosso projeto com Terraform e AWS na prática.
