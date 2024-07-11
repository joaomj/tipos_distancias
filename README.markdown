---
jupyter:
  colab:
    authorship_tag: ABX9TyMAiW+Z/Vpm+BzjtIeO31T1
    collapsed_sections:
    - \_kjFRT19XTtF
    - "-S98j7IDh2Q4"
    - 4ER4e-D7hrcf
    - nthS_2fUhkia
    - di3ajdKih9BX
    - xYVt6dUViFTX
    - 6CtLRocipPLL
    - I9sCBWO7yv4V
    include_colab_link: true
  kernelspec:
    display_name: Python 3
    name: python3
  language_info:
    name: python
  nbformat: 4
  nbformat_minor: 0
---

::: {.cell .markdown colab_type="text" id="view-in-github"}
`<a href="https://colab.research.google.com/github/joaomj/tipos_distancias/blob/master/tipos_distancias.ipynb" target="_parent">`{=html}`<img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/>`{=html}`</a>`{=html}
:::

::: {.cell .markdown id="IbqM5VmborLQ"}
# **Orientações gerais**

-   Este código tem **fins educacionais apenas.**
-   Você precisa criar um dataset de exemplo para testar esse código. Eu
    criei um bem simples aqui.
-   Cuidado com o excesso de requisições às APIs. Eu testei com 200 ceps
    distintos e demorei quase 2h para obter as distâncias.
:::

::: {.cell .markdown id="lYdeCGrVXLL1"}
# **Imports**
:::

::: {.cell .code id="F7Hg06bmV3ln"}
``` python
import pandas as pd
import os
import numpy as np
import warnings
import time
import requests
import csv
import random
import string
```
:::

::: {.cell .code id="UDm6jVb_9Gq8"}
``` python
# Configurações gerais

# determinando que ele não exiba os dados em formato notação científica
pd.set_option('display.float_format', lambda x: '%.5f' % x)

# Configuração para exibir apenas 2 dígitos após a vírgula
pd.options.display.float_format = '{:.2f}'.format

warnings.filterwarnings("ignore", category=DeprecationWarning)
warnings.filterwarnings("ignore", category=UserWarning)
```
:::

::: {.cell .markdown id="pP4xipIV9bhB"}
# **Funções**
:::

::: {.cell .code id="zN8z3alK9CaV"}
``` python
# -----------------------------------------
# Função para escolher aleatoriamente ceps de uma lista de ceps válidos
def generate_random_cep():
    ceps = ['14875-530', '69315-228', '83203-716', '09432-115'] # exemplos de ceps válidos

    # Seleciona aleatoriamente da lista de CEPs
    selected_cep = random.choice(ceps)

    return selected_cep

# -----------------------------------------
# Função para gerar um dataset de exemplo
def generate_dataset(n):

    # Gera dados de exemplo
    data = []

    for _ in range(n):
        sku = ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))
        origin = generate_random_cep()
        destination = generate_random_cep()
        data.append([sku, origin, destination])

    # Escreve os dados no arquivo CSV
    with open('dados.csv', mode='w', newline='') as file:
        writer = csv.writer(file)
        writer.writerow(['sku', 'origin', 'destination'])  # Escrever o cabeçalho
        writer.writerows(data)  # Escrever as linhas

    print("Arquivo CSV gerado com sucesso!")

    # Obtendo o path deste notebook
    current_dir = os.getcwd()

    # Importando arquivos
    df = pd.read_csv('dados.csv')
    return df

# -------------------------------------------
# Função para obter coordenadas de um CEP usando a API, com rate limiting
def get_coordinates(cep):
    url = f'https://cep.awesomeapi.com.br/json/{cep}'
    try:
        response = requests.get(url)
        if response.status_code == 200:
            data = response.json()
            return float(data['lat']), float(data['lng'])
        else:
            return np.nan, np.nan
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data for {cep}: {e}")
        return np.nan, np.nan
    finally:
        # Esperar 1 segundo entre as requisições para não sobrecarregar a API
        time.sleep(1)

# -------------------------------------------
# Função para obter a distância de condução entre duas coordenadas com rate limit e cache
def get_driving_distance(origin_coords, dest_coords):

    # Declarar o cache de distâncias
    distance_cache = {}

    # Verificar se a distância já foi calculada e está no cache
    if (origin_coords, dest_coords) in distance_cache:
        return distance_cache[(origin_coords, dest_coords)]

    profile = "driving"  # Tipo de perfil, pode ser driving, walking, cycling etc.
    coordinates = f"{origin_coords[1]},{origin_coords[0]};{dest_coords[1]},{dest_coords[0]}"
    url = f"http://router.project-osrm.org/route/v1/{profile}/{coordinates}?overview=false&alternatives=false&steps=false&annotations=false"
    response = requests.get(url)
    if response.status_code == 200:
        data = response.json()
        if 'routes' in data and len(data['routes']) > 0:
            distance_km = data['routes'][0]['distance'] / 1000  # Convertendo de metros para quilômetros
            distance_cache[(origin_coords, dest_coords)] = distance_km
            time.sleep(1)  # Intervalo de 1 segundo entre as requisições
            return distance_km

    return np.nan

# -------------------------------------------
# Função para obter a distância do dicionário
def get_distance(row):
    return cep_distances.get((row['origin'], row['destination']), 'Distância não encontrada')
```
:::

::: {.cell .markdown id="_kjFRT19XTtF"}
# **Loading data**
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\",\"height\":129}" id="jeX8CWW-vVjr" outputId="65a8be4b-6da2-43b7-8cd1-e483a180c539"}
``` python
df = generate_dataset(2) #dataset com 2 linhas
df.head()
```

::: {.output .stream .stdout}
    Arquivo CSV gerado com sucesso!
:::

::: {.output .execute_result execution_count="62"}
``` json
{"summary":"{\n  \"name\": \"df\",\n  \"rows\": 2,\n  \"fields\": [\n    {\n      \"column\": \"sku\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 2,\n        \"samples\": [\n          \"0T9NT3W5\",\n          \"ZMF8MDD4\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"origin\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 2,\n        \"samples\": [\n          \"69315-228\",\n          \"83203-716\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"destination\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 1,\n        \"samples\": [\n          \"14875-530\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    }\n  ]\n}","type":"dataframe","variable_name":"df"}
```
:::
:::

::: {.cell .markdown id="xCKRpkyt90Ww"}
# **Obtendo as distâncias**

Campos \'origin\' e \'destination\' são CEPs. Alguns ceps que deveriam
começar com zero estão sem esses números, sendo necessário formatá-los
para que todos os ceps tenham 8 dígitos.
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\"}" id="LS7fzFRO-PBO" outputId="c7911bad-3647-40e4-af4b-11d94b5deac5"}
``` python
df['origin'].unique()
```

::: {.output .execute_result execution_count="63"}
    array(['83203-716', '69315-228'], dtype=object)
:::
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\"}" id="wAH0MdUE-0RW" outputId="dc1a7a13-fa2d-4d5b-895d-d3f05b452f13"}
``` python
df['destination'].unique()
```

::: {.output .execute_result execution_count="64"}
    array(['14875-530'], dtype=object)
:::
:::

::: {.cell .markdown id="DijwBahr12G2"}
###Cálculo das distâncias de **deslocamento** entre os ceps, **em km:**
:::

::: {.cell .code id="js0OOfO-AsP2"}
``` python
# colunas 'origin' e 'destination' têm os CEPs
unique_origins = df['origin'].unique()
unique_destinations = df['destination'].unique()

# Dicionário para armazenar as distâncias entre os CEPs
cep_distances = {}

# Iterar sobre os pares únicos de origem e destino
for origin_cep in unique_origins:
    # Obter coordenadas do CEP de origem
    origin_coords = get_coordinates(origin_cep)
    if np.isnan(origin_coords).any():
        print('CEP de origem inválido')
        break

    for dest_cep in unique_destinations:
        # Obter coordenadas do CEP de destino
        dest_coords = get_coordinates(dest_cep)
        if np.isnan(dest_coords).any():
            print('CEP de destino inválido')
            break

        # Calcular a distância de condução entre as coordenadas
        distance = get_driving_distance(origin_coords, dest_coords)

        # Armazenar a distância no dicionário
        cep_distances[(origin_cep, dest_cep)] = distance
```
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\",\"height\":112}" id="Kx7aFx2p2QDk" outputId="b17ff5c4-7404-4133-967c-97479e8159d8"}
``` python
# grava distâncias no dataframe
df['distancia_km'] = df.apply(get_distance, axis=1)
df.head()
```

::: {.output .execute_result execution_count="66"}
``` json
{"summary":"{\n  \"name\": \"df\",\n  \"rows\": 2,\n  \"fields\": [\n    {\n      \"column\": \"sku\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 2,\n        \"samples\": [\n          \"0T9NT3W5\",\n          \"ZMF8MDD4\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"origin\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 2,\n        \"samples\": [\n          \"69315-228\",\n          \"83203-716\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"destination\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 1,\n        \"samples\": [\n          \"14875-530\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"distancia_km\",\n      \"properties\": {\n        \"dtype\": \"number\",\n        \"std\": 2555.0886345174995,\n        \"min\": 716.046,\n        \"max\": 4329.487,\n        \"num_unique_values\": 2,\n        \"samples\": [\n          4329.487\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    }\n  ]\n}","type":"dataframe","variable_name":"df"}
```
:::
:::
