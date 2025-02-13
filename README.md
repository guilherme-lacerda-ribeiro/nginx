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

## Configuração Inicial
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

## Proxy Reverso
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

## Microserviços
- Cada aplicação cuida da sua parte (financeira, acadêmica, streaming de vídeos, etc). Organizado por pequenos (micro) serviços. [Curso](https://cursos.alura.com.br/course/microsservicos-padroes-projeto).
- url é a página estática do site principal institucional, responde rapidamente no próprio nginx
- url/servico1 é uma aplicação, em um servidor, próprio
- url/servico2 é outra aplicação ou serviço, em outro local, por exemplo.

### Serviços 1 e 2
Arquivo services.conf por exemplo.
```nginx
server {
  listen 8001;
  server_name localhost;

  location / {
    root C:/www/servico1;
    index index.html;
  }
}

server {
  listen 8002;
  server_name localhost;

  location / {
    root C:/www/servico2;
    index index.html;
  }
}
```

### Proxy reverso
- localhost/servico1 vai pra porta 8001
- localhost/servico2 vai pra porta 8002

```nginx
#nginx.conf por exemplo
(...)

location /servico1 {
  # / depois da porta : vai mandar só o que receber a partir do /servico1
  # localhost/servico1/teste vira localhost:8001/teste
  proxy_pass http://localhost:8001/;
  
  # sem a barra ele adiciona o que está no location no final
  # localhost/servico1/teste vira localhost:8001/servico1/teste
  # proxy_pass http://localhost:8001;  
}

location /servico2 {
  proxy_pass http://localhost:8002/;
}
```

### API Gateway
[Artigo](https://www.f5.com/company/blog/nginx/deploying-nginx-plus-as-an-api-gateway-part-1)
- conceito usado nesta implementação, só redireciona
- ao invés de cada cliente conversar um com o outro livremente, cria-se esse intermediador para facilitar manutenção por exemplo e poder aplicar regras específicas
- gera uma fachada com url amigável
- atentar porque pode se tornar um ponto central de falha, avaliar HA (high availability)
- posso redirecionar ou proibir
- uso do _Decorator_ para adicionar informações ao request ou para remover do response.
  - cache
  - compressão
  - cabeçalhos na ida
  - cabeçalhos na volta


## Upstream (load balance)
Servidor final está recebendo mais requisições do que consegue suportar.
- localhost:8003 vai mandar para o servico1 ou servico2
- considera que eles tenham a mesma aplicação, o mesmo código, ao fazer o build manda para os dois servidores.
- há diversos algoritmos de balanceamento, round robin por exemplo, mas o modo mais rudimentar é este exemplificado.
```nginx
upstream servicos {
  # ora vai cair num servidor, ora no outro
  server localhost:8001;
  server localhost:8002;
}

server {
  listen 8003;
  server_name localhost;

  location / {
    proxy_pass http://servicos;
  }

}
```

### Pesos (weight)
Servidores podem ter capacidades de hardware diferentes, para contornar isso pode-se colocar pesos nos servidores. Não informar, assume peso 1.
```nginx
upstream servicos {
  # a cada 7 requisições, 5 irão para o 1 enquanto 2 para o server 2
  server localhost:8001 weight=5;
  server localhost:8002 weight=2;
}
```


## Logs
https://nginx.org/en/docs/http/ngx_http_log_module.html
- `access_log` - todo acesso
- `error_log` - log apenas dos erros
- descomenta o log_format
- o caminho para os logs deve existir (ele não vai criar o diretório) e o usuário do nginx deve ter permissão de escrita (atributo user dentro do nginx.conf).

```nginx
server {
  listen 8001;
  server_name localhost;
  access_log C:/logs/servico1.log;

  location / {
    root C:/www/servico1;
    index index.html;
  }
}

server {
  listen 8002;
  server_name localhost;
  access_log C:/logs/servico2.log;

  location / {
    root C:/www/servico2;
    index index.html;
  }
}
```

### Log format
```nginx
log_format main 'Remote: $remote_addr, Time: $time_local, '
                'Request: "$request", Status: $status ';
(...)

access_log C:/logs/servico1.log main;
```

### IP Real de quem fez a requisição
O log do servico1 e servico2 ficaria indicaria o IP do load balancer, o que não é bom.
- usuário acessa o serviço (porta 8003)
- o load balancer redireciona para o service1 ou service2
- o IP fica do load balancer e não do usuário

No load-balance.conf (por exemplo) eu adiciono o cabeçalho http. Para cabeçalhos personalizados a especificação sugere incluir **X-** no nome.
```nginx
upstream servicos {
  # ora vai cair num servidor, ora no outro
  server localhost:8001;
  server localhost:8002;
}

server {
  listen 8003;
  server_name localhost;

  location / {
    proxy_pass http://servicos;
    proxy_set_header X-Real-IP $remote_addr;
  }

}
```

E então no log_format eu adiciono o cabeçalho criado. Eu troco o '-' por '_' e posso pegar qualquer cabeçalho.
```nginx
log_format main 'Remote: $http_x_real_ip, Time: $time_local, '
                'Request: "$request", Status: $status ';
```

Desta forma, o IP real da requisição foi incluído em um cabeçalho próprio, para poder ser recuperado posteriormente.
