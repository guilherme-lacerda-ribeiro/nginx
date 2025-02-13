# NGINX
Servidor muito rápido. Se comparado com o Apache inclusive porque o apache cria o processo, trata as requisições, mata o processo, etc. E isso é custoso.

https://nginx.org/

Um proxy comum intermedia as conexões de saída da instituição, fica no lado do cliente. O proxy reverso é exatamente o contrário, o que vier da internet é intermediado por ele antes de encaminhar para os servidores internos da instituição.

## Conceitos
### Processos
- Master process administra os worker process
- Worker process normalmente é um para cada núcleo de processador do computador
  - Cada worker process trata inúmeras requisições, multiplexing IO, faz várias coisas de forma assíncrona
  - `worker_processes auto` ou `worker_processes 1` (ou 2, 3, ...)
  - `events { worker_connections 1024; }` quantas _coisas assíncronas_ (file descriptors) cada worker pode processar. Um file descriptor pode ser socket, conexões, arquivos, etc.

### Diretórios
`nginx -h` retorna as informações do nginx e seus diretórios envolvidos
```sh
meu@server ~ $ nginx -h
nginx version: nginx/1.24.0
Usage: nginx [-?hvVtTq] [-s signal] [-p prefix]
             [-e filename] [-c filename] [-g directives]

Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages during configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /etc/nginx/)
  -e filename   : set error log file (default: /var/log/nginx/error.log)
  -c filename   : set configuration file (default: /etc/nginx/nginx.conf)
  -g directives : set global directives out of configuration file

```

## Comandos
`nginx -s reload` - recarrega

## Configuração
### Básico
- http://localhost/ direciona para index.html
- http://localhost/teste.html vai procurar o arquivo teste.html em C:/www.
```nginx
server {
  listen 80;
  server_name localhost;

  location / {
    root C:/www;
    index index.html;
  }
}
```

### Erros
- o arquivo erro.html deve existir no diretório _root_.
```nginx
server {
  listen 80;
  server_name localhost;

  location / {
    root C:/www;
    index index.html;
  }

  error_page 404 403 401 /erro.html
}
```

### Redirecionando requisições
- Tudo o que vier na porta 8080 será encaminhado para a porta 80 (padrão web, logo pode ser omitida).
```nginx
server {
  listen 8080;
  (...)

  location / {
    #proxy_pass http://localhost:80;
    proxy_pass http://localhost;
  }
}
```

### Nginx + php
- `php -S localhost:8000` - considerando que tem um servidor php na máquina
- Neste cenário a ideia é que tudo estático vá para o C:/www e os arquivos que precisam de processamento (php) sejam redirecionados ao servidor de php.
```nginx
server {
  listen 80;
  server_name localhost;

  location / {
    root C:/www;
    index index.html;
  }

  # ~ case sensitive
  # ~* case insensitive
  # \ caracter de escape
  # tudo que for .php
  # $ tem que terminal com .php, não pode ser .php.old ou .php.bkp
  location ~ \.php$ {
    proxy_pass http://localhost:8000;
  }

  error_page 404 403 401 /erro.html
}
```