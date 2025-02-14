# NGINX
Servidor muito rápido. Se comparado com o Apache inclusive porque o apache cria o processo, trata as requisições, mata o processo, etc. E isso é custoso.

https://nginx.org/

Um proxy comum intermedia as conexões de saída da instituição, fica no lado do cliente. O proxy reverso é exatamente o contrário, o que vier da internet é intermediado por ele antes de encaminhar para os servidores internos da instituição.

## Conteúdo
- [NGINX](#nginx)
  - [Conteúdo](#conteúdo)
  - [Conceitos](#conceitos)
    - [Processos](#processos)
    - [Diretórios](#diretórios)
  - [Comandos](#comandos)
  - [Configuração Inicial](#configuração-inicial)
    - [Básico](#básico)
    - [Erros](#erros)
  - [Proxy Reverso](#proxy-reverso)
    - [Redirecionando requisições](#redirecionando-requisições)
    - [Nginx + php](#nginx--php)
  - [Microserviços](#microserviços)
    - [Serviços 1 e 2](#serviços-1-e-2)
    - [Proxy reverso](#proxy-reverso-1)
    - [API Gateway](#api-gateway)
  - [Upstream (load balance)](#upstream-load-balance)
    - [Pesos (weight)](#pesos-weight)
    - [Algoritmos de balanceamento de cargas](#algoritmos-de-balanceamento-de-cargas)
    - [Diretiva Backup](#diretiva-backup)
  - [Logs](#logs)
    - [Log format](#log-format)
    - [IP Real de quem fez a requisição](#ip-real-de-quem-fez-a-requisição)
  - [Fast-CGI ou servidores auto contidos](#fast-cgi-ou-servidores-auto-contidos)
    - [Configurando](#configurando)
  - [Performance](#performance)
    - [Cache Navegador](#cache-navegador)
    - [Cache Public](#cache-public)
    - [Cache Private](#cache-private)
    - [Compressão](#compressão)
    - [Conexões](#conexões)


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
```nginx
upstream servicos {
  # a cada 3 requisições, 2 irão para o 1 enquanto 1 para o server 2
  server localhost:8001 weight=2;
  server localhost:8002;
}
```

### Algoritmos de balanceamento de cargas
- `round-robin` - roda a lista de servidores disponíveis, em cada momento atribuindo a um
- `weighted round-robin` - round robin ponderado, conceito weight
- `least_conn` - Observe: quantidade de requisições que estão chegando e não a quantidade de requisições que estão abertas. Pode acontecer de ter requisições rápidas e outras demoradas. Posso acabar sobrecarregando meu server se fizer apenas um round robin. Ou seja, não é mais requisições mas sim conexões (abertas). [Documentação](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_conn).
  ```nginx
    upstream servicos {
    least_conn;
    server localhost:8001;
    server localhost:8002;
  }
  ```
- `ip_hash` - o IP sempre será direcionado para um mesmo servidor

**Objetivamente**:
- as conexões e as capacidades dos servidores são semelhantes, o padrão (round robin) já atende.
- os servidores tem capacidades diferentes, então weighted round robin.
- as requisições são muito diferentes e demora mais ou menos, least conn atende.
- não se encaixa, [pesquisar as alternativas](https://nginx.org/en/docs/http/ngx_http_upstream_module.html).
- round robin é mais rápido do que least conn porque só armazena qual foi o último enviado.

### Diretiva Backup
Meu servidor principal é o determinado sem diretiva, mas se por acaso ele falhar, o outro assume enquanto o primeiro não voltar. [Doc](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#backup).
```nginx
  upstream servicos {
  server localhost:8001;
  server localhost:8002 backup;
}
```

O NGINX, para saber se está ou não disponível:
- community: ao recarregar o servidor, ele garante que os servidores existem.
  - Se a resposta do servidor falhar, o nginx assume que caiu e de tempo em tempo ([configurável](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#max_fails)) ele testa o servidor pra ver se voltou.
  - `max_fails` padrão é 1 (falhou uma vez, considere que caiu)
  - `fail_timeout` depois de quanto tempo vai tentar enviar outra requisição
    ```nginx
    upstream servicos {
      # depois de 2 minutos tenta outra requisição
      server localhost:8001 fail_timeout=120s;
      server localhost:8002 backup;
    }
    ```
- nginx+ (comercial): diretiva `health_check` verifica automaticamente sem requisição.
  ```nginx
  server {
    listen 8001;
    server_name localhost;

    location / {
      root C:/www/servico1;
      index index.html;
      health_check; # apenas com licença comercial
    }
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

## Fast-CGI ou servidores auto contidos
- `cgi`: Páginas estáticas (html) retornavam sem processamento. Nas demais a requisição chegava, criava-se o processo, ele fazia as consultas aos discos (por exemplo), retornava o processamento que era necessário e devolvia, matando o processo. Ficava lento.
- `fast-cgi`: Não precisa criar um processo por requisição. O gerenciador de fastcgi comunica com o fastcgi e solicita e recebe as demandas. É uma forma de se comunicar, não é específico de linguagem.
  - Cenários mais comuns, algumas requisições por segundo, queries mais simples, a adoção do fast-cgi é recomendada. O PHP por exemplo usa por padrão o PHP-FPM (PHP Fastcgi Process Manager).
  - A vantagem é que após a resposta da requisição, o processo que foi executado é limpo: conexões com o banco de dados automaticamente fechadas, recursos liberados, arquivos abertos são fechados. Não precisa de pool de conexões com o banco por exemplo, etc. O fastCGI está cuidando disso tudo.
- `Auto-contido`: Cenários complexos (1 milhão de requisições por segundo, queries complicadas e demoradas, muitos arquivos, etc), use o servidor auto-contido, que é a implementação de cada linguagem para o processamento da requisição, o servidor escrito para aquela linguagem. Usa o proxy reverso e excelente. Toda linguagem interpretada possui algum servidor neste sentido. Qualquer linguagem que permita a abertura de um socket consegue, basta que alguém desenvolva o servidor.

### Configurando
- Criar um arquivo index.php num diretório.
- Rodar `docker run --rm -it -p 9000:9000 -v $(pwd):/caminho/projeto php:fpm` dentro deste diretório.
- Configurar o fastcgi_pass.
  ```nginx
  server {
    listen 8004;
    root /caminho/projeto;

      location / {
        include fastcgi.conf;
        fastcgi_pass localhost:9000;
    }
  }
  ```
- Recarrega o nginx e `http://localhost:8004/index.php` responde pelo arquivo index.php criado.
- Se criar outro arquivo `http://localhost:8004/teste.php` onde está sendo executado o docker run vai funcionar. porque foi mapeado na chamada do docker o `$(pwd)`. É possível ser outro diretório, é claro, basta ajustar a chamada do docker run. `docker run --rm -it -p 9000:9000 -v /root/arquivos-php:/caminho/projeto php:fpm`

## Performance
### Cache Navegador
Com visão voltada para performance, você indica ao cliente que pode armazenar o dado em cache.
Isso se dá através dos cabeçalhos http. Em especial o `Cache-Control` e `Expires`.
Configuramos com a opção expires.
```nginx
server {
  listen 8005;
  root /var/www/performance/;
  index index.html;

  location ~ \.jpg$ {
    # cache-control: max-age=2592000
    # max-age=2592000: Define que o recurso pode ser armazenado por 30 dias (em segundos: 30 * 24 * 60 * 60).
    # expires: Sun, 16 Mar 2025 19:41:04 GMT
    expires 30d;
  }
}
```

### Cache Public
O cabeçalho Cache-Control: public é uma diretiva do protocolo HTTP que informa que a resposta pode ser armazenada em cache por qualquer intermediário entre o servidor e o cliente final. Isso inclui:
- Navegadores
- Proxies reversos
- CDNs (Content Delivery Networks)
- Servidores de cache intermediários

Quando um servidor retorna uma resposta com Cache-Control: public, ele permite que qualquer cache intermediário (não apenas o navegador do usuário) armazene e reutilize essa resposta para futuras requisições. Obviamente isso é **desaconselhável para páginas dinamicas (.php, .aspx, .jsp) e para APIs que retornam dados sensíveis**. Deve ser usado então para conteúdos estáticos, como:
- Imagens (.jpg, .png, .svg)
- Arquivos CSS e JavaScript (.css, .js)
- Vídeos e áudios
- Arquivos de fontes (.woff2, .ttf)

Se um cliente (como um navegador) ou um servidor intermediário armazenar a resposta em cache, outras requisições que pedirem o mesmo recurso poderão ser atendidas sem que o servidor de origem seja acessado novamente, economizando banda, processamento e tempo de resposta.

```nginx
location ~* \.(jpg|png|gif|css|js|woff2|ttf)$ {
    expires 30d; # nginx transforma em max-age
    add_header Cache-Control public;
    # public: Permite que o recurso seja armazenado por qualquer cache intermediário.

    # Ou, somando os cabeçalhos em uma só informação
    # add_header Cache-Control "public, max-age=2592000";
}
```

### Cache Private
Para conteúdos personalizados para cada usuário (como dashboards e dados autenticados), use `Cache-Control: private`, que permite que somente o navegador do usuário armazene a resposta.

### Compressão
`gzip on` por padrão comprime apenas o html. Cabeçalho `Content-Encoding: gzip` é adicionado.

`gzip_types text/css` eu adiciono quais arquivos serão comprimidos. Ressalta-se que esse tipo de compressão funciona melhor com texto. Arquivos binários como imagens não são muito beneficiados. Ideal incluir então os .js, .svg, etc. Isso reduz bastante os dados trafegados na rede.

O arquivo CSS _bootstrap.min.css_ que possui 233kB chegou com 42kB!
```nginx
server {
  listen 8005;
  root /var/www/performance/;
  index index.html;
  gzip on;
  gzip_types text/css;

  location ~ \.jpg$ {
    expires 30d;
  }
}
```

### Conexões
`worker_processes` sugere-se um por núcleo (auto faz isso), mas pode tunar e avaliar com cada caso.
```nginx
worker_processes auto;
```

`Connection: keep-alive` cabeçalho http que seta um timeout antes de fechar a conexão. Senão faria uma conexão para cada elemento a ser baixado na aba network do devtools. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Keep-Alive).
```nginx
server {
  listen 8005;
  root /var/www/performance/;
  index index.html;
  gzip on;
  gzip_types image/jpeg text/css;
  
  # se dentro de 5s tentar fazer uma requisição essa conexão vai ser aproveitada
  add_header Keep-Alive "timeout=5, max=200";

  location ~ \.jpg$ {
    expires 30d;
  }
}
```
No devtools: `keep-alive: timeout=5, max=200`.
