# System-monitor 
Este repositório implementa um pipeline completo para a análise de dados do monitoramento do OpenShift por meio de modelos de Machine Learning.

## Visão Geral
O projeto está organizado em módulos bem definidos, que se comunicam entre si para formar o fluxo de dados:
* `models/` → contém a lógica dos modelos de IA (pré-processamento, treinamento, predição).
* `controllers/` → realizam as ações diversas de análise, tal como o treinamento, teste e visualização dos modelos.
* `views/` → responsáveis pela visualização e geração dos gráficos.
* `outputs/` → repositório dos resultados estáticos (gráficos `.png`, modelos salvos).
* `params/` → repositório dos parâmetros, weights e biases dos modelos.
* Scripts principais (`app.py`, `importCSV.py`) → entrada do sistema e utilidades.

``` bash 
├── importCSV.py           # Script auxiliar para ingestão de dados
│
├── controllers/           
│   ├── multi_nn_controller.py
│   ├── no_training_controller.py
│   ├── results_controller.py
│   ├── simple_nn_controller.py
│   ├── super_nn_controller.py
│   ├── overview_controller.py
│   └── xgboost_controller.py
│
├── models/                # Modelos de IA e DataLoader
│   ├── base_model.py
│   ├── dataloader.py
│   ├── multi_nn_model.py
│   ├── simple_nn_model.py
│   ├── super_nn_model.py
│   ├── nn_model.py
│   └── xgboost_model.py
│
├── views/                 # Visualizações e geração de gráficos
│   ├── base_view.py
│   ├── overview.py
│   └── time_series_view.py
│   └── hyperparameters_view.py
│   └── results_view.py
│
├── params/                # Os weights, biases e outros parametros dos modelos 
│   └── super_nn/
│
├── outputs/               # Resultados do projeto
│   ├── neural_network/    # Gráficos das redes neurais
│   ├── xgboost/           # Gráficos do modelo XGBoost
│   ├── decomposition_*    # Decomposição de séries temporais
│   ├── samples_daily_*    # Amostras diárias
│   ├── Top50Namespaces*   # Estatísticas agregadas
│   └── *.keras            # Modelos treinados salvos
│
├── requirements.txt       # Dependências do projeto
└── README.md              # Documentação principal
```

## Configuração do ambiente
1. Clonar repositório:  
``` bash
git clone https://github.com/LuizF14/system-monitor.git
cd system-monitor
``` 
2. Criar ambiente virtual (recomendado):
``` bash
python -m venv venv
source .venv/bin/activate
```
3. Instalar dependências:
``` bash
pip install --upgrade pip
pip install -r requirements.txt
``` 
## Iniciando o banco de dados
Esta aplicação utiliza um banco de dados PostgreSql para interagir com os dados. Para inicializá-lo é necessário rodar um container Docker. Preencha o template abaixo de acordo suas especificações, execute-o e configure apropriadamente o `.env`: 
``` docker
services:
  db:
    image: postgres:15
    container_name: postgres-db
    restart: always
    environment:
      POSTGRES_DB: nome_da_db
      POSTGRES_USER: seu_usuario
      POSTGRES_PASSWORD: sua_senha
    ports:
      - "5432:5432"
    volumes:
``` 

## Importando CSV para o banco de dados

## Dataloader
O módulo `models/dataloader.py` fornece a classe `DataLoader`, responsável por gerenciar conexões com um banco de dados PostgreSQL, incluindo abertura de túnel SSH, execução de queries e criação de materialized views para otimizar consultas. Todas as configurações de conexão devem ser definidas no arquivo `.env`:
``` env
# Banco de dados
DB_HOST=localhost
DB_NAME=meu_banco
DB_USER=usuario
DB_PASS=senha
DB_TABLE=ocnr_dados

# Conexão SSH (opcional)
SSH_HOST=servidor.remoto.com
SSH_PORT=22
SSH_USER=ssh_user
SSH_PASS=ssh_senha
LOCAL_PORT=5433
```  
Se a conexão for local, basta configurar apenas as variáveis do banco. Se a conexão for via túnel SSH, é necessário preencher também os campos `SSH_*`. Os principais métodos dessa classe são: 
* `start_ssh_tunnel`: Abre um túnel SSH local para o servidor remoto, redirecionando a porta do PostgreSQL.
* `connect_to_db`: Estabelece a conexão com o banco de dados PostgreSQL.
* `close`: Encerra tanto a conexão com o banco de dados quanto o túnel SSH (se aberto).
* `query_to_db`: Executa uma query SQL e retorna o resultado como um `DataFrame` pandas.
* `createOverviewView`: Cria a materialized view `ocnr_overview` com estatísticas resumidas de cada namespace e query: máximo, mínimo, média, desvio padrão e número de amostras.
* `fullDataOverview`: retorna os dados da materialized view `ocnr_overview`.
* `createNamespaceView`: Cria uma materialized view específica para um namespace, facilitando consultas filtradas. Se a view já existir, nenhuma ação é tomada.

## Models
O diretório `models/` contém as classes responsáveis pela lógica intrínseca dos modelos de IA: preparação dos dados, definição de hiperparâmetros, treinamento e validação, predição com novos dados, etc. Todos os modelos devem herdar da classe abstrata `BaseModel`, que define uma interface comum para padronizar o uso dentro do projeto. Ela define os seguintes métodos: 
* `preprocess`: Realiza o pré-processamento dos dados de entrada, adaptando-os à lógica específica do modelo.
* `train`: Treina o modelo com base nos dados.
* `predict`: Realiza previsões a partir dos dados informados.
* `load`: Carrega um modelo salvo em disco.

## Controllers
O diretório `controllers/` contém as classes responsáveis por orquestrar todo pipeline de treinamento, teste e visualização dos modelos. Os controllers atuam como camada de integração, coordenando chamadas para o `Dataloader`, os `models` e as `views`. Exigi-se que toda classe `controller` implemente no mínimo o método `run`.
O diretório `controllers/` contém scripts que realizam tarefas diversas, tais como treinamento, teste e visualização dos modelos e análise de resultados. Os controllers atuam como camada de integração, coordenando chamadas para o `Dataloader`, os `models` e as `views`. É possível rodar qualquer `controller` por meio do comando:
```
python -m controllers.<nome_do_controller>
```

## Views
O diretório `views/` contém as funções responsáveis pela plotagem e visualização dos dados e resultados dos modelos. A plotagem é feita por meio das bibliotecas `matplotlib`, `seaborn` e `pandas`. Todos os gráficos são salvos em arquivos .png e as tabelas em arquivos .csv dentro do diretório `outputs/`. É possível definir subpastas na chamada destas funções para organizar melhor os arquivos.

## Outputs
O diretório `outputs/` contém todos os arquivos estáticos gerados pelo projeto, como imagens (`.png`) e tabelas (`.csv`). A organização segue a seguinte hierarquia:
* `neural_network/`: resultados relacionados a redes neurais.
* `xgboost/`: resultados relacionados ao modelo XGBoost.
* arquivos na raiz (`outputs/`): análises gerais, decomposições e estatísticas.
Os gráficos seguem o padrão: 
```
<prefixo>_<período>_<modelo>_<tarefa>_<namespace>.png
``` 
1. Prefixo  
* Indica o tipo de gráfico: 
* `tm` → Timeseries: plota somente a série temporal ao longo do tempo.
* `p` → Performance: plota valores reais vs previstos.
* `e` → Evolution: plota reais vs previstos e adiciona a linha de separação entre treino e teste.
* `l` → Lag search: mostra a performance do modelo em função da variação dos lags.

2. Período 
* Indica o intervalo de tempo da análise. Exemplo: `day15`, `nov`.

3. Modelo
* Nome do modelo utilizado, como `xgboost`.

4. Tarefa
* `forecast`: previsão de séries temporais.
* `classification`: classifacação de dados.

5. Namespace
* Namespace utilizado: `panda-druid`, `panda-nifi`.

## Params
O diretório `params` contém os parâmetros, weights e biases dos modelos. Por meio dele, é possível reconstruir um modelo com o método `.load()`. O método `.save()` salva os parâmetros dos modelos em arquivos nesse diretório. 
