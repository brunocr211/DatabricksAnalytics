# Processamento de fluxo com o Azure Databricks

Esta arquitetura de referência mostra um pipeline completo de [processamento de fluxo](/azure/architecture/data-guide/big-data/real-time-processing). Esse tipo de pipeline tem quatro etapas: ingestão, processamento, armazenamento e análise e relatórios. Para esta arquitetura de referência, o pipeline ingere dados de duas fontes, realiza uma junção de registos relacionados de cada fluxo, enriquece o resultado e calcula uma média em tempo real. Os resultados são armazenados para análise posterior. 

![](https://github.com/mspnp/architecture-center/blob/master/docs/reference-architectures/data/images/stream-processing-databricks.png)

**Cenário**: Uma empresa de táxis recolhe dados sobre cada viagem de táxi. Para este cenário, assumimos que existem dois dispositivos separados a enviar dados. O táxi tem um taxímetro que envia informações sobre cada viagem — a duração, a distância e os locais de embarque e desembarque. Um dispositivo separado aceita pagamentos dos clientes e envia dados sobre as tarifas. Para identificar tendências de passageiros, a empresa de táxis deseja calcular a gorjeta média por quilómetro percorrido, em tempo real, para cada bairro.

## Implante a solução

Uma implantação para esta arquitetura de referência está disponível no [GitHub](https://github.com/mspnp/azure-databricks-streaming-analytics). 

### Pré-requisitos

1. Clone, faça um fork ou transfira este repositório GitHub.

2. Instale o [Docker](https://www.docker.com/) para executar o gerador de dados.

3. Instale o [Azure CLI 2.0](/cli/azure/install-azure-cli?view=azure-cli

4. Instale o [Databricks CLI](https://docs.databricks.com/user-guide/dev-tools/databricks-cli.html).

5. A partir de um prompt de comando, prompt bash ou prompt PowerShell, inicie sessão na sua conta Azure da seguinte forma:
```shell
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
