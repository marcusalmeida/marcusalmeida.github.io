---
layout: post
title:  "Desenvolvendo com Django & Docker Compose"
date:   2016-08-26 12:00:00
categories: desenvolvimento
tags: [dev,django,docker,python]
quote_text: "Uma longa viagem começa com um único passo."
quote_author: "Lao Tse"
---

Nesse post vamos demonstrar como podemos configurar um ambiente de desenvolvimento Python/Django utilizando Docker Compose, de forma prática e direta.

Pré-requisitos:

1.  Docker Engine instalado - [Como instalar](https://docs.docker.com/engine/installation/){:target="_blank"} 
2.  Docker Compose instalado - [Como instalar](https://docs.docker.com/compose/install/){:target="_blank"}

Com os pré-requisitos instalados, podemos iniciar criando o diretório do projeto.

{% highlight shell %}
  $ mkdir django-sandbox && cd $_ 
{% endhighlight %}

Precisamos criar alguns arquivos para configurar o ambiente.

{% highlight shell %}
  $ touch Dockerfile docker-compose.yml requirements.txt 
{% endhighlight %}

O **Dockerfile** define o conteúdo da imagem do container que será utilizado. Através de um conjunto de instruções, criamos um script de construção da imagem do container.

Adicione o seguinte conteúdo ao arquivo **Dockerfile**.

{% highlight plaintext %}
# Container base: python 3.6 Alpine Linux
FROM python:3.6-alpine

ENV PYTHONUNBUFFERED 1

# Cria diretório onde vão ficar os fontes
RUN mkdir /code

# Define o diretório de trabalho
WORKDIR /code

# "Copia" arquivo requirements.txt para o diretorio code
ADD requirements.txt /code/

# Executa o pip
RUN pip install -r requirements.txt

# "Copia" os arquivos locais para o diretorio code no container 
ADD . /code/        
{% endhighlight %}

Esse **Dockerfile** utiliza o Python 3.6 Alpine Linux como imagem base, mas pode ser utilizada qualquer imagem base Python no [Docker Hub](https://hub.docker.com/_/python/){:target="_blank"}.  Para mais informações sobre Dockerfiles, veja a [documentação de referência](https://docs.docker.com/engine/reference/builder/){:target="_blank"}

Para gerenciar as dependências do projeto utilizamos um arquivo **requirements.txt**. 
Esse arquivo será utilizado no Dockerfile na construção da imagem do container.

Inicialmente adicionamos somente o seguinte conteúdo ao **requirements.txt**, para instalar o Django ao projeto.

{% highlight plaintext %}
Django
{% endhighlight %}

Vamos configurar o arquivo **docker-compose.yml**. Esse arquivo descreve os serviços que vão compor o stack do projeto.
Neste exemplo, declaramos o serviço do servidor web. O **docker-compose.yml** também descreve qual imagem Docker o serviço vai utilizar, que portas devem ser expostas e os volumes que devem ser montados dentro do container.
Para mais informações sobre **docker-compose.yml**, veja a [documentação de referência](https://docs.docker.com/compose/compose-file/){:target="_blank"}.

Adicione o seguinte conteúdo ao arquivo **docker-compose.yml**

{% highlight plaintext %}
version: '2'
services:
    web:
        build: . 
        image: django-sandbox:latest
        command: python manage.py runserver 0.0.0.0:8000
        volumes:
            - .:/code
        ports:
            - "8000:8000"
{% endhighlight %}

Estamos configurando o serviço **web** que será iniciado pelo container, *"buildado"* pelo **Dockerfile** declarado anteriormente.
Neste container será montado o volume `/code` com o conteúdo do diretório local do projeto.
Quando o container for iniciado, será executado o comando `python manage.py runserver 0.0.0.0:8000`. 
Teremos um webserver "ouvindo" na porta 8000 no container, para redirecionar o tráfego para a porta 8000 no host local utilizamos a instrução "ports" no arquivo **docker-compose.yml**. 

Com isso temos todos as configurações prontas. Salve todos os arquivos. 

Depois de toda essas configurações, vamos a parte boa.

## Criando o projeto Django

A partir daqui, vamos *"buildar"* a imagem do container e criar o projeto.

Para criar o projeto Django usando o comando *docker-compose* execute o seguinte.
{% highlight shell %}
  $ docker-compose run web django-admin.py startproject mysite .
{% endhighlight %}

Com esse comando o *Compose* tenta executar `django-admin.py startproject mysite` no container, que contém o serviço **web**, usando a imagem `django-sandbox:latest`. 
Como a imagem ainda não existe, o *Compose* constrói esta imagem a partir do diretório atual utilizando o **Dockerfile**, como especificado na linha `build: .` do arquivo **docker-compose.yml**.

Após a construção da imagem do container para o serviço **web**, o *Compose* inicia o serviço e executa o comando `django-admin.py startproject mysite` no container do serviço **web**.

Após o comando `docker-compose` finalizar, liste os arquivos no diretório do projeto:

{% highlight shell %}
$ ls -l
  drwxr-xr-x 2 root   root   mysite
  -rw-rw-r-- 1 user   user   docker-compose.yml
  -rw-rw-r-- 1 user   user   Dockerfile
  -rwxr-xr-x 1 root   root   manage.py
  -rw-rw-r-- 1 user   user   requirements.txt
{% endhighlight %}

##### * Se estiver executando o Docker no Linux será necessário trocar o dono do diretório com `$ sudo chown -R $USER:$USER .`


Agora vamos executar o comando `docker-compose up` para iniciar os componentes da stack do projeto.

{% highlight shell %}
  $ docker-compose up
  Starting djangosandbox_web_1
  Attaching to djangosandbox_web_1
  web_1  | Performing system checks...
  web_1  | 
  web_1  | System check identified no issues (0 silenced).
  web_1  | 
  web_1  | You have 13 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
  web_1  | Run 'python manage.py migrate' to apply them.
  web_1  | August 27, 2016 - 23:27:40
  web_1  | Django version 1.10, using settings 'mysite.settings'
  web_1  | Starting development server at http://0.0.0.0:8000/
  web_1  | Quit the server with CONTROL-C.
{% endhighlight %}

Neste ponto, finalmente, você deve ter a app Django executando na porta 8000. Acesse [http://localhost:8000](http://localhost:8000){:target="_blank"}

![Django It Worked]({{ "images/pngs/django-it-worked.png" | prepend: site.baseurl }})

### Executando outros comandos do Django dentro do container.

Caso queria executar os comandos que o Django oferece dentro do container.

Por exemplo, para criar uma nova app. Execute em outro shell na pasta do projeto o seguinte comando.
{% highlight shell %}
$ docker-compose exec web python manage.py startapp polls
{% endhighlight %}

Para executar as migrações. 
{% highlight shell %}
$ docker-compose exec web python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Rendering model states... DONE
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying sessions.0001_initial... OK
{% endhighlight %}

##### * Uma dica é criar uma alias na sessão do shell para o comando `docker-compose exec web python manage.py` com: 

#####  `alias pm='docker-compose exec web python manage.py'`

### Comandos úteis do Docker Compose
Para verificar o status dos containers:
{% highlight shell %}
$ docker-compose ps
       Name                 Command                 State         Ports          
----------------------------------------------------------------------------------
djangosandbox_web_1  python manage.py runserver ...  Up     0.0.0.0:8000->8000/tcp 
-
{% endhighlight %}

Para parar os containers:
{% highlight shell %}
$ docker-compose stop
Stopping djangosandbox_web_1 ... done
-
{% endhighlight %}

Para reinicar os containers:
{% highlight shell %}
$ docker-compose restart
Restarting djangosandbox_web_1 ... done
-
{% endhighlight %}

Para parar e destruir toda a stack de containers:
{% highlight shell %}
$ docker-compose down
Stopping djangosandbox_web_1 ... done
Removing djangosandbox_web_1 ... done
Removing djangosandbox_web_run_1 ... done
Removing network djangosandbox_default
-
{% endhighlight %}

Neste post vimos que podemos configurar um ambiente simples de desenvolvimento Django isolado e que posteriormente pode ser *"deployiado"* em alguma plataforma que suporta containers Docker.

Ainda podemos utilizar o *Compose* para adicionar outros componentes a stack do projeto como: banco de dados (SQL ou NoSQL), cache (p.ex. Memcached) e etc. Em um outro post podemos abordar como realizar essas configurações.

A proposta aqui era apresentar de forma prática com podemos utilizar o Docker como alternativa a outras soluções como [Vagrant](https://www.vagrantup.com/) para configuração de um ambiente de desenvolvimento.

*sóisso!*