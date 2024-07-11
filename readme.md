<a href="https://colab.research.google.com/github/joaomj/tipos_distancias/blob/master/tipos_distancias.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

# **Orientações gerais**
- Este código tem **fins educacionais apenas.**
- Você precisa criar um dataset de exemplo para testar esse código. Eu criei um bem simples aqui.
- Cuidado com o excesso de requisições às APIs. Eu testei com 200 ceps distintos e demorei quase 2h para obter as distâncias.

# **Imports**


```python
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


```python
# Configurações gerais

# determinando que ele não exiba os dados em formato notação científica
pd.set_option('display.float_format', lambda x: '%.5f' % x)

# Configuração para exibir apenas 2 dígitos após a vírgula
pd.options.display.float_format = '{:.2f}'.format

warnings.filterwarnings("ignore", category=DeprecationWarning)
warnings.filterwarnings("ignore", category=UserWarning)
```

# **Funções**


```python
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

# **Loading data**


```python
df = generate_dataset(2) #dataset com 2 linhas
df.head()
```

    Arquivo CSV gerado com sucesso!






  <div id="df-c8478dcd-2c1c-45a3-ad52-cbbbcb948a48" class="colab-df-container">
    <div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>sku</th>
      <th>origin</th>
      <th>destination</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>ZMF8MDD4</td>
      <td>83203-716</td>
      <td>14875-530</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0T9NT3W5</td>
      <td>69315-228</td>
      <td>14875-530</td>
    </tr>
  </tbody>
</table>
</div>
    <div class="colab-df-buttons">

  <div class="colab-df-container">
    <button class="colab-df-convert" onclick="convertToInteractive('df-c8478dcd-2c1c-45a3-ad52-cbbbcb948a48')"
            title="Convert this dataframe to an interactive table."
            style="display:none;">

  <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 -960 960 960">
    <path d="M120-120v-720h720v720H120Zm60-500h600v-160H180v160Zm220 220h160v-160H400v160Zm0 220h160v-160H400v160ZM180-400h160v-160H180v160Zm440 0h160v-160H620v160ZM180-180h160v-160H180v160Zm440 0h160v-160H620v160Z"/>
  </svg>
    </button>

  <style>
    .colab-df-container {
      display:flex;
      gap: 12px;
    }

    .colab-df-convert {
      background-color: #E8F0FE;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      display: none;
      fill: #1967D2;
      height: 32px;
      padding: 0 0 0 0;
      width: 32px;
    }

    .colab-df-convert:hover {
      background-color: #E2EBFA;
      box-shadow: 0px 1px 2px rgba(60, 64, 67, 0.3), 0px 1px 3px 1px rgba(60, 64, 67, 0.15);
      fill: #174EA6;
    }

    .colab-df-buttons div {
      margin-bottom: 4px;
    }

    [theme=dark] .colab-df-convert {
      background-color: #3B4455;
      fill: #D2E3FC;
    }

    [theme=dark] .colab-df-convert:hover {
      background-color: #434B5C;
      box-shadow: 0px 1px 3px 1px rgba(0, 0, 0, 0.15);
      filter: drop-shadow(0px 1px 2px rgba(0, 0, 0, 0.3));
      fill: #FFFFFF;
    }
  </style>

    <script>
      const buttonEl =
        document.querySelector('#df-c8478dcd-2c1c-45a3-ad52-cbbbcb948a48 button.colab-df-convert');
      buttonEl.style.display =
        google.colab.kernel.accessAllowed ? 'block' : 'none';

      async function convertToInteractive(key) {
        const element = document.querySelector('#df-c8478dcd-2c1c-45a3-ad52-cbbbcb948a48');
        const dataTable =
          await google.colab.kernel.invokeFunction('convertToInteractive',
                                                    [key], {});
        if (!dataTable) return;

        const docLinkHtml = 'Like what you see? Visit the ' +
          '<a target="_blank" href=https://colab.research.google.com/notebooks/data_table.ipynb>data table notebook</a>'
          + ' to learn more about interactive tables.';
        element.innerHTML = '';
        dataTable['output_type'] = 'display_data';
        await google.colab.output.renderOutput(dataTable, element);
        const docLink = document.createElement('div');
        docLink.innerHTML = docLinkHtml;
        element.appendChild(docLink);
      }
    </script>
  </div>


<div id="df-f5254a25-28f1-4837-8a2a-8f9e4c037325">
  <button class="colab-df-quickchart" onclick="quickchart('df-f5254a25-28f1-4837-8a2a-8f9e4c037325')"
            title="Suggest charts"
            style="display:none;">

<svg xmlns="http://www.w3.org/2000/svg" height="24px"viewBox="0 0 24 24"
     width="24px">
    <g>
        <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zM9 17H7v-7h2v7zm4 0h-2V7h2v10zm4 0h-2v-4h2v4z"/>
    </g>
</svg>
  </button>

<style>
  .colab-df-quickchart {
      --bg-color: #E8F0FE;
      --fill-color: #1967D2;
      --hover-bg-color: #E2EBFA;
      --hover-fill-color: #174EA6;
      --disabled-fill-color: #AAA;
      --disabled-bg-color: #DDD;
  }

  [theme=dark] .colab-df-quickchart {
      --bg-color: #3B4455;
      --fill-color: #D2E3FC;
      --hover-bg-color: #434B5C;
      --hover-fill-color: #FFFFFF;
      --disabled-bg-color: #3B4455;
      --disabled-fill-color: #666;
  }

  .colab-df-quickchart {
    background-color: var(--bg-color);
    border: none;
    border-radius: 50%;
    cursor: pointer;
    display: none;
    fill: var(--fill-color);
    height: 32px;
    padding: 0;
    width: 32px;
  }

  .colab-df-quickchart:hover {
    background-color: var(--hover-bg-color);
    box-shadow: 0 1px 2px rgba(60, 64, 67, 0.3), 0 1px 3px 1px rgba(60, 64, 67, 0.15);
    fill: var(--button-hover-fill-color);
  }

  .colab-df-quickchart-complete:disabled,
  .colab-df-quickchart-complete:disabled:hover {
    background-color: var(--disabled-bg-color);
    fill: var(--disabled-fill-color);
    box-shadow: none;
  }

  .colab-df-spinner {
    border: 2px solid var(--fill-color);
    border-color: transparent;
    border-bottom-color: var(--fill-color);
    animation:
      spin 1s steps(1) infinite;
  }

  @keyframes spin {
    0% {
      border-color: transparent;
      border-bottom-color: var(--fill-color);
      border-left-color: var(--fill-color);
    }
    20% {
      border-color: transparent;
      border-left-color: var(--fill-color);
      border-top-color: var(--fill-color);
    }
    30% {
      border-color: transparent;
      border-left-color: var(--fill-color);
      border-top-color: var(--fill-color);
      border-right-color: var(--fill-color);
    }
    40% {
      border-color: transparent;
      border-right-color: var(--fill-color);
      border-top-color: var(--fill-color);
    }
    60% {
      border-color: transparent;
      border-right-color: var(--fill-color);
    }
    80% {
      border-color: transparent;
      border-right-color: var(--fill-color);
      border-bottom-color: var(--fill-color);
    }
    90% {
      border-color: transparent;
      border-bottom-color: var(--fill-color);
    }
  }
</style>

  <script>
    async function quickchart(key) {
      const quickchartButtonEl =
        document.querySelector('#' + key + ' button');
      quickchartButtonEl.disabled = true;  // To prevent multiple clicks.
      quickchartButtonEl.classList.add('colab-df-spinner');
      try {
        const charts = await google.colab.kernel.invokeFunction(
            'suggestCharts', [key], {});
      } catch (error) {
        console.error('Error during call to suggestCharts:', error);
      }
      quickchartButtonEl.classList.remove('colab-df-spinner');
      quickchartButtonEl.classList.add('colab-df-quickchart-complete');
    }
    (() => {
      let quickchartButtonEl =
        document.querySelector('#df-f5254a25-28f1-4837-8a2a-8f9e4c037325 button');
      quickchartButtonEl.style.display =
        google.colab.kernel.accessAllowed ? 'block' : 'none';
    })();
  </script>
</div>

    </div>
  </div>




# **Obtendo as distâncias**
Campos 'origin' e 'destination' são CEPs. Alguns ceps que deveriam começar com zero estão sem esses números, sendo necessário formatá-los para que todos os ceps tenham 8 dígitos.


```python
df['origin'].unique()
```




    array(['83203-716', '69315-228'], dtype=object)




```python
df['destination'].unique()
```




    array(['14875-530'], dtype=object)



###Cálculo das distâncias de **deslocamento** entre os ceps, **em km:**


```python
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


```python
# grava distâncias no dataframe
df['distancia_km'] = df.apply(get_distance, axis=1)
df.head()
```





  <div id="df-0768cf51-36b0-43b0-a1c3-099bad19655b" class="colab-df-container">
    <div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>sku</th>
      <th>origin</th>
      <th>destination</th>
      <th>distancia_km</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>ZMF8MDD4</td>
      <td>83203-716</td>
      <td>14875-530</td>
      <td>716.05</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0T9NT3W5</td>
      <td>69315-228</td>
      <td>14875-530</td>
      <td>4329.49</td>
    </tr>
  </tbody>
</table>
</div>
    <div class="colab-df-buttons">

  <div class="colab-df-container">
    <button class="colab-df-convert" onclick="convertToInteractive('df-0768cf51-36b0-43b0-a1c3-099bad19655b')"
            title="Convert this dataframe to an interactive table."
            style="display:none;">

  <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 -960 960 960">
    <path d="M120-120v-720h720v720H120Zm60-500h600v-160H180v160Zm220 220h160v-160H400v160Zm0 220h160v-160H400v160ZM180-400h160v-160H180v160Zm440 0h160v-160H620v160ZM180-180h160v-160H180v160Zm440 0h160v-160H620v160Z"/>
  </svg>
    </button>

  <style>
    .colab-df-container {
      display:flex;
      gap: 12px;
    }

    .colab-df-convert {
      background-color: #E8F0FE;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      display: none;
      fill: #1967D2;
      height: 32px;
      padding: 0 0 0 0;
      width: 32px;
    }

    .colab-df-convert:hover {
      background-color: #E2EBFA;
      box-shadow: 0px 1px 2px rgba(60, 64, 67, 0.3), 0px 1px 3px 1px rgba(60, 64, 67, 0.15);
      fill: #174EA6;
    }

    .colab-df-buttons div {
      margin-bottom: 4px;
    }

    [theme=dark] .colab-df-convert {
      background-color: #3B4455;
      fill: #D2E3FC;
    }

    [theme=dark] .colab-df-convert:hover {
      background-color: #434B5C;
      box-shadow: 0px 1px 3px 1px rgba(0, 0, 0, 0.15);
      filter: drop-shadow(0px 1px 2px rgba(0, 0, 0, 0.3));
      fill: #FFFFFF;
    }
  </style>

    <script>
      const buttonEl =
        document.querySelector('#df-0768cf51-36b0-43b0-a1c3-099bad19655b button.colab-df-convert');
      buttonEl.style.display =
        google.colab.kernel.accessAllowed ? 'block' : 'none';

      async function convertToInteractive(key) {
        const element = document.querySelector('#df-0768cf51-36b0-43b0-a1c3-099bad19655b');
        const dataTable =
          await google.colab.kernel.invokeFunction('convertToInteractive',
                                                    [key], {});
        if (!dataTable) return;

        const docLinkHtml = 'Like what you see? Visit the ' +
          '<a target="_blank" href=https://colab.research.google.com/notebooks/data_table.ipynb>data table notebook</a>'
          + ' to learn more about interactive tables.';
        element.innerHTML = '';
        dataTable['output_type'] = 'display_data';
        await google.colab.output.renderOutput(dataTable, element);
        const docLink = document.createElement('div');
        docLink.innerHTML = docLinkHtml;
        element.appendChild(docLink);
      }
    </script>
  </div>


<div id="df-fe25c318-d1cb-457b-bcae-90c1a74021cd">
  <button class="colab-df-quickchart" onclick="quickchart('df-fe25c318-d1cb-457b-bcae-90c1a74021cd')"
            title="Suggest charts"
            style="display:none;">

<svg xmlns="http://www.w3.org/2000/svg" height="24px"viewBox="0 0 24 24"
     width="24px">
    <g>
        <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zM9 17H7v-7h2v7zm4 0h-2V7h2v10zm4 0h-2v-4h2v4z"/>
    </g>
</svg>
  </button>

<style>
  .colab-df-quickchart {
      --bg-color: #E8F0FE;
      --fill-color: #1967D2;
      --hover-bg-color: #E2EBFA;
      --hover-fill-color: #174EA6;
      --disabled-fill-color: #AAA;
      --disabled-bg-color: #DDD;
  }

  [theme=dark] .colab-df-quickchart {
      --bg-color: #3B4455;
      --fill-color: #D2E3FC;
      --hover-bg-color: #434B5C;
      --hover-fill-color: #FFFFFF;
      --disabled-bg-color: #3B4455;
      --disabled-fill-color: #666;
  }

  .colab-df-quickchart {
    background-color: var(--bg-color);
    border: none;
    border-radius: 50%;
    cursor: pointer;
    display: none;
    fill: var(--fill-color);
    height: 32px;
    padding: 0;
    width: 32px;
  }

  .colab-df-quickchart:hover {
    background-color: var(--hover-bg-color);
    box-shadow: 0 1px 2px rgba(60, 64, 67, 0.3), 0 1px 3px 1px rgba(60, 64, 67, 0.15);
    fill: var(--button-hover-fill-color);
  }

  .colab-df-quickchart-complete:disabled,
  .colab-df-quickchart-complete:disabled:hover {
    background-color: var(--disabled-bg-color);
    fill: var(--disabled-fill-color);
    box-shadow: none;
  }

  .colab-df-spinner {
    border: 2px solid var(--fill-color);
    border-color: transparent;
    border-bottom-color: var(--fill-color);
    animation:
      spin 1s steps(1) infinite;
  }

  @keyframes spin {
    0% {
      border-color: transparent;
      border-bottom-color: var(--fill-color);
      border-left-color: var(--fill-color);
    }
    20% {
      border-color: transparent;
      border-left-color: var(--fill-color);
      border-top-color: var(--fill-color);
    }
    30% {
      border-color: transparent;
      border-left-color: var(--fill-color);
      border-top-color: var(--fill-color);
      border-right-color: var(--fill-color);
    }
    40% {
      border-color: transparent;
      border-right-color: var(--fill-color);
      border-top-color: var(--fill-color);
    }
    60% {
      border-color: transparent;
      border-right-color: var(--fill-color);
    }
    80% {
      border-color: transparent;
      border-right-color: var(--fill-color);
      border-bottom-color: var(--fill-color);
    }
    90% {
      border-color: transparent;
      border-bottom-color: var(--fill-color);
    }
  }
</style>

  <script>
    async function quickchart(key) {
      const quickchartButtonEl =
        document.querySelector('#' + key + ' button');
      quickchartButtonEl.disabled = true;  // To prevent multiple clicks.
      quickchartButtonEl.classList.add('colab-df-spinner');
      try {
        const charts = await google.colab.kernel.invokeFunction(
            'suggestCharts', [key], {});
      } catch (error) {
        console.error('Error during call to suggestCharts:', error);
      }
      quickchartButtonEl.classList.remove('colab-df-spinner');
      quickchartButtonEl.classList.add('colab-df-quickchart-complete');
    }
    (() => {
      let quickchartButtonEl =
        document.querySelector('#df-fe25c318-d1cb-457b-bcae-90c1a74021cd button');
      quickchartButtonEl.style.display =
        google.colab.kernel.accessAllowed ? 'block' : 'none';
    })();
  </script>
</div>

    </div>
  </div>



