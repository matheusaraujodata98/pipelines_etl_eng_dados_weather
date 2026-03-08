# ⛅ Pipeline de ETL: Clima de São Paulo (OpenWeatherMap)

[![Python 3.14+](https://img.shields.io/badge/Python_3.14+-3776AB?style=flat&logo=python&logoColor=white)](https://www.python.org/)
[![Status](https://img.shields.io/badge/Status-Concluído-success.svg)]()
[![Airflow](https://img.shields.io/badge/Airflow-017CEE?style=flat&logo=Apache%20Airflow&logoColor=white)]()
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)]()
[![Docker](https://img.shields.io/badge/Docker-2CA5E0?style=flat&logo=docker&logoColor=white)]()
[![Pandas](https://img.shields.io/badge/Pandas-150458?style=flat&logo=pandas&logoColor=white)]()
[![Parquet](https://img.shields.io/badge/Parquet-ffffff?style=flat&logo=apacheparquet&logoColor=black)]()

## 📖 Visão Geral do Projeto

Este projeto é um pipeline de **ETL (Extract, Transform, Load)** desenvolvido para coletar e processar dados climáticos da cidade de São Paulo de forma 100% automatizada. 

O sistema consome a API do **OpenWeatherMap**, realiza a limpeza e desestruturação de dados complexos em JSON usando **Pandas**, armazena temporariamente em formato otimizado **Parquet** e carrega os resultados finais em um banco de dados relacional. Todo o fluxo é orquestrado pelo **Apache Airflow**, com execuções agendadas de hora em hora.

---

## ⚙️ Detalhamento das Etapas do ETL

### 📥 ETAPA 1: Extração (Extract)
**Script:** `src/extract_data.py`

A primeira fase do pipeline é responsável por buscar os dados em tempo real. O script realiza uma requisição HTTP (GET) direcionada à API do OpenWeatherMap. Para garantir a confiabilidade da ingestão, o código valida o *status code* da resposta (esperando um `200 OK`). Em caso de sucesso, o payload (dado bruto) é salvo localmente no diretório `data/weather_data.json`.

**Métricas e dados coletados:**
* Temperaturas (Atual, Mínima, Máxima) e Sensação Térmica.
* Indicadores atmosféricos: Umidade e Pressão.
* Dinâmica do ar: Velocidade e direção do vento, além do nível de nebulosidade.
* Astronomia local: Horários exatos do nascer e pôr do sol.
* Coordenadas geográficas da medição.

### 🔄 ETAPA 2: Transformação (Transform)
**Script:** `src/transform_data.py`

Nesta etapa, o dado bruto em JSON é processado e modelado para uso analítico através de 5 sub-etapas utilizando o `pandas`:

1. **Ingestão e Achatamento:** O arquivo JSON é lido e convertido em um DataFrame. Utiliza-se a função `pd.json_normalize()` para "achatar" (flatten) as estruturas de dados iniciais.
2. **Descompactação da coluna 'Weather':** Como a coluna climática vem estruturada como uma lista de dicionários, o script a desmembra para extrair atributos específicos (`weather_id`, `weather_main`, `weather_description` e `weather_icon`), concatenando-os ao DataFrame principal.
3. **Limpeza de Atributos:** Colunas que não agregam valor analítico ao negócio são descartadas para poupar armazenamento.
4. **Padronização de Nomenclatura:** Chaves complexas ou aninhadas são renomeadas para o padrão inglês claro (ex: `main.temp` torna-se `temperature`, `sys.sunrise` torna-se `sunrise`).
5. **Normalização Temporal:** Tratamento de fuso horário, convertendo timestamps Unix para o formato datetime na *timezone* correta de São Paulo:
   ```python
   # Converte para datetime do fuso horário de São Paulo
   df[col] = pd.to_datetime(df[col], unit='s', utc=True).dt.tz_convert('America/Sao_Paulo')
   ```
*Resultado:* Um DataFrame íntegro, limpo e preparado para o banco de dados.

### 💾 ETAPA 3: Carga (Load)
**Script:** `src/load_data.py`

A etapa final persiste os dados transformados em um Data Warehouse/Data Mart local.

1. **Conexão:** Estabelece comunicação segura com o PostgreSQL utilizando `SQLAlchemy`.
2. **Inserção Histórica:** Os dados são gravados na tabela `sp_weather`. A estratégia de inserção utiliza o parâmetro `if_exists='append'`, garantindo que o pipeline construa um histórico contínuo (série temporal) a cada nova execução, sem sobrescrever o passado.
3. **Auditoria e Validação:** Após o *commit*, o script realiza um `SELECT COUNT(*)` diretamente no banco. Esse total de registros é logado no console, permitindo auditar o crescimento da base e garantir o sucesso da transação.

---

## 📊 Orquestração e Fluxo da DAG (Apache Airflow)

**Arquivo:** `dags/weather_pipeline.py`

O fluxo de execução e as dependências das tarefas são gerenciados pelo **Apache Airflow**. A DAG foi configurada para ser executada de hora em hora, de forma autônoma. 

### Definição das Tasks e Ordem de Execução

O pipeline é dividido em tarefas atômicas (usando a Taskflow API do Airflow):

```python
@task
def extract():
    extract_weather_data(url)

@task
def transform():
    df = data_transformations()
    # Salvando em Parquet para trânsito entre as tasks
    df.to_parquet('/opt/airflow/data/temp_data.parquet', index=False)

@task
def load():
    import pandas as pd
    df = pd.read_parquet('/opt/airflow/data/temp_data.parquet')
    load_weather_data('sp_weather', df)

# Definição do fluxo e dependências
extract() >> transform() >> load()
```

### 💡 Decisão Arquitetural: Por que utilizar o formato Parquet?
Em vez de trafegar o DataFrame diretamente pelo banco de metadados do Airflow (via XCom), optou-se por salvar um arquivo temporário na etapa de transformação e lê-lo na etapa de carga. Essa prática traz vantagens cruciais:
* **Eficiência de Armazenamento:** Sendo um formato binário colunar, o Parquet é extremamente leve e rápido de ler/escrever.
* **Integridade de Dados:** Preserva fielmente os tipos de dados nativos do Pandas (como *datetime* com fuso horário e *floats*), o que se perde em formatos como CSV ou JSON.
* **Prevenção de Gargalos (XCom Limits):** Evita problemas de serialização e sobrecarga de memória no banco de dados interno do Airflow, que não foi desenhado para transitar grandes volumes de dados.

### 🏗️ Arquitetura do Pipeline

```mermaid
graph LR
    %% Definindo os nós e ícones
    API((☁️ API OpenWeatherMap))
    JSON_RAW[📄 weather_data.json]
    PARQUET[🧊 temp_data.parquet]
    DB[(🛢️ PostgreSQL: sp_weather)]

    %% Definindo a caixa do Airflow
    subgraph "Orquestração: Apache Airflow (Docker)"
        T1(🐍 Task: extract)
        T2(🐼 Task: transform)
        T3(⚙️ Task: load)
    end

    %% Conectando o fluxo
    API -->|HTTP GET| T1
    T1 -->|Salva Raw| JSON_RAW
    JSON_RAW -->|Lê| T2
    T2 -->|Limpa e Modela| PARQUET
    PARQUET -->|Lê via Pandas| T3
    T3 -->|SQLAlchemy| DB

    %% Estilização (Cores)
    style API fill:#f9f9f9,stroke:#333,stroke-width:2px
    style DB fill:#316192,color:#fff,stroke:#333
    style PARQUET fill:#fff,stroke:#150458,stroke-width:2px
```

---

## 🚀 Como Configurar e Executar Localmente

### Pré-requisitos

Antes de começar, você precisará ter as seguintes ferramentas instaladas na sua máquina:

* [Docker e Docker Compose](https://docs.docker.com/get-docker/) para rodar os containers do Airflow e PostgreSQL.
* [Git](https://git-scm.com/downloads) para clonar o repositório.
* Uma conta gratuita no [OpenWeatherMap](https://openweathermap.org/) para obter a chave de acesso (API Key).

---

### 1️⃣ Clone o Repositório

Abra o seu terminal e execute:

```bash
git clone [https://github.com/matheusaraujodata98/pipelines_etl_eng_dados_weather.git](https://github.com/matheusaraujodata98/pipelines_etl_eng_dados_weather.git)
cd pipelines_etl_eng_dados_weather
```

### 2️⃣ Obtenha sua API Key do OpenWeatherMap

1. Acesse o site do [OpenWeatherMap](https://openweathermap.org/).
2. Crie uma conta gratuita.
3. Vá até o seu *Dashboard* e gere uma nova **API Key**.
4. Guarde essa chave, pois você precisará dela no próximo passo.

### 3️⃣ Configure as Variáveis de Ambiente

Crie as pastas necessárias e o arquivo `.env` dentro da pasta `config/`:

```bash
mkdir -p config data
touch config/.env
```

Abra o arquivo `.env` que você acabou de criar e adicione suas credenciais (substitua pelo seu valor real):

```env
# config/.env

# OpenWeatherMap API
API_KEY=sua_chave_api_aqui

# PostgreSQL (para testes locais)
POSTGRES_USER=airflow
POSTGRES_PASSWORD=airflow
POSTGRES_DB=weather_db
```

### 4️⃣ Inicie a Infraestrutura

Com tudo configurado, suba os containers do Airflow e do banco de dados utilizando o Docker Compose:

```bash
docker-compose up -d --build
```

### 5️⃣ Acesse o Airflow

1. Abra o seu navegador e acesse a URL: `http://localhost:8080`
2. Faça o login utilizando as credenciais padrão (geralmente `airflow` para usuário e senha, dependendo do seu docker-compose).
3. Localize a DAG, ative o botão (unpause) e acompanhe a execução automática!

---

*Projeto desenvolvido para fins de estudos voltado para a Engenharia de Dados.*