# Lab: TimeSeries Dashboard, utilizando InfluxDB e Grafana

## Objetivo

Criar um laboratório simplificado, com produtos OpenSource e binários provisionados via "tarbal" (não implica em instalação de software no sistema operacional).

Neste laborátio, vamos utilizar o Linux SHELL para publicar dados de demonstraçao no InfluxDB utilizando o programa cUrl para postar dados via API HTTP, em seguida extrair seu conteúdo em um dashboard do próprio InfluxDB e também via Grafana.

## Versões de produtos

- InfluxDB 2.1.1
- Grafana 8.3

## Requisitos

- Linux Ubuntu x86_64 versão 20.04
- 1GB livre em /dev/shm
- Acesso a internet
- 30 minutos de tempo livre

## Preparacao

__Download e install dos produtos__

```bash
cd /dev/shm
mkdir lab
cd lab

wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.1.1-linux-amd64.tar.gz
wget https://dl.grafana.com/enterprise/release/grafana-enterprise-8.3.4.linux-amd64.tar.gz
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu2004-5.0.6.tgz

for f in *.tgz *.gz; do tar xvf $f; done
rm -f *.tgz *.tar.gz
mv grafana-8.3.4 grafana
mv influxdb2-2.1.1-linux-amd64 influxdb
mv mongodb-linux-x86_64-ubuntu2004-5.0.6 mongodb
```

__Setup InfluxDB__

```bash
cat <<'EOF' >influxdb/config.yaml 
bolt-path: "./data/influxd.bolt"
sqlite-path: "./data/influxd.sqlite"
engine-path: "./data/engine"
EOF
```

__Setup MongoDB__

```bash
cat <<'EOF'>mongodb/etc/mongod.conf 
security:
  authorization: disabled
processManagement:
  fork: true
  pidFilePath: /dev/shm/lab/mongodb/var/run/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo
net:
   ipv6: false
   bindIpAll: false
   bindIp: 0.0.0.0
   port: 27017
   maxIncomingConnections: 10
   wireObjectCheck: true
   compression:
      compressors: zstd
   unixDomainSocket:
      enabled: true
      pathPrefix: /dev/shm/lab/mongodb/var/run
      filePermissions: 700
storage:
   dbPath: /dev/shm/lab/mongodb/var/lib/mongod
   journal:
      enabled: true
      commitIntervalMs: 100
   directoryPerDB: true
   engine: wiredTiger
   wiredTiger:
      engineConfig:
         cacheSizeGB: 1
         journalCompressor: zstd
         directoryForIndexes: true
      collectionConfig:
         blockCompressor: zstd
      indexConfig:
         prefixCompression: true
systemLog:
  destination: file
  logAppend: true
  path: /dev/shm/lab/mongodb/var/log/mongod.log
EOF

mkdir -p mongodb/var/lib/mongod
mkdir -p mongodb/var/{log,run}
mkdir -p mongodb/etc
```

__Criar de inicializacao__

```bash
cat <<'EOF' >start.sh
#!/bin/bash
OPWD=$(pwd)
cd /dev/shm/lab/influxdb/
nohup ./influxd &
cd /dev/shm/lab/grafana/
nohup bin/grafana-server &
cd /dev/shm/lab/mongodb/
nohup bin/mongod -f etc/mongod.conf &
cd ${OPWD}
EOF
```

__Iniciar os daemons__

```bash
bash start.sh
```

# Executar LAB

__InfluxDB__

- Acessar https://localhost:8086

- Setup inicial do InfluxDB
	- Logar com usuário "admin" senha "admin"
	- Definir a nova senha como "admin123"
	- Definir o "org" como "mylab"

- Criar o bucket e token API
	-  Entrar no item "data" no menu lateral esquerdo
	- Selecionar a aba "API Tokens"
	- Clicar em "generate api token" e criar um novo Token
	- Selecionar a aba "Buckets"
	- Clicar em "create bucket"
	- Definir com nome "bucket1" e selecionar a opcao "never" em "Delete data" e clicar "Create"
- Popular dados de demonstracao
	- Executar o script 1 ao final deste documento, via shell em outra janela de terminal
- Consultar os dados
	- Acessar o painel do influxdb
	- Clicar no menu lateral "Explore"
	- Na caixa "FROM" selecionar "bucket1"
	- Na caixa "Filter" selecionar "randomData"
	- Na nova caixa "Filter" selecionar "rand1" e "rand2"
	- Na caixa "WINDOW PERIOD" clicar em "custom" e no campo abaixo selecionar "10s"
	- No item abaixo do seguimento "WINDOW PERIOD" nome "AGGREGATE FUNCTION" selecionar "custom" e marcar "mean"
	- Clicar no botao "Submit"

__Grafana__

- Acessar http://localhost:3000/
	- Login "admin" e senha "admin"
	- Redefinir a senha para "admin"
- Configurar datasource influxdb
	- No painel esquerdo, clicar na engrenagem e selecionar "datasources"
	- Na aba "Datasources" clicar em "add datasource"
	- Selecionar "influxDB"
	- Na tela de configuracao "Settings"
		- Query Language -> influx
		- HTTP
			- Url: http://localhost:8086
		- InfluxDB Details
			- Organization: mylab
			- Token:  <<<colarTokenApi>>>
			- Default bucket: bucket1
		- Clicar em "Save & test", deverá exibir uma mensagem "Datasource updated" com sinal verde.
- Criar dashboard teste
	- No menu esquerdo, clicar no icone com sinal positivo "+", e selecionar "Dashboard"
	- Clicar em "add panel"
	- Clicat em "sample query"
	- No texto da "query" substituir:
		- "db/rp" por "bucket1"
		- "example-measurement" por "randomData"
		- "example-field" por "rand1"
	- Será exibido o gráfico de pontos no painel acima

## Proposta de exercício aprofundado

Criar um script shell para ler a cada 1 segundo os dados de desempenho do mongodb via mongoshell, publicar no InfluxDB e criar um Dashboard com os dados de "operations" do MongoDB. Manda-Bala!!! =)

## Anexos

__Script 1__

```bash
#!/bin/bash
# ---------------------------------------------------------------
# Atencao, substituir o conteudo das variaveis "apiToken" e "orgName"
# pelos valores gerados na configuracao via painel do InfluxDB
#---------------------------------------------------------------
apiToken="38T4MHuJ9kxRaOX98dhQNYFVaqLItZffz3qOj6T4gNZMdmkmQxyKn0thz1KBzgZlz9_vMUpaFGAd_Juk5PErGg=="
orgName="mylab"
bucketName="bucket1"
# ---
for n in $(seq 1 1000); do
  timestamp="$(date +%s%N)"
  curl --request POST \
    "http://localhost:8086/api/v2/write?org=${mylab}&bucket=${bucketName}&precision=ns" \
    --header "Authorization: Token ${apiToken}" \
    --header "Content-Type: text/plain; charset=utf-8" \
    --header "Accept: application/json" \
    --data-binary "
      randomData,hostname='${HOSTNAME}' rand1=$(($RANDOM%120)).$(($RANDOM%99)),rand2=$(($RANDOM%1000)).$(($RANDOM%1000)) ${timestamp}
      randomData,hostname='${HOSTNAME}' rand1=$(($RANDOM%120)).$(($RANDOM%99)),rand2=$(($RANDOM%1000)).$(($RANDOM%1000)) ${timestamp}
      "
  sleep 0.25
  echo -en "."
done
echo done
```
