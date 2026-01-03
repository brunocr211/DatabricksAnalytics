# Processamento de fluxo com o Azure Databricks

Esta arquitetura de referência mostra um pipeline completo de [processamento de fluxo](/azure/architecture/data-guide/big-data/real-time-processing). Esse tipo de pipeline tem quatro etapas: ingestão, processamento, armazenamento e análise e relatórios. Para esta arquitetura de referência, o pipeline ingere dados de duas fontes, realiza uma junção de registos relacionados de cada fluxo, enriquece o resultado e calcula uma média em tempo real. Os resultados são armazenados para análise posterior. 

![](https://github.com/mspnp/architecture-center/blob/master/docs/reference-architectures/data/images/stream-processing-databricks.png)

**Cenário**: Uma empresa de táxis recolhe dados sobre cada viagem de táxi. Para este cenário, assumimos que existem dois dispositivos separados a enviar dados. O táxi tem um taxímetro que envia informações sobre cada viagem — a duração, a distância e os locais de embarque e desembarque. Um dispositivo separado aceita pagamentos dos clientes e envia dados sobre as tarifas. Para identificar tendências de passageiros, a empresa de táxis deseja calcular a gorjeta média por quilómetro percorrido, em tempo real, para cada bairro.

## Implante a solução

Uma implantação para esta arquitetura de referência está disponível no [GitHub](https://github.com/mspnp/azure-databricks-streaming-analytics). 

### Pré-requisitos

1. Clone, faça um fork ou transfira este repositório GitHub.

2. Instale o [Docker](https://www.docker.com/) para executar o gerador de dados.

3. Instale o [Azure CLI 2.0](/cli/azure/install-azure-cli?view=azure-cli.

4. Instale o [Databricks CLI](https://docs.databricks.com/user-guide/dev-tools/databricks-cli.html).

5. A partir de um prompt de comando, prompt bash ou prompt PowerShell, inicie sessão na sua conta Azure da seguinte forma:
shell
    az login
    ```
6. Instale um IDE Java com os seguintes recursos:
    - JDK 1.8
    - Scala SDK 2.11
    - Maven 3.5.4

### Baixe os ficheiros de dados sobre táxis e bairros da cidade de Nova Iorque

1. Crie um diretório chamado `DataFile` na raiz do repositório Github clonado no seu sistema de ficheiros local.

2. Abra um navegador da Web e aceda a https://uofi.app.box.com/v/NYCtaxidata/folder/2332219935.

3. Clique no botão **Download** nesta página para descarregar um ficheiro zip com todos os dados de táxis desse ano.

4. Extraia o ficheiro zip para o diretório `DataFile`.

    > [!NOTA]
    > Este ficheiro zip contém outros ficheiros zip. Não extraia os ficheiros zip filhos.

    A estrutura do diretório deve ficar assim:

```shell
    /DataFile
        /FOIL2013
            trip_data_1.zip
            trip_data_2.zip
            trip_data_3.zip
            ...
```
5. Abra um navegador da Web e aceda a https://www.zillow.com/howto/api/neighborhood-boundaries.htm.

6. Clique em **New York Neighborhood Boundaries** para descarregar o ficheiro.

7. Copie o ficheiro **ZillowNeighborhoods-NY.zip** da pasta **downloads** do seu navegador para a pasta `DataFile`.

### Implemente os recursos do Azure

1. A partir de um shell ou do Prompt de Comando do Windows, execute o seguinte comando e siga as instruções de início de sessão:

```bash
    az login
```

2. Navegue até a pasta chamada `azure` no repositório GitHub:

```bash
    cd azure
```

3. Execute os seguintes comandos para implementar os recursos do Azure:

```bash
    export resourceGroup=‘[Nome do grupo de recursos]’
    export resourceLocation='[Região]'
    export eventHubNamespace=‘[Nome do namespace do Event Hubs]’
    export databricksWorkspaceName=‘[Nome do espaço de trabalho do Azure Databricks]’
    export cosmosDatabaseAccount=‘[Nome do banco de dados do Cosmos DB]’
    export logAnalyticsWorkspaceName=‘[Nome do espaço de trabalho do Log Analytics]’
    export logAnalyticsWorkspaceRegion='[Região do Log Analytics]'

# Criar um grupo de recursos
    az group create --name $resourceGroup --location $resourceLocation

# Implantar recursos
    az group deployment create --resource-group $resourceGroup \
        --template-file deployresources.json --parameters \
	    eventHubNamespace=$eventHubNamespace \
        databricksWorkspaceName=$databricksWorkspaceName \
        cosmosDatabaseAccount=$cosmosDatabaseAccount \
        logAnalyticsWorkspaceName=$logAnalyticsWorkspaceName \
        logAnalyticsWorkspaceRegion=$logAnalyticsWorkspaceRegion
```
4. A saída da implementação é gravada na consola assim que concluída. Procure na saída o seguinte JSON:

```JSON
“outputs”: {
        “cosmosDb”: {
          “type”: “Object”,
          “value”: {
            “hostName”: <valor>,
            “secret”: <valor>,
            “username”: <valor>
          }
        },
        “eventHubs”: {
          “type”: “Object”,
          “value”: {
            “taxi-fare-eh”: <valor>,
            “taxi-ride-eh”: <valor>
          }
        },
        “logAnalytics”: {
          “type”: “Object”,
          “value”: {
            “secret”: <valor>,
            “workspaceId”: <valor>
          }
        }
},
```
Esses valores são os segredos que serão adicionados aos segredos do Databricks nas próximas secções. Mantenha-os em segurança até adicioná-los nessas secções.

### Adicionar uma tabela Cassandra à conta Cosmos DB

1. No portal do Azure, navegue até ao grupo de recursos criado na secção **implantar os recursos do Azure** acima. Clique em **Conta Cosmos DB do Azure**. Crie uma tabela com a API Cassandra.

2. Na guia **Visão geral**, clique em **Adicionar tabela**.

3. Quando a guia **Adicionar tabela** for aberta, insira `newyorktaxi` na caixa de texto **Nome do espaço de chaves**. 


4. Na secção **introduzir comando CQL para criar a tabela**, introduza `neighborhoodstats` na caixa de texto ao lado de `newyorktaxi`.

5. Na caixa de texto abaixo, introduza o seguinte:
```shell
(neighborhood text, window_end timestamp, number_of_rides bigint,total_fare_amount double, primary key(neighborhood, window_end))
```
6. Na caixa de texto **Throughput (1.000 - 1.000.000 RU/s)**, insira o valor `4000`.

7. Clique em **OK**.

### Adicione os segredos do Databricks usando a CLI do Databricks

Primeiro, insira os segredos para o EventHub:

1. Usando a **CLI do Azure Databricks** instalada na etapa 2 dos pré-requisitos, crie o escopo secreto do Azure Databricks:
```shell
    databricks secrets create-scope --scope “azure-databricks-job”
 ```
2. Adicione o segredo para o EventHub de viagem de táxi:
```shell
    databricks secrets put --scope “azure-databricks-job” --key “taxi-ride”
 ```
Uma vez executado, este comando abre o editor vi. Introduza o valor **taxi-ride-eh** da secção de saída **eventHubs** na etapa 4 da secção *implantar os recursos do Azure*. Guarde e saia do vi.

3. Adicione o segredo para o EventHub da tarifa de táxi:
```shell
    databricks secrets put --scope “azure-databricks-job” --key “taxi-fare”
```
    Depois de executado, este comando abre o editor vi. Introduza o valor **taxi-fare-eh** da secção de saída **eventHubs** no passo 4 da secção *implantar os recursos do Azure*. Guarde e saia do vi.

Em seguida, introduza os segredos para o Cosmos DB:

1. Abra o portal do Azure e navegue até ao grupo de recursos especificado no passo 3 da secção **implantar os recursos do Azure**. Clique na conta do Azure Cosmos DB.

2. Usando a **CLI do Azure Databricks**, adicione o segredo para o nome de utilizador do Cosmos DB:
```shell
    databricks secrets put --scope azure-databricks-job --key “cassandra-username”
```
Depois de executado, este comando abre o editor vi. Insira o valor do **nome de utilizador** da secção de saída **CosmosDb** na etapa 4 da secção *implantar os recursos do Azure*. Guarde e saia do vi.

3. Em seguida, adicione o segredo para a palavra-passe do Cosmos DB:
```shell
    databricks secrets put --scope azure-databricks-job --key “cassandra-password”
```

Depois de executado, este comando abre o editor vi. Introduza o valor **secret** da secção de saída **CosmosDb** no passo 4 da secção *implantar os recursos do Azure*. Guarde e saia do vi.

> [!NOTA]

> Se estiver a utilizar um [escopo secreto suportado pelo Azure Key Vault](https://docs.azuredatabricks.net/user-guide/secrets/secret-scopes.html#azure-key-vault-backed-scopes), o escopo deve ser nomeado **azure-databricks-job** e os segredos devem ter exatamente os mesmos nomes que os acima.

### Adicione o ficheiro de dados Zillow Neighborhoods ao sistema de ficheiros Databricks

1. Crie um diretório no sistema de ficheiros Databricks:
```bash
    dbfs mkdirs dbfs:/azure-databricks-jobs
    ```

2. Navegue até o diretório `DataFile` e insira o seguinte:
```bash
    dbfs cp ZillowNeighborhoods-NY.zip dbfs:/azure-databricks-jobs
 ```

### Adicione o ID da área de trabalho do Azure Log Analytics e a chave primária aos ficheiros de configuração

Para esta secção, necessita do ID da área de trabalho do Log Analytics e da chave primária. O ID da área de trabalho é o valor **workspaceId** da secção de saída **logAnalytics** na etapa 4 da secção *implantar os recursos do Azure*. A chave primária é o **secret** da secção de saída. 

1. Para configurar o registo log4j, abra `\azure\AzureDataBricksJob\src\main\resources\com\microsoft\pnp\azuredatabricksjob\log4j.properties`. Edite os dois valores seguintes:
```shell
    log4j.appender.A1.workspaceId=<ID do espaço de trabalho do Log Analytics>
    log4j.appender.A1.secret=<Chave primária do Log Analytics>
 ```

2. Para configurar o registo personalizado, abra `\azure\azure-databricks-monitoring\scripts\metrics.properties`. Edite os dois valores seguintes:
```shell
    *.sink.loganalytics.workspaceId=<ID do espaço de trabalho do Log Analytics>
    *.sink.loganalytics.secret=<Chave primária do Log Analytics>
 ```

### Crie os ficheiros .jar para a tarefa Databricks e o monitoramento Databricks

1. Use o seu IDE Java para importar o ficheiro de projeto Maven chamado **pom.xml** localizado no diretório raiz. 

2. Realize uma compilação limpa. O resultado dessa compilação são os ficheiros denominados **azure-databricks-job-1.0-SNAPSHOT.jar** e **azure-databricks-monitoring-0.9.jar**. 

### Configure o registo personalizado para a tarefa Databricks

1. Copie o ficheiro **azure-databricks-monitoring-0.9.jar** para o sistema de ficheiros Databricks, introduzindo o seguinte comando na **CLI Databricks**:
```shell
    databricks fs cp --overwrite azure-databricks-monitoring-0.9.jar dbfs:/azure-databricks-job/azure-databricks-monitoring-0.9.jar
 ```

2. Copie as propriedades de registo personalizadas de `\azure\azure-databricks-monitoring\scripts\metrics.properties` para o sistema de ficheiros Databricks, introduzindo o seguinte comando:
```shell
    databricks fs cp --overwrite metrics.properties dbfs:/azure-databricks-job/metrics.properties
```

