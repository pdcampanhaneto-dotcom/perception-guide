# Pipeline de Percepção LiDAR

Este documento descreve o funcionamento do pipeline responsável por transformar a nuvem de pontos do LiDAR em detecções de cones utilizadas pelo sistema Driverless.

O objetivo não é apenas explicar os algoritmos utilizados, mas também registrar como as diferentes partes do código se conectam para que futuros membros da microdivisão consigam compreender, executar e modificar o sistema.

---

## 1. Visão geral

O sistema recebe uma nuvem de pontos tridimensional produzida pelo LiDAR LeiShen CH128X1.

A partir dessa nuvem, o processamento busca identificar grupos de pontos que correspondam aos cones da pista.

De forma simplificada:

```text
                 LiDAR
                   │
                   ▼
              PointCloud2
                   │
                   ▼
            Região de interesse
                   │
                   ▼
           Remoção do chão
              (MLESAC)
                   │
                   ▼
             Clusterização
                   │
                   ▼
              Restauração
                   │
                   ▼
         Filtro geométrico
                   │
                   ▼
            Cones detectados
                   │
                   ├───────────────┐
                   │               │
                   ▼               ▼
              LiDAR only       LiDAR + ZED
                   │               │
                   │               ▼
                   │        classificação de cor
                   │               │
                   └───────┬───────┘
                           ▼
                         saída
```

O pipeline é implementado principalmente pelo pacote `detector_pkg`.

---

## 2. Arquitetura do código

Os principais arquivos relacionados ao processamento são:

| Arquivo | Responsabilidade |
|---|---|
| `main.py` | Nó ROS 2 principal e execução do pipeline |
| `constants.py` | Parâmetros utilizados pelo processamento |
| `ROI.py` | Filtragem da região de interesse |
| `MLESAC.py` | Estimativa e remoção do plano do chão |
| `clustering.py` | Agrupamento dos pontos e cálculo dos centroides |
| `geometric_filters.py` | Extração de características e classificação dos clusters |
| `fusio_engine.py` | Processamento das detecções da ZED e fusão com LiDAR |
| `transform.py` | Transformação das coordenadas da ZED para o referencial utilizado pelo LiDAR |
| `artificial_lidar.py` | Publicação de nuvem de pontos artificial |
| `artificial_framing_lidar.py` | Reprodução de nuvens de pontos armazenadas |
| `artificial_zed.py` | Publicação de detecções artificiais da ZED |

O arquivo `main.py` coordena essas etapas e conecta o processamento aos tópicos ROS 2.

---

## 3. Entradas e saídas

### Entrada do LiDAR

O processamento recebe uma mensagem:

```text
sensor_msgs/msg/PointCloud2
```

O tópico consumido pelo nó é definido em:

```python
LIDAR_TOPIC_TO_BE_SUBSCRIBED
```

no arquivo `constants.py`.

### Entrada da câmera

Quando utilizada, a informação da ZED é recebida através do tópico definido por:

```python
CAMERA_TOPIC_TO_BE_SUBSCRIBED
```

A informação contém as coordenadas dos cones detectados pela câmera e sua classificação de cor.

### Saída

As detecções finais são organizadas em grupos de três valores:

```text
[x, y, color]
```

Assim, uma mensagem com vários cones possui a estrutura:

```text
[x1, y1, color1, x2, y2, color2, ...]
```

As coordenadas utilizadas na saída correspondem à posição dos cones detectados pelo processamento LiDAR.

---

## 4. Fluxo do processamento

O processamento principal pode ser entendido como seis etapas:

1. **Filtragem da região de interesse (ROI)**
2. **Remoção do chão utilizando MLESAC**
3. **Clusterização dos pontos**
4. **Restauração de pontos**
5. **Classificação geométrica dos clusters**
6. **Coloração e publicação**

Cada etapa recebe o resultado da anterior e reduz progressivamente a nuvem até chegar aos objetos considerados cones.

---

## 5. Gerenciamento dos sensores

O nó de processamento acompanha a disponibilidade das mensagens do LiDAR e da câmera.

Para isso, as mensagens recebidas são armazenadas e um temporizador verifica se os dados são recentes o suficiente para serem utilizados.

O sistema considera quatro situações:

```text
                 LiDAR válido?
                 /           \
              sim             não
              /                 \
       câmera válida?       câmera válida?
        /       \             /       \
      sim       não         sim       não
       │         │           │         │
     BOTH    LIDAR ONLY  CAMERA ONLY  NO SENSOR
```

A intenção desse mecanismo é permitir que o sistema continue utilizando um sensor quando o outro estiver indisponível ou atrasado.

---

## 6. Região de Interesse (ROI)

A primeira filtragem reduz a quantidade de pontos que será processada.

O filtro considera limites mínimos e máximos para as três coordenadas:

```text
xmin ≤ x ≤ xmax
ymin ≤ y ≤ ymax
zmin ≤ z ≤ zmax
```

Os limites são definidos em `constants.py`.

Quando existem detecções provenientes da ZED, o processamento pode utilizar essas posições para restringir ainda mais a região considerada.

Para cada cone detectado pela câmera, são considerados pontos do LiDAR próximos à posição estimada do cone.

O parâmetro:

```python
POINT_CONE_DISTANCE
```

define essa distância.

---

## 7. Remoção do chão — MLESAC

Depois da filtragem inicial, a nuvem ainda contém os pontos correspondentes ao chão.

O pipeline utiliza MLESAC para estimar o plano que representa o solo.

De forma conceitual:

```text
Nuvem filtrada
      │
      ▼
Downsampling
      │
      ▼
Geração de planos candidatos
      │
      ▼
Avaliação dos candidatos
      │
      ▼
Melhor plano
      │
      ▼
Remoção dos pontos próximos ao plano
      │
      ▼
Nuvem sem o chão
```

A implementação utiliza uma região específica da nuvem e um voxel grid para reduzir a quantidade de pontos utilizada na estimativa.

Os principais parâmetros relacionados ao MLESAC estão em `constants.py`, incluindo:

```text
XMIN
XMAX
YMIN
YMAX
VOXEL_LEAF_SIZE
NUM_PLANES
INLIER_PROB
P_OUTLIER
DIST2PLANE_THRESHOLD
INLIER_STD_DEV
```

---

## 8. Clusterização

Após a remoção do chão, os pontos restantes precisam ser separados em objetos.

O arquivo `clustering.py` implementa essa etapa utilizando o `DBSCAN` da biblioteca Scikit-learn.

A configuração utilizada é:

```text
eps = RADIUS_THRESHOLD
min_samples = 1
```

Com `min_samples = 1`, o algoritmo é utilizado pela equipe como uma forma de **Euclidean Clustering**, agrupando pontos de acordo com sua conectividade espacial.

O resultado é um rótulo para cada ponto:

```text
Point 1 → cluster 0
Point 2 → cluster 0
Point 3 → cluster 1
Point 4 → cluster 1
...
```

Esses rótulos são posteriormente convertidos em um dicionário no formato:

```text
cluster_id → pontos pertencentes ao cluster
```

Por fim, o centroide de cada cluster é calculado pela média das coordenadas dos seus pontos.

---

## 9. Restauração de pontos

A remoção do chão pode remover pontos que pertencem à parte inferior dos cones.

Para recuperar parte dessas informações, o pipeline possui uma etapa de restauração.

A partir do centroide de cada cluster, são procurados pontos na nuvem utilizada anteriormente que estejam próximos à sua posição.

A ideia é:

```text
Antes da restauração:

      • •
    • • •
      ↑
   cluster

──────────── chão removido ────────────


Depois da restauração:

      • •
    • • •
   • • • •
      ↑
   cluster restaurado
```

A distância utilizada nessa busca é definida pelos parâmetros de restauração em `constants.py`.

---

## 10. Classificação geométrica

Nem todo cluster corresponde a um cone.

Por isso, cada cluster é analisado utilizando características geométricas.

O código considera, entre outras:

- diâmetro;
- extensão vertical;
- razão altura/base;
- desvio padrão em Z;
- elongação da caixa delimitadora;
- diagonal da caixa delimitadora;
- altura do centroide;
- quantidade de pontos;
- omnivariance.

Essas características são comparadas com valores esperados para clusters correspondentes a cones.

Existe também uma quantidade mínima de pontos:

```python
MIN_LEN
```

Clusters que não possuem pontos suficientes são descartados antes da avaliação.

O resultado da classificação é uma pontuação utilizada para decidir se o cluster deve ser considerado um cone.

---

## 11. Fusão com a ZED

Quando informações da câmera estão disponíveis, o processamento pode associar os cones encontrados pelo LiDAR às detecções da ZED.

O processo utiliza a proximidade entre as posições dos cones.

A informação da câmera também fornece a classificação de cor.

De forma simplificada:

```text
              LiDAR
                │
                ▼
        clusters detectados
                │
                │ posição
                ▼
          associação espacial
                ▲
                │ posição + cor
                │
                ZED
```

A associação utiliza `POINT_CONE_DISTANCE` como limite de proximidade.

Quando a câmera não está disponível, o sistema também possui uma forma de produzir uma classificação de cor baseada apenas no LiDAR.

---

## 12. Dados artificiais

O pacote possui nós auxiliares que permitem testar o pipeline sem utilizar diretamente os sensores físicos.

### `artificial_lidar`

Carrega uma nuvem de pontos armazenada em arquivo e a publica como `PointCloud2`.

### `framing_lidar`

Percorre uma pasta de nuvens de pontos e publica os arquivos sequencialmente.

### `artificial_zed`

Publica um conjunto de detecções de cones simuladas.

Esses nós são úteis para desenvolvimento e testes, pois permitem reproduzir entradas conhecidas para o pipeline.

---

## 13. Parâmetros

Os principais parâmetros do pipeline estão concentrados em:

```text
constants.py
```

Entre eles estão os limites da ROI, parâmetros do MLESAC, distância de clusterização, parâmetros de restauração e dimensões esperadas dos cones.

Ao alterar o comportamento do detector, esse arquivo é um dos primeiros locais que devem ser consultados.

---

## 14. Limitações conhecidas

A documentação deve registrar também as limitações conhecidas do sistema, especialmente aquelas que podem afetar futuras decisões de desenvolvimento.

O README do projeto `lidar-perception-node` destaca, entre outros pontos, a necessidade de melhorias na fusão entre câmera e LiDAR, no gerenciamento do estado dos sensores e no tratamento de movimento da nuvem de pontos em velocidades maiores.

Esses pontos devem ser considerados juntamente com o roadmap da microdivisão.