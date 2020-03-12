# Análise - Engenharia de Dados

Este repositório possui um projeto pessoal para análise de uma base de dados das corridas de taxi da cidade de NY entre os anos de 2009 e 2012. Todo o código fonte (códigos e queries) pode ser encontrado dentro do arquivo [Analise.html](https://github.com/fabdutra/ds-dataengineer-test/blob/master/Analise.html) bem como a explicação sobre cada questão. Para a parte de streaming, foi gerado o arquivo [Analise-streaming.html](https://github.com/fabdutra/ds-dataengineer-test/blob/master/Analise-streaming.html). Foram disponibilidados também os notebooks utilizados na realização da análise. São eles [Analise.ipynb](https://github.com/fabdutra/ds-dataengineer-test/blob/master/notebooks/Analise.ipynb) e [Analise-streaming.ipynb](https://github.com/fabdutra/ds-dataengineer-test/blob/master/notebooks/Analise-streaming.ipynb).

## Conteúdo

- [Questões Respondidas](#Questões Respondidas)
- [Arquitetura GCP](#Arquitetura)
    - Cloud Storage
    - Dataproc Cluster
    - VM Instance
- [Databricks](#Databricks)
- [Arquivos](#Arquivos)

## Questões Respondidas
Foram respondidas as seguintes questões no notebook [Analise.ipynb](https://github.com/fabdutra/ds-dataengineer-test/blob/master/notebooks/Analise.ipynb):
1. Qual a distância média percorrida por viagens com no máximo 2 passageiros?
2. Quais os 3 maiores vendors em quantidade total de dinheiro arrecadado?
3. Criação de um histograma da distribuição mensal, nos 4 anos, de corridas pagas em dinheiro.
4. Qual o tempo médio das corridas nos dias de sábado e domingo?

E no notebook [Analise-streaming.ipynb](https://github.com/fabdutra/ds-dataengineer-test/blob/master/notebooks/Analise-streaming.ipynb) foi simulado um streaming de dados e feita uma visualização acompanhando uma métrica em tempo-real.

## Arquitetura GCP

Em toda a solução foram provisionadas instâncias na GCP (Google Cloud Platform). O motivo de ter escolhido a GCP e não outras, como por exemplo AWS ou Azure, se deve ao fato do custo. A opção free tier da AWS não me possibilitaria criar as instâncias conforme foram provisionadas na minha conta trial da GCP. Foi então criado o projeto data-sprints contendo:

### Cloud Storage
No bucket `ds-nyc-taxi-trips` foram armazenados todos os datasets utilizados no teste, os gráficos gerados em cada questão e os notebooks criados durante a solução.


[![ds-nyc-taxi-trips bucket](https://storage.googleapis.com/public-data-engineering/bucket-details.png)]()

### Dataproc Cluster
O cluster `nyc-taxi-trips` possui um nó master e dois workers. Foi criado com a opção do componente Jupyter notebook. Nele foi executado o notebook Analise.ipynb

[![vm instances detail](https://storage.googleapis.com/public-data-engineering/VM-instances-details.png)]()

### Ubuntu VM Instance
A VM confluent foi criada para utilização do kafka durante a execução do teste bônus de streaming.

```shell
#setting the instance environment
$ sudo apt update
$ sudo apt install default-jre -y
$ sudo apt install default-jdk -y
$ sudo apt-get install jq -y

# downloading and installing confluent
$ curl -O http://packages.confluent.io/archive/5.4/confluent-5.4.0-2.12.tar.gz
$ tar xzf confluent-5.4.0-2.12.tar.gz

#configuring confluent according to its documentation
#https://docs.confluent.io/current/quickstart/ce-quickstart.html#ce-quickstart
$ curl -L https://cnfl.io/cli | sh -s -- -b /home/fabricio_dutra87/confluent-5.4.0/bin

#installing spool dir connector to handle json files
$ confluent-hub install jcustenborder/kafka-connect-spooldir:latest

#creating the input, error and finished path for the json files
$ cd /home/fabricio_dutra87/
$ mkdir nyc-taxi-trips
$ mkdir nyc-taxi-trips/data
$ mkdir nyc-taxi-trips/error
$ mkdir nyc-taxi-trips/finished

#configuring the connectors (do the same for all connectors)
$ nano spooldir2009.properties
#insert the following configuration into the file
#
#name=JsonSpoolDir
#tasks.max=1
#connector.class=com.github.jcustenborder.kafka.connect.spooldir.SpoolDirSchemaLessJsonSourceConnector
#input.path=/home/fabricio_dutra87/nyc-taxi-trips/data
#input.file.pattern=data-sample_data-nyctaxi-trips-2009-json_corrigido.json
#error.path=/home/fabricio_dutra87/nyc-taxi-trips/error
#finished.path=/home/fabricio_dutra87/nyc-taxi-trips/finished
#halt.on.error=false
#topic=spooldir2009-json-topic
#value.converter=org.apache.kafka.connect.storage.StringConverter

#Loading the SpoolDir JSON Source Connectors using the confluent local load command
$ confluent local load spool2009dir -- -d spooldir2009.properties
$ confluent local load spool2010dir -- -d spooldir2010.properties
$ confluent local load spool2011dir -- -d spooldir2011.properties
$ confluent local load spool2012dir -- -d spooldir2012.properties

#downloading the data
$ curl https://s3.amazonaws.com/data-sprints-eng-test/data-sample_data-nyctaxi-trips-2009-json_corrigido.json > /home/fabricio_dutra87/nyc-taxi-trips/data/data-sample_data-nyctaxi-trips-2009-json_corrigido.json
$ curl https://s3.amazonaws.com/data-sprints-eng-test/data-sample_data-nyctaxi-trips-2010-json_corrigido.json > /home/fabricio_dutra87/nyc-taxi-trips/data/data-sample_data-nyctaxi-trips-2010-json_corrigido.json
$ curl https://s3.amazonaws.com/data-sprints-eng-test/data-sample_data-nyctaxi-trips-2011-json_corrigido.json > /home/fabricio_dutra87/nyc-taxi-trips/data/data-sample_data-nyctaxi-trips-2011-json_corrigido.json
$ curl https://s3.amazonaws.com/data-sprints-eng-test/data-sample_data-nyctaxi-trips-2012-json_corrigido.json > /home/fabricio_dutra87/nyc-taxi-trips/data/data-sample_data-nyctaxi-trips-2012-json_corrigido.json

#tracking the processed messages
$ kafka-console-consumer --bootstrap-server localhost:9092 --property schema.registry.url=http://localhost:8081 --property print.key=true --from-beginning --topic spooldir2009-json-topic
```


## Databricks
Para execução do notebook [Analise-streaming.ipynb](https://github.com/fabdutra/ds-dataengineer-test/blob/master/notebooks/Analise-streaming.ipynb) foi utilizado o [databricks](https://community.cloud.databricks.com/ "Databricks Community's Homepage") em sua versão gratuita community. A escolha se deu pela possibilidade de implementação de uma arquitetura Kappa, onde os dados são ingeridos no Kafka, processados no Spark e posteriormente armazenados diretamente no Delta Lake.

[![kappa](https://storage.googleapis.com/public-data-engineering/kappa.png)]()

## Arquivos
Foram utilizados os seguintes arquivos na Análise.
1. Arquivos de extensão `json` contendo os dados sobre corridas de taxi em NY entre 2009 e 2012
   - data-sample_data-nyctaxi-trips-2009-json_corrigido.json
   - data-sample_data-nyctaxi-trips-2010-json_corrigido.json
   - data-sample_data-nyctaxi-trips-2011-json_corrigido.json
   - data-sample_data-nyctaxi-trips-2012-json_corrigido.json
2. Arquivo `csv` com os dados sobre empresas de serviço de taxi
   - data-vendor_lookup-csv.csv
3. Arquivo `csv` com o mapa entre prefixos e tipos reais de pagamento
   - data-payment_lookup-csv.csv
