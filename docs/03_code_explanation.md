# Pipeline de Percepção — LiDAR, ZED e YOLO

Este documento descreve o funcionamento do software de percepção utilizado pelo Driverless da Unicamp E-Racing.

O objetivo é registrar como os dados provenientes do **LiDAR LeiShen CH128X1** e da **câmera ZED 2i** são processados para obter as detecções dos cones da pista, além de explicar como os principais módulos do código se relacionam.

O documento possui caráter de transferência de conhecimento: um novo membro deve conseguir utilizá-lo para compreender a arquitetura atual e localizar no código cada etapa do processamento.

---

## 1. Visão geral

O sistema de percepção utiliza principalmente dois sensores:

- **LiDAR LeiShen CH128X1**, utilizado para obter uma nuvem de pontos tridimensional;
- **ZED 2i**, utilizada para detectar os cones na imagem e fornecer sua posição e classificação de cor.

A arquitetura geral pode ser resumida como:

```text
                    PERCEPÇÃO
                        │
             ┌──────────┴──────────┐
             │                     │
           LiDAR                  ZED 2i
             │                     │
             ▼                     ▼
        PointCloud2             imagem
             │                     │
             │                  YOLOv8
             │                     │
             │                     ▼
             │              detecções dos cones
             │                     │
             └──────────┬──────────┘
                        │
                  Fusão sensorial
                        │
                        ▼
                  cones detectados
```

De forma mais detalhada, o processamento do LiDAR segue:

```text
PointCloud2
    │
    ▼
ROI
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
Clusters considerados cones
    │
    ├───────────────────────┐
    │                       │
    ▼                       ▼
LiDAR only             LiDAR + ZED
    │                       │
    │                       ▼
    │                 associação com
    │                    detecções
    │                       │
    │                       ▼
    │                  cor da ZED
    │                       │
    └───────────┬───────────┘
                ▼
         saída final dos cones
```

O nó principal responsável por coordenar esse processo é o `lidar_node`, implementado em `main.py`.

---

# 2. Arquitetura do código

Os principais módulos utilizados pelo processamento são:

| Arquivo | Responsabilidade |
|---|---|
| `main.py` | Nó ROS 2 principal e coordenação do pipeline |
| `constants.py` | Parâmetros e constantes utilizados pelo processamento |
| `roi.py` | Filtragem da região de interesse |
| `MLESAC.py` | Estimativa e remoção do plano do chão |
| `clustering.py` | Clusterização e cálculo dos centroides |
| `geometric_filters.py` | Extração das características dos clusters e classificação geométrica |
| `fusion_engine.py` | Processamento das informações da ZED e fusão com o LiDAR |
| `transform.py` | Transformação das coordenadas da ZED para o referencial utilizado pelo LiDAR |
| `artificial_lidar.py` | Publicação de nuvem de pontos artificial |
| `artificial_framing_lidar.py` | Reprodução de nuvens de pontos armazenadas |
| `artificial_zed.py` | Publicação de detecções artificiais da ZED |
| `auxiliar_tools.py` | Funções auxiliares utilizadas pelo pipeline |

O `main.py` instancia o nó ROS 2, recebe as mensagens dos sensores, decide qual modo de operação deve ser utilizado e executa as etapas de processamento.

---

# 3. Entradas e saídas

## 3.1 Entrada do LiDAR

O LiDAR publica uma mensagem ROS 2 do tipo:

```text
sensor_msgs/msg/PointCloud2
```

Essa mensagem contém a nuvem de pontos tridimensional observada pelo sensor.

O tópico utilizado pelo `lidar_node` é definido em:

```python
LIDAR_TOPIC_TO_BE_SUBSCRIBED
```

no arquivo `constants.py`.

Depois de recebida, a mensagem `PointCloud2` é convertida para uma representação NumPy contendo pontos no formato:

```text
[x, y, z]
```

---

## 3.2 Entrada da ZED

Quando a informação da câmera está disponível, o `lidar_node` também recebe as detecções produzidas pela ZED.

O tópico utilizado é definido por:

```python
CAMERA_TOPIC_TO_BE_SUBSCRIBED
```

A informação utilizada pelo processamento possui o formato:

```text
[x, y, color]
```

onde:

- `x` e `y` representam a posição do cone utilizada pelo pipeline;
- `color` representa a classificação de cor fornecida pela câmera.

Antes de ser utilizada pelo pipeline LiDAR, essa informação passa pela transformação definida em `transform.py`.

---

## 3.3 Saída

As detecções finais são publicadas pelo `lidar_node` como:

```text
std_msgs/msg/Float32MultiArray
```

A informação é organizada em grupos de três valores:

```text
[x, y, color]
```

Para vários cones:

```text
[x1, y1, color1,
 x2, y2, color2,
 x3, y3, color3,
 ...]
```

O tópico utilizado para essa publicação é:

```text
final_array
```

Durante o modo de depuração, o nó também pode publicar informações utilizadas para visualização no RViz2.

---

# 4. Gerenciamento dos sensores

O `lidar_node` recebe dados do LiDAR e da câmera de forma assíncrona.

As mensagens mais recentes de cada sensor são armazenadas e um temporizador verifica periodicamente se esses dados ainda são considerados recentes.

O sistema possui quatro estados:

```text
                    LiDAR válido?
                    /           \
                  sim            não
                  /                \
        câmera válida?        câmera válida?
          /       \              /       \
        sim       não          sim       não
         │         │             │         │
       BOTH    LIDAR ONLY   CAMERA ONLY  NO SENSOR
```

### `BOTH`

Os dois sensores estão disponíveis.

Nesse estado, o pipeline utiliza a informação da câmera para auxiliar o processamento do LiDAR.

### `LIDAR_ONLY`

Somente o LiDAR está disponível.

O pipeline continua realizando a detecção utilizando apenas a nuvem de pontos do sensor.

### `CAMERA_ONLY`

Somente a câmera está disponível.

O estado existe na máquina de estados do nó, embora o processamento completo de cones nesse modo dependa da implementação disponível para a câmera.

### `NO_SENSOR`

Nenhum dos sensores é considerado disponível.

---

# 5. Pipeline LiDAR

Quando o LiDAR está disponível, a nuvem passa pelas etapas abaixo:

```text
PointCloud2
     │
     ▼
Conversão para XYZ
     │
     ▼
ROI
     │
     ▼
MLESAC
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
Cones
```

Quando a ZED também está disponível, existe uma etapa adicional de filtragem baseada nas posições fornecidas pela câmera antes do MLESAC.

---

# 6. Região de Interesse (ROI)

A primeira etapa reduz a quantidade de pontos que será processada.

A função `ROIfilter()` aplica um filtro espacial utilizando limites mínimos e máximos para as três coordenadas:

```text
xmin ≤ x ≤ xmax
ymin ≤ y ≤ ymax
zmin ≤ z ≤ zmax
```

Os limites são definidos em `constants.py`.

Os parâmetros utilizados atualmente incluem:

```text
XRMIN
XRMAX
YRMIN
YRMAX
ZRMIN
ZRMAX
```

O objetivo é descartar pontos que estão fora da região onde os cones podem ser relevantes.

---

## 6.1 ROI baseada na ZED

Quando existem detecções da ZED, o processamento utiliza também as posições fornecidas pela câmera.

Para cada cone detectado pela ZED, é considerada uma região quadrada ao seu redor.

O parâmetro:

```python
POINT_CONE_DISTANCE
```

define a distância utilizada para selecionar os pontos do LiDAR próximos às detecções.

A ideia é:

```text
                 ZED
                  │
                  ▼
           posição do cone
                  │
          ┌───────┴───────┐
          │      ROI      │
          │               │
          │   • • • •     │
          │   • cone •    │
          │   • • • •     │
          └───────────────┘
                  │
                  ▼
          pontos do LiDAR
       próximos ao cone
```

Essa etapa permite concentrar o processamento LiDAR nas regiões nas quais a ZED detectou cones.

---

# 7. Remoção do chão — MLESAC

Depois da filtragem inicial, a nuvem ainda contém pontos pertencentes ao solo.

O módulo `MLESAC.py` estima um plano que representa o chão e remove os pontos próximos a esse plano.

O processo pode ser resumido como:

```text
Nuvem após ROI
       │
       ▼
Filtro espacial para estimativa
       │
       ▼
Voxel Grid
       │
       ▼
Geração de planos candidatos
       │
       ▼
Avaliação de verossimilhança
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

A implementação utiliza uma região específica da nuvem para realizar a estimativa do plano e utiliza downsampling por voxel grid para reduzir a quantidade de pontos processados.

Os principais parâmetros estão em `constants.py`:

```text
XMIN
XMAX
YMIN
YMAX
VOXEL_LEAF_SIZE
NUM_PLANES
DOWNSAMPLE_SIZE
INLIER_PROB
P_OUTLIER
DIST2PLANE_THRESHOLD
INLIER_STD_DEV
```

---

# 8. Clusterização

Depois da remoção do chão, os pontos restantes são agrupados em conjuntos espacialmente próximos.

O arquivo `clustering.py` utiliza o `DBSCAN` da biblioteca Scikit-learn.

A configuração utilizada é:

```text
eps = RADIUS_THRESHOLD
min_samples = 1
metric = euclidean
algorithm = kd_tree
```

Com `min_samples = 1`, essa implementação é utilizada pela equipe como uma forma de **Euclidean Clustering**, agrupando pontos de acordo com sua conectividade espacial.

O resultado da clusterização é um rótulo para cada ponto:

```text
Point 1 → cluster 0
Point 2 → cluster 0
Point 3 → cluster 1
Point 4 → cluster 1
...
```

Posteriormente, esses rótulos são convertidos em um dicionário:

```text
cluster_id → pontos do cluster
```

O centroide de cada cluster é calculado pela média das coordenadas de seus pontos.

O parâmetro principal dessa etapa é:

```python
RADIUS_THRESHOLD
```

---

# 9. Restauração de pontos

A etapa de remoção do chão pode remover pontos que pertencem à parte inferior dos cones.

Para recuperar parte dessas informações, o pipeline executa uma etapa de restauração.

A partir dos centroides dos clusters, pontos anteriormente removidos junto com o chão são procurados na nuvem original.

O processo utiliza:

```python
RADIUS_RESTORATION
```

para definir a região inicial de busca.

Posteriormente, outras condições geométricas são utilizadas para determinar quais pontos devem realmente ser reinseridos no cluster.

Conceitualmente:

```text
Nuvem antes da remoção do chão
           │
           ├──────── pontos removidos
           │
           ▼
       clusters
           │
           ▼
       centroides
           │
           ▼
 busca por pontos removidos
 próximos aos centroides
           │
           ▼
       clusters
     restaurados
```

---

# 10. Classificação geométrica

Depois da restauração, cada cluster é avaliado para determinar se possui características compatíveis com um cone.

Essa etapa é implementada em `geometric_filters.py`.

Antes da classificação, existe uma quantidade mínima de pontos:

```python
MIN_LEN
```

Clusters com quantidade insuficiente de pontos são descartados.

---

## 10.1 Características utilizadas

O código extrai características geométricas como:

- diâmetro;
- extensão vertical;
- razão altura/base;
- desvio padrão em Z;
- elongação da caixa delimitadora;
- diagonal;
- altura do centroide;
- quantidade de pontos;
- omnivariance.

Essas características são comparadas com valores esperados para clusters que correspondam a cones.

---

## 10.2 Pontuação

Cada característica possui valores de referência e uma tolerância que varia de acordo com a distância do cluster.

As contribuições das características são combinadas para obter uma pontuação final.

O cluster é considerado cone quando essa pontuação ultrapassa o limite utilizado pelo classificador.

O objetivo dessa etapa é eliminar objetos que foram agrupados pela clusterização, mas que não possuem geometria compatível com um cone.

---

# 11. ZED 2i

A ZED 2i é utilizada como segundo sensor do sistema de percepção.

O código da câmera está no repositório da microdivisão de Percepção.

O pipeline visual pode ser resumido como:

```text
ZED 2i
   │
   ▼
captura da imagem
   │
   ▼
YOLOv8
   │
   ▼
bounding boxes
   │
   ▼
ZED SDK
   │
   ▼
posição dos cones
   +
confiança
   +
classe/cor
   │
   ▼
informação utilizada
pelo pipeline LiDAR
```

---

## 11.1 Inicialização

A implementação da ZED configura parâmetros como:

- frequência de captura;
- resolução;
- modo de profundidade;
- unidade das coordenadas;
- distância mínima de profundidade;
- distância máxima de profundidade.

A ZED também é configurada para receber objetos a partir de **custom bounding boxes**.

Essas bounding boxes são produzidas pelo modelo YOLO.

---

## 11.2 Inferência

Durante a execução:

1. uma imagem é capturada pela ZED;
2. a imagem é fornecida ao modelo YOLO;
3. o modelo retorna as detecções;
4. as bounding boxes são convertidas para o formato esperado pelo SDK da ZED;
5. a ZED processa os objetos;
6. as informações relevantes são extraídas para utilização pelo sistema.

As detecções possuem uma confiança associada.

Existe um limite mínimo:

```python
DISCARD_THRESHOLD
```

Detecções abaixo desse valor são descartadas.

---

## 11.3 Informação produzida pela ZED

A informação utilizada na integração com o LiDAR é representada como:

```text
[x, y, color]
```

As coordenadas são utilizadas para localizar regiões de interesse na nuvem do LiDAR.

A classificação também é utilizada posteriormente para determinar a cor do cone na saída final.

---

# 12. YOLOv8

A detecção de cones na imagem utiliza o modelo **YOLOv8x**.

Esse modelo substituiu a YOLOv5 anteriormente utilizada pela equipe.

O treinamento atual utiliza um dataset sintético produzido com auxílio do Blender.

---

## 12.1 Dataset

O dataset anterior possuía aproximadamente 700 mil labels distribuídas entre cones azuis, amarelos e laranjas.

O dataset atualmente utilizado possui aproximadamente **68 milhões de labels**, criadas sinteticamente.

O dataset está armazenado no Google Drive da equipe:

[Dataset de cones — Google Drive](https://drive.google.com/drive/folders/1o_lj_FDLIeJCDfVL1FtbH7qVvsN7JF_i)

---

## 12.2 Treinamento

O treinamento é realizado em uma GPU NVIDIA Quadro RTX 6000 com 24 GB de VRAM.

O processo documentado pela equipe possui três etapas principais:

```text
Dataset
   │
   ▼
Tuning dos hiperparâmetros
   │
   ▼
Pré-treinamento
   │
   ▼
Treinamento final
   │
   ▼
conesModel.pt
```

### Tuning

Antes do treinamento principal, são buscados hiperparâmetros adequados ao dataset.

### Pré-treinamento

O modelo utilizado como ponto de partida é:

```text
yolov8x.pt
```

Parâmetros registrados para essa etapa:

```text
epochs = 300
imgsz = 640
batch = 20
cache = ram
```

Após essa etapa, o `last.pt` é utilizado como modelo de partida para o treinamento final.

### Treinamento final

Parâmetros registrados:

```text
epochs = 300
imgsz = 1280
batch = 8
cache = disk
```

O modelo resultante é:

```text
conesModel.pt
```

---

## 12.3 Teste do modelo

O projeto possui o script:

```text
Codes/yoloTest.py
```

que carrega o `conesModel.pt`, obtém imagens de uma câmera e executa a inferência.

A saída é visualizada sobre a imagem para inspeção das detecções.

---

## 12.4 Exportação para ONNX

O script:

```text
Codes/yolotoonnx.py
```

exporta o modelo para ONNX utilizando:

```python
model.export(format="onnx", opset=12)
```

O modelo exportado pode ser utilizado como parte das etapas de implantação e otimização para o hardware NVIDIA.

---

## 12.5 Otimização para Jetson

O projeto prevê a utilização de TensorRT para otimizar a inferência do modelo em hardware NVIDIA.

O modelo exportado utilizado nesse processo é:

```text
conesModel.onnx
```

Os detalhes de implantação do modelo na Jetson devem ser mantidos junto da documentação específica do ambiente de execução.

---

# 13. Fusão LiDAR–ZED

A integração entre os dois sensores ocorre principalmente de duas formas:

1. as **coordenadas da ZED são utilizadas para restringir o processamento do LiDAR**;
2. a **classificação de cor da ZED é utilizada na saída das detecções do LiDAR**.

Portanto, a ZED não é utilizada apenas depois que o LiDAR terminou seu processamento.

A posição fornecida pela câmera influencia o processamento desde a etapa de ROI.

---

## 13.1 Utilização das coordenadas

O fluxo é:

```text
                 ZED
                  │
                  ▼
          detecção dos cones
                  │
                  │ coordenadas
                  ▼
         transformação ZED→LiDAR
                  │
                  ▼
          ROI ao redor dos cones
                  │
                  ▼
                 LiDAR
                  │
                  ▼
        processamento da nuvem
```

O parâmetro:

```python
POINT_CONE_DISTANCE
```

determina a proximidade utilizada para selecionar os pontos do LiDAR ao redor das posições fornecidas pela câmera.

Isso reduz a região da nuvem que será processada e concentra o detector nas regiões onde a ZED encontrou cones.

---

## 13.2 Utilização da cor

Depois do processamento LiDAR, os clusters classificados como cones são comparados com as detecções fornecidas pela ZED.

A associação utiliza a proximidade espacial entre as posições.

Quando existe correspondência, a cor fornecida pela ZED é atribuída ao cone detectado pelo LiDAR.

O resultado final mantém o formato:

```text
[x, y, color]
```

---

## 13.3 Referenciais

As coordenadas da ZED são processadas por:

```text
transform.py
```

para serem utilizadas no referencial adotado pelo pipeline LiDAR.

A implementação atual de `Transform` trabalha com uma transformação no plano XY e mantém a informação de cor junto das coordenadas.

Qualquer alteração nessa transformação deve ser tratada junto da documentação de calibração entre os dois sensores.

---

# 14. Modo LiDAR-only

Quando a câmera não está disponível, o pipeline continua podendo executar o processamento do LiDAR.

Nesse caso:

```text
LiDAR
  │
  ▼
ROI
  │
  ▼
MLESAC
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
Cones LiDAR
```

Como não existe uma classificação de cor fornecida pela ZED, o sistema possui uma heurística para produzir a cor dos cones a partir da posição obtida pelo LiDAR.

---

# 15. Dados artificiais

O pacote possui nós auxiliares para executar o pipeline com dados previamente preparados.

Eles são importantes para desenvolvimento, testes e reprodução de casos sem depender continuamente dos sensores físicos.

## `artificial_lidar`

Carrega uma nuvem de pontos armazenada em arquivo e publica essa informação como `PointCloud2`.

Tópico:

```text
artificialLIDAR
```

## `framing_lidar`

Percorre uma pasta contendo nuvens de pontos armazenadas e publica os arquivos sequencialmente.

Tópico:

```text
framingLIDAR
```

Esse modo permite reproduzir uma sequência de casos previamente registrados.

## `artificial_zed`

Publica um conjunto de detecções simuladas da câmera.

Tópico:

```text
artificial_zed
```

Esses nós permitem testar o pipeline utilizando entradas conhecidas e são especialmente úteis durante o desenvolvimento sem acesso ao carro.

---

# 16. Parâmetros importantes

Os principais parâmetros utilizados pelo pipeline estão concentrados em:

```text
constants.py
```

Alguns dos grupos mais importantes são:

### ROI

```text
XRMIN
XRMAX
YRMIN
YRMAX
ZRMIN
ZRMAX
POINT_CONE_DISTANCE
```

### MLESAC

```text
XMIN
XMAX
YMIN
YMAX
VOXEL_LEAF_SIZE
NUM_PLANES
DOWNSAMPLE_SIZE
INLIER_PROB
P_OUTLIER
DIST2PLANE_THRESHOLD
INLIER_STD_DEV
```

### Clusterização

```text
RADIUS_THRESHOLD
```

### Restauração

```text
RADIUS_RESTORATION
```

### Geometria do cone

```text
CONE_WIDTH
CONE_HEIGHT
```

Ao investigar por que um cone não está sendo detectado, esses parâmetros são alguns dos primeiros pontos a serem consultados.

---

# 17. Publicações para depuração

Quando:

```python
DEBUGGING_MODE = True
```

o nó publica informações adicionais utilizadas para inspeção durante o desenvolvimento.

Isso permite visualizar partes do resultado do processamento no RViz2 e acompanhar o comportamento da nuvem após as etapas do pipeline.

Esse modo é especialmente útil para verificar:

- região de interesse;
- pontos restantes após a remoção do chão;
- resultado do processamento;
- posição dos cones detectados.

---

# 18. Sincronização temporal

O processamento depende de informações provenientes de sensores diferentes.

Por isso, a sincronização temporal é relevante principalmente em situações nas quais o veículo está em movimento.

A documentação específica do procedimento de sincronização entre o LiDAR e a NVIDIA Jetson está em:

```text
06_sensor_synchronization_ptp.md
```

Esse documento deve ser consultado quando o objetivo for configurar ou investigar a sincronização temporal do sistema.

A sincronização temporal e a calibração espacial são problemas distintos:

```text
Sincronização temporal
        │
        ▼
"Quando o sensor observou o objeto?"

Calibração espacial
        │
        ▼
"Em que posição o objeto está no referencial do outro sensor?"
```

Os dois são relevantes para uma associação correta entre LiDAR e ZED.

---

# 19. Limitações conhecidas

O sistema possui limitações e pontos identificados para evolução futura.

Entre eles:

- calibração entre LiDAR e ZED;
- arquitetura de fusão sensorial;
- gerenciamento do estado dos sensores;
- processamento da nuvem durante movimentos em velocidades elevadas;
- necessidade potencial de deskewing;
- possibilidades de otimização da implementação.

Esses pontos são discutidos com mais detalhes em:

```text
05_roadmap_future.md
```

O objetivo desta documentação é registrar o estado atual do sistema; alterações futuras devem ser incorporadas à documentação conforme forem implementadas.

---

# 20. Onde procurar no código

Quando for necessário investigar uma parte específica da percepção:

| Problema / assunto | Arquivo principal |
|---|---|
| Entrada e execução do pipeline | `main.py` |
| Tópicos e parâmetros | `constants.py` |
| ROI | `roi.py` |
| Remoção do chão | `MLESAC.py` |
| Clustering | `clustering.py` |
| Classificação dos clusters | `geometric_filters.py` |
| Fusão LiDAR–ZED | `fusion_engine.py` |
| Transformação das coordenadas | `transform.py` |
| Dados artificiais do LiDAR | `artificial_lidar.py` |
| Reprodução de nuvens | `artificial_framing_lidar.py` |
| Dados artificiais da ZED | `artificial_zed.py` |
| YOLO / câmera | repositório da ZED no GitLab |

---

## 21. Resumo do funcionamento

Em uma frase, o sistema pode ser entendido como:

> **A ZED identifica onde estão os cones e suas cores; o LiDAR utiliza essas posições para concentrar o processamento da nuvem, identifica geometricamente os objetos e, quando possível, recebe da ZED a classificação de cor.**

Em forma de fluxo:

```text
                       ZED 2i
                          │
                       YOLOv8
                          │
                posição + classe/cor
                          │
                          ▼
LiDAR ──► PointCloud2 ──► ROI orientada pela ZED
                          │
                          ▼
                     MLESAC
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
                  cones detectados
                          │
                          ▼
                  associação com ZED
                          │
                          ▼
                     [x, y, cor]
```