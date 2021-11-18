## Service Discovery e Hashcorp Consul
Esse repositório é um compilado de informações referente ao [Consul](https://www.consul.io/docs/intro), nesse resumo estou focando no uso do Consul como service discovery, porém ele pode ter outras aplicações.

Para os exemplos e testes estou utilizando o Docker compose, onde tenho 3 containers que serão configurados como agent servers e 1 container como agent client.

### Subir os containers
`docker-compose up -d`

### Acessar o container
`docker exec -it consulserver01 sh`

### Set Server
Com os containers levantados precisamos startar o agent do Consul, no meu exemplo estou usando 3 Servers.
Um item muito importante é descobrir o IP de cada container, sendo assim, acessamos o container e usamos o comando abaixo:
`ifconfig`

Antes de iniciar o agente no modo server precisamos criar 2 diretórios:
`mkdir /etc/consul.d`
`mkdir /var/lib/consul`

Agora é só startar o agente usando seguinte comando com seus parametros:

`consul agent -server -bootstrap-expect=3 -node=consulserver01 -bind=172.24.0.3 -data-dir=/var/lib/consul -config-dir=/etc/consul.d`

`consul agent -server -bootstrap-expect=3 -node=consulserver02 -bind=172.24.0.2 -data-dir=/var/lib/consul -config-dir=/etc/consul.d`

`consul agent -server -bootstrap-expect=3 -node=consulserver03 -bind=172.24.0.4 -data-dir=/var/lib/consul -config-dir=/etc/consul.d`

### Criar o cluster de Servidores Consul
Acessamos o primeiro container: 
`docker exec -it consulserver01 sh`

Dentro do container consulserver01 usamos o comando de join:
`consul join IP_DO_SERVER_QUE_QUERO_JUNTAR`

`consul join 172.24.0.2`

`consul join 172.24.0.3`

### Set client

`consul agent -bind=172.24.0.5 -data-dir=/var/lib/consul -config-dir=/etc/consul.d`

### Adicionando Serviços
Na pasta clients/consulclient01 temos um arquivo services.json, nele informamos os dados do(s) serviço(s) rodando no client. Para recarregar as configurações, e o consul reconhecer esse serviço, basta rodar o comando `consul reload` dentro do container consulclient01. O consul adicionará o serviço no servidor de DNS dele.

### Consulta de DNS
Para consultar o DNS, primeiro precisamos instalr o app dig no container, para isso, use o comando:

`apk -U add bind-tools`

Uma vez instalador, basta rodar o seguinte comando para consultar o DNS:

`dig @localhost -p 8600 nginx.service.consul`

Se o serviço foi registrado corretamente, será exibido uma informação parecida com:

```
; <<>> DiG 9.16.20 <<>> @localhost -p 8600 nginx.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51244
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx.service.consul.          IN      A

;; ANSWER SECTION:
nginx.service.consul.   0       IN      A       172.24.0.5

;; Query time: 4 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Nov 18 21:29:40 UTC 2021
;; MSG SIZE  rcvd: 65

```
A partir do registro do serviço, todos os agentes conseguirão visualizar esse serviço.

Uma outra forma de consultar o serviço é usar a API, para isso use:

`curl localhost:8500/v1/catalog/services`

Também pode usar o comando abaixo para o consul retornar informações de onde esse serviço está sendo executado:

`consul catalog nodes -service nginx`

Para detalhes sobre os nós do cluster use:

`consul catalog nodes -detailed`

### Subir um novo client e adiciona-lo automaticamente no cluster
Para adicionar o novo agente client no cluster automaticamente, basta usar o parametro -retry-join passando o IP de um dos servidores.


`consul agent -bind=172.24.0.6 -data-dir=/var/lib/consul -config-dir=/etc/consul.d -retry-join=172.24.0.3
`
### Registrando 2 serviços iguais.
No segundo client que adicionamos também registramos o nginx como serviço, sendo assim ao consultar o DNS ele deve retornar os 2 apontamentos do Nginx.
