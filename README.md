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

## ⚙️ Detalhamento do ETL e Regras de Negócio

### 1. Extração (Extract)
* **Fonte:** API OpenWeatherMap (consultando `Sao Paulo,BR` com parâmetros de unidade no sistema métrico, garantindo que a temperatura já venha em graus Celsius).
* **Processo:** Requisição HTTP gerenciada pela task `extract`, salvando o dado bruto no formato `weather_data.json`.

### 2. Transformação (Transform)
Processamento feito na memória utilizando o `pandas` através da função `data_transformation()`. As regras aplicadas incluem:
* **Leitura e Normalização:** Criação do DataFrame a partir do JSON e normalização ('flattening') dos dados aninhados da coluna `weather` (extraindo `weather_id`, `weather_main`, e `weather_description`).
* **Limpeza:** Remoção de colunas desnecessárias para o negócio (como ícones de clima e tipos de sistema interno).
* **Padronização:** Renomeação completa de chaves do JSON para nomes de colunas claros no banco de dados (ex: `main.temp` virou `temperature`, `coord.lon` virou `longitude`).
* **Conversão de Tempo:** Conversão dos timestamps Unix (`dt`, `sunrise`, `sunset`) para o fuso horário local (`America/Sao_Paulo`).
* **Armazenamento Intermediário:** Os dados tratados são salvos no formato colunar `.parquet` para ganho de performance na leitura da próxima etapa.

### 3. Carga (Load)
A task `load` lê o arquivo temporário `.parquet` e insere os dados de forma estruturada na tabela `sp_weather` do banco de dados (PostgreSQL), garantindo que os dados estejam prontos para consumo por ferramentas de BI ou Cientistas de Dados.

## 🕸️ Fluxo da DAG no Apache Airflow

A DAG `weather_pipeline` foi configurada para rodar **a cada hora** (`0 */1 * * *`) e conta com as seguintes configurações principais:
* `depends_on_past`: False
* `retries`: 2 tentativas (com 5 minutos de intervalo em caso de falha da API)
* `catchup`: False

**Ordem de Execução das Tasks:**
`extract()` ➔ `transform()` ➔ `load()`

## 📊 Modelagem de Dados

Abaixo estão os principais dados extraídos e processados que compõem a tabela final `sp_weather`:

| Categoria | Colunas |
| :--- | :--- |
| **Identificação** | `city_id`, `city_name`, `country` |
| **Tempo** | `datetime`, `timezone`, `sunrise`, `sunset` |
| **Geolocalização** | `longitude`, `latitude` |
| **Temperatura (°C)** | `temperature`, `feels_like`, `temp_min`, `temp_max` |
| **Condições Atmosféricas** | `pressure`, `humidity`, `sea_level`, `grnd_level` |
| **Vento e Nuvens** | `wind_speed`, `wind_deg`, `wind_gust`, `clouds`, `visibility` |
| **Clima (Descritivo)** | `weather_id`, `weather_main`, `weather_description` |

## 🛠️ Tecnologias e Stack Detalhada

* **Core & Orquestração:** Python 3.14+, Apache Airflow
* **Processamento de Dados:** `pandas`
* **Armazenamento e Transporte:** Arquivos `.json` (Raw) e `.parquet` (Staging)
* **Banco de Dados:** PostgreSQL (`SQLAlchemy`, `psycopg2`)
* **Gerenciamento de Ambiente:** `python-dotenv` (para ocultar a API_KEY de forma segura)

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
git clone https://github.com/matheusaraujodata98/pipelines_etl_eng_dados_weather.git
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
3. Localize a DAG `weather_pipeline`, ative o botão (unpause) e acompanhe a execução automática!

---

*Projeto desenvolvido para fim de estudos voltado para a Engenharia de Dados*
