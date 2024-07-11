# Tipos de Distâncias

#### -- Status do Projeto: [Concluído]

## Objetivos
O objetivo deste projeto é mostrar um modo de calcular a distância relevante para empresas de logística e outros casos de uso. É a distância de deslocamento (*driving distance*).

Quando você precisa calcular a distância entre 2 coordenadas geográficas, é comum pensar em libs como a *geopy*. O problema é que ela retorna a distância **geodésica**, que é a distância "em linha reta" pelo globo terrestre. Muitas vezes a distância percorrida é diferente da distância geodésica, pois ela depende da existência de obstáculos naturais, escolha de rotas, etc.

### Métodos Utilizados
* Geração aleatória de um dataset pequeno
* API que retorna coordenadas geográficas para CEPs (https://cep.awesomeapi.com.br)
* API que retorna a distância de deslocamento entre 2 coordenadas geográficas (http://router.project-osrm.org)

### Tecnologias
* Python
* Pandas, Jupyter 
* json

## Descrição do Projeto

1. Gerar um dataset aleatório com ceps válidos.
2. Obter as coordenadas geográficas para ceps de origem e destino.
3. Calcular a distância de deslocamento entre as coordenadas.
4. Gravar os resultados em uma nova coluna no dataframe ('distancia_km').

## Faça você mesmo

1. Clone este repositório (leia o [tutorial](https://help.github.com/articles/cloning-a-repository/)).
2. Execute o notebook no [Google Colab](https://colab.google/). Assim você não precisa configurar um ambiente de desenvolvimento no seu computador.

## Contribuíram

* **João Marcos**: [Github](https://github.com/joaomj), [Linkedin](https://linkedin.com/in/joaomj).

## Contato
* Fique à vontade para entrar em contato comigo! Sugestões são bem vindas!
