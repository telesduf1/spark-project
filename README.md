## Descrição
Projeto proposto pela Semantix ao final do treinamento de Engenharia de Dados para ingestão de Dados de Covid (https://mobileapps.saude.gov.br/esus-vepi/files/unAFkcaNDeXajurGB7LChj8SgQYS2ptm/04bd3419b22b9cc5c6efac2c6528100d_HIST_PAINEL_COVIDBR_06jul2021.rar)

O objetivo deste projeto é utilizar as ferramentas apresentadas no treinamento para consumir, armazenar e apresentar os dados de covid.

Segue uma breve descrição das ferramentas e plataformas utilizadas no projeto:

- wsl/wsl2: Subsistema do Windows para comandos linux com a finalidade de executar binários compilados para Linux;
- Docker: Plataforma open-source utilizada para gerenciamento e criação de ambientes de com aplicações isoladas em containers;
- Hadoop: Plataforma open-source para armazenamento e processamento de dados;
- Hive: Utilizado em conjunto com o Hadoop para definição, armazenamento e visualização de estrutura de dados;
- Kafka: Plataforma distribuída open-source de stream de dados;
- Elastic: Plataforma utilizada principalmente pelo motor de busca para processamento e armazenamento de dados dsitribuídos;
- Spark: Plataforma open-source para computação em cluster.

## Resolução
### 1. Enviar os dados para o hdfs
Dados foram baixados e descompactados em pasta local e copiados para uma nova pasta criada no HDFS via comando pel wsl:
```bash
# Criando diretório no HDFS
docker exec -it namenode hdfs dfs -mkdir -p /user/fabricio/projeto/dataset

# Execução de migrations para criação do banco
docker cp /mnt/d/treinamentos/spark/projeto/ namenode:/

# Entrando no namenode via bash para copiar os arquivos
sudo docker cp /mnt/d/treinamentos/spark/projeto/ namenode:/

# Executando dentro do namenode
root@namenode: hdfs dfs -put /projeto/*.csv /user/fabricio/projeto/dataset
```

### 2. Otimizar todos os dados do hdfs para uma tabela Hive particionada por município.
Para criação dos dados em uma tabela Hive, foi utilizado o jupyter notebook com PySpark com o objetivo de visualizar de forma mais rápida o schema do dataset e ingerir os dados a partir do DataFrame.

Porém, ao tentar inserir os dados com particionamento dinâmico utilizando o comando abaixo, o procesamento não chegou a finalizar:

```python
# Salva dados em tabela Hive
dados_covid_tratados.write.partitionBy('municipio').format("hive").mode("append").saveAsTable("painel_covid")
```

Dessa forma, foi mudado a estratégia e utilizado o beeline para criação de 2 (duas) tabelas: uma tabela raw, com dados brutos, e outra tabela formatada e particionada, com os dados nulos ignorados:

```bash
# Acessando o hive-server
docker exec -it hive-server bash

# Acessando o beeline
root@hive_server:/opt# beeline -u jdbc:hive2://localhost:10000

# Selecionando o banco
0: jdbc:hive2://localhost:10000> use dados_covid;

# Criando a tabela de dados brutos
0: jdbc:hive2://localhost:10000> CREATE TABLE painel_covid_raw(regiao string,
. . . . . . . . . . . . . . . .>              estado string,
. . . . . . . . . . . . . . . .>              municipio string,
. . . . . . . . . . . . . . . .>              coduf string,
. . . . . . . . . . . . . . . .>              codmun string,
. . . . . . . . . . . . . . . .>              codRegiaoSaude string,
. . . . . . . . . . . . . . . .>              nomeRegiaoSaude string,
. . . . . . . . . . . . . . . .>              data string,
. . . . . . . . . . . . . . . .>              semanaEpi string,
. . . . . . . . . . . . . . . .>              populacaoTCU2019 string,
. . . . . . . . . . . . . . . .>              casosAcumulado string,
. . . . . . . . . . . . . . . .>              casosNovos string,
. . . . . . . . . . . . . . . .>              obitosAcumulado string,
. . . . . . . . . . . . . . . .>              obitosNovos string,
. . . . . . . . . . . . . . . .>              Recuperadosnovos string,
. . . . . . . . . . . . . . . .>              emAcompanhamentoNovos string,
. . . . . . . . . . . . . . . .>              interior_metropolitana string)
. . . . . . . . . . . . . . . .> row format delimited
. . . . . . . . . . . . . . . .> fields terminated by ';'
. . . . . . . . . . . . . . . .> lines terminated by '\n'
. . . . . . . . . . . . . . . .> tblproperties('skip.header.line.count'='1');

# Criando a tabela otimizada
0: jdbc:hive2://localhost:10000> CREATE TABLE painel_covid(regiao string,
. . . . . . . . . . . . . . . .>              estado string,
. . . . . . . . . . . . . . . .>              cod_uf int,
. . . . . . . . . . . . . . . .>              cod_mun int,
. . . . . . . . . . . . . . . .>              cod_regiao_saude int,
. . . . . . . . . . . . . . . .>              nome_regiao_saude string,
. . . . . . . . . . . . . . . .>              data date,
. . . . . . . . . . . . . . . .>              semana_epi int,
. . . . . . . . . . . . . . . .>              populacao_tcu_2019 int,
. . . . . . . . . . . . . . . .>              casos_acumulados int,
. . . . . . . . . . . . . . . .>              casos_novos int,
. . . . . . . . . . . . . . . .>              obitos_acumulado int,
. . . . . . . . . . . . . . . .>              obitos_novos int,
. . . . . . . . . . . . . . . .>              recuperados_novos int,
. . . . . . . . . . . . . . . .>              em_acompanhamento_novos int,
. . . . . . . . . . . . . . . .>              interior_metropolitana string)
. . . . . . . . . . . . . . . .> partitioned by (municipio string)
. . . . . . . . . . . . . . . .> row format delimited
. . . . . . . . . . . . . . . .> fields terminated by ';'
. . . . . . . . . . . . . . . .> lines terminated by '\n'
. . . . . . . . . . . . . . . .> tblproperties('skip.header.line.count'='1');

# Inserido os dados dentro da tabela raw
0: jdbc:hive2://localhost:10000> load data inpath '/user/fabricio/projeto/dataset' overwrite into table painel_covid_raw;

# Setando partição de forma dinâmica
0: jdbc:hive2://localhost:10000> set hive.exec.dynamic.partition.mode=nonstrict;

# Inserido os dados dentro da tabela otimizada
INSERT OVERWRITE TABLE painel_covid 
PARTITION(municipio)
SELECT regiao, estado, coduf, codmun, codRegiaoSaude, nomeRegiaoSaude, data, semanaEpi, 
       populacaoTCU2019, casosAcumulado, casosNovos, obitosAcumulado, obitosNovos, Recuperadosnovos, 
       emAcompanhamentoNovos, interior_metropolitana, municipio
FROM painel_covid_raw
WHERE LENGTH(municipio) > 1 
  AND LENGTH(codmun) > 1
```

Ao inserir os dados dentro da tabela otimizada, o cluster apresentou um erro com o "reduce operator":

```bash
Query ID = root_20220809014851_53d6b9eb-0d42-4e7b-bf2b-f4640e1517f0
Total jobs = 3
Launching Job 1 out of 3
Number of reduce tasks is set to 0 since there's no reduce operator
Job running in-process (local Hadoop)
2022-08-09 01:48:57,248 Stage-1 map = 0%,  reduce = 0%
Ended Job = job_local494866582_0001 with errors
Error during job, obtaining debugging information...
FAILED: Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask
MapReduce Jobs Launched:
Stage-Stage-1:  HDFS Read: 0 HDFS Write: 0 FAIL
Total MapReduce CPU Time Spent: 0 msec
```
