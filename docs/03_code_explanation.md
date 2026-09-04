# Pipeline de Percepção — LiDAR, ZED e YOLO

Este documento descreve o funcionamento do software de percepção utilizado pelo Driverless da Unicamp E-Racing.

O objetivo é registrar como os dados provenientes do **LiDAR LeiShen CH128X1** e da **câmera ZED 2i** são processados para obter as detecções dos cones da pista, além de explicar como os principais módulos do código se relacionam.

O documento possui caráter de transferência de conhecimento: um novo membro deve conseguir utilizá-lo para compreender a arquitetura atual, localizar cada etapa no código e entender quais informações são trocadas entre os sensores e os nós ROS 2.

---

## 1. Visão geral

O sistema de percepção utiliza principalmente dois sensores:

- **LiDAR LeiShen CH128X1**, utilizado para obter uma nuvem de pontos tridimensional;
- **ZED 2i**, utilizada para detectar cones na imagem e fornecer informações espaciais e de classificação que são utilizadas pelo processamento.

A arquitetura geral pode ser resumida como:

```text
                         PERCEPÇÃO
                             │
             ┌───────────────┴───────────────┐
             │                               │
           LiDAR                            ZED 2i
             │                               │
             ▼                               ▼
        PointCloud2                       imagem
             │                               │
             │                            YOLO
             │                               │
             │                               ▼
             │                       bounding boxes
             │                               │
             │                               ▼
             │                           ZED SDK
             │                               │
             │                         posição + cor
             │                               │
             └───────────────┬───────────────┘
                             ▼
                      Processamento
                      / Fusão sensorial
                             │
                             ▼
                       cones detectados
```

O pipeline LiDAR possui, de forma simplificada, as seguintes etapas:

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
    │             associação com
    │              detecções ZED
    │                       │
    │                       ▼
    │                  cor da ZED
    │                       │
    └────────────┬──────────┘
                 ▼
          saída dos cones
```

O nó `lidar_node`, implementado em `main.py`, coordena o recebimento dos dados e as etapas do processamento.

---

# 2. Arquitetura do código

Os principais módulos do processamento LiDAR são:

| Arquivo | Responsabilidade |
|---|---|
| `main.py` | Nó ROS 2 principal e coordenação do pipeline |
| `constants.py` | Parâmetros e constantes utilizados pelo processamento |
| `roi.py` | Filtragem da região de interesse |
| `MLESAC.py` | Estimativa e remoção do plano do chão |
| `clustering.py` | Clusterização e cálculo dos centroides |
| `geometric_filters.py` | Extração das características dos clusters e classificação geométrica |
| `fusion_engine.py` | Operações relacionadas à fusão LiDAR–ZED |
| `transform.py` | Transformação das coordenadas fornecidas pela ZED |
| `artificial_lidar.py` | Publicação de uma nuvem de pontos previamente armazenada |
| `artificial_framing_lidar.py` | Reprodução sequencial de nuvens armazenadas |
| `artificial_zed.py` | Publicação de detecções ZED simuladas |
| `auxiliar_tools.py` | Funções auxiliares |

Na parte da ZED, os principais arquivos fornecidos são:

| Arquivo | Responsabilidade |
|---|---|
| `cameraProcessing.py` | Inicialização da ZED, inferência YOLO e obtenção das detecções |
| `main.py` | Interface simples para criação da câmera e execução da medição |
| `dataProcessing.py` | Nó ROS 2 que publica as detecções produzidas pela ZED |

---

# 3. Entradas e saídas

## 3.1 Entrada do LiDAR

O processamento recebe uma mensagem ROS 2 do tipo:

```text
sensor_msgs/msg/PointCloud2
```

Essa mensagem representa a nuvem de pontos tridimensional.

O tópico consumido pelo `lidar_node` é definido em:

```python
LIDAR_TOPIC_TO_BE_SUBSCRIBED
```

no arquivo `constants.py`.

A mensagem é convertida para uma representação NumPy contendo pontos no formato:

```text
[x, y, z]
```

---

## 3.2 Entrada da ZED no processamento LiDAR

O `lidar_node` também possui uma assinatura para receber as detecções da câmera.

O tópico é definido por:

```python
CAMERA_TOPIC_TO_BE_SUBSCRIBED
```

no arquivo `constants.py`.

No ambiente de teste do processamento LiDAR, uma das fontes possíveis é o nó `artificial_zed`, mas a implementação real da câmera possui seu próprio nó ROS 2, descrito na seção da ZED.

As informações utilizadas pelo processamento possuem o formato:

```text
[x, y, color]
```

Antes de serem utilizadas pelo pipeline LiDAR, essas coordenadas passam pela transformação definida em `transform.py`.

---

## 3.3 Saída do pipeline LiDAR

As detecções finais do `lidar_node` são publicadas como:

```text
std_msgs/msg/Float32MultiArray
```

no tópico:

```text
final_array
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

---

## 3.4 Saída da ZED

O nó ROS 2 da câmera publica suas detecções no tópico:

```text
camera_cones
```

utilizando:

```text
std_msgs/msg/Float32MultiArray
```

A mensagem contém os dados produzidos pelo pipeline da câmera em formato achatado.

Cada detecção é representada por:

```text
[x, y, color]
```

O arquivo `dataProcessing.py` converte o resultado de `cameraProcessing.py` em uma mensagem ROS 2 e publica o número de cones detectados. 

---

# 4. Gerenciamento dos sensores

O `lidar_node` recebe dados do LiDAR e da câmera de forma assíncrona.

As mensagens mais recentes são armazenadas e um temporizador verifica periodicamente se os dados são suficientemente recentes.

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

Nesse estado, as informações da ZED são utilizadas para auxiliar o processamento do LiDAR.

### `LIDAR_ONLY`

Somente o LiDAR está disponível.

O pipeline de processamento da nuvem continua sendo executado.

### `CAMERA_ONLY`

Somente a câmera está disponível.

Esse estado existe na máquina de estados do `lidar_node`, embora o processamento específico desse modo dependa da implementação utilizada no restante do sistema.

### `NO_SENSOR`

Nenhum dos sensores é considerado disponível.

---

# 5. Pipeline do LiDAR

Quando o LiDAR está disponível, o fluxo principal é:

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

Quando a ZED está disponível, suas coordenadas são utilizadas na etapa de ROI para concentrar o processamento em regiões próximas às detecções da câmera.

---

# 6. Região de Interesse (ROI)

A função `ROIfilter()` aplica um filtro espacial utilizando limites mínimos e máximos para:

```text
xmin ≤ x ≤ xmax
ymin ≤ y ≤ ymax
zmin ≤ z ≤ zmax
```

Os valores são definidos em `constants.py` por:

```text
XRMIN
XRMAX
YRMIN
YRMAX
ZRMIN
ZRMAX
```

O objetivo é retirar pontos que estão fora da região relevante para a detecção dos cones.

---

## 6.1 ROI baseada nas detecções da ZED

Quando a ZED fornece detecções, a função `ROIonZED()` utiliza suas posições para restringir ainda mais a nuvem.

O parâmetro:

```python
POINT_CONE_DISTANCE
```

define a distância utilizada ao redor das posições fornecidas pela câmera.

O fluxo é:

```text
                  ZED
                   │
             cone detectado
                   │
              posição [x,y]
                   │
                   ▼
       ┌─────────────────────┐
       │ região de interesse │
       │          ●          │
       └─────────────────────┘
                   │
                   ▼
             pontos LiDAR
```

Essa é uma das principais formas pelas quais a informação da ZED é utilizada para melhorar o processamento LiDAR.

---

# 7. Remoção do chão — MLESAC

Depois da ROI, a nuvem ainda contém pontos pertencentes ao chão.

O módulo `MLESAC.py` estima um plano que representa o solo e remove os pontos próximos a esse plano.

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
Planos candidatos
      │
      ▼
Avaliação de verossimilhança
      │
      ▼
Melhor plano
      │
      ▼
Remoção do chão
      │
      ▼
Nuvem sem o chão
```

Os principais parâmetros são:

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

Após a remoção do chão, os pontos restantes são agrupados de acordo com sua proximidade espacial.

A implementação está em `clustering.py` e utiliza o `DBSCAN` da biblioteca Scikit-learn.

A configuração utilizada é:

```text
eps = RADIUS_THRESHOLD
min_samples = 1
metric = euclidean
algorithm = kd_tree
```

Com `min_samples = 1`, a implementação é utilizada pela equipe como uma forma de **Euclidean Clustering**.

O resultado é um rótulo para cada ponto:

```text
Point 1 → cluster 0
Point 2 → cluster 0
Point 3 → cluster 1
...
```

Depois, os pontos são agrupados em um dicionário:

```text
cluster_id → pontos do cluster
```

O centroide de cada cluster é calculado pela média de suas coordenadas.

---

# 9. Restauração de pontos

A remoção do chão pode retirar pontos da parte inferior de objetos que devem permanecer nos clusters.

Para recuperar parte dessas informações, o pipeline possui uma etapa de restauração.

A partir dos centroides dos clusters, são procurados pontos que foram removidos junto com o plano do chão.

O parâmetro:

```python
RADIUS_RESTORATION
```

define a região inicial de busca.

Posteriormente, condições adicionais de proximidade são utilizadas para determinar quais pontos serão reinseridos.

---

# 10. Classificação geométrica

Depois da restauração, os clusters são avaliados para verificar se suas características são compatíveis com um cone.

Essa etapa é implementada em `geometric_filters.py`.

Entre as características utilizadas estão:

- diâmetro;
- extensão vertical;
- razão altura/base;
- desvio padrão em Z;
- elongação da caixa delimitadora;
- diagonal;
- altura do centroide;
- quantidade de pontos;
- omnivariance.

Clusters com quantidade de pontos abaixo do limite definido em:

```python
MIN_LEN
```

são descartados.

As características restantes são comparadas com valores de referência e utilizadas para calcular uma pontuação.

O cluster é considerado cone quando a pontuação ultrapassa o limite utilizado pelo classificador.

---

# 11. ZED 2i

A ZED 2i é utilizada para fornecer informações visuais e espaciais dos cones.

O fluxo implementado pelo código da câmera é:

```text
ZED 2i
   │
   ▼
captura da imagem
   │
   ▼
YOLO
   │
   ▼
bounding boxes
   │
   ▼
ZED SDK
   │
   ▼
objetos detectados
   │
   ▼
posição + classe/confiança
   │
   ▼
[x, y, color]
```

---

## 11.1 Inicialização da ZED

O arquivo `cameraProcessing.py` contém a classe `ZED`.

Durante a inicialização são definidos:

- FPS;
- resolução;
- modo de profundidade;
- unidade das coordenadas.

Os valores aceitos para FPS na implementação são:

```text
15
30
60
100
```

Para resolução, a implementação possui as seguintes associações:

| Valor | Resolução ZED |
|---:|---|
| `376` | VGA |
| `720` | HD720 |
| `1080` | HD1080 |
| `1242` | HD2K |

O modo de profundidade utilizado pelo código é:

```text
DEPTH_MODE.ULTRA
```

e as coordenadas são expressas em:

```text
UNIT.METER
```

---

## 11.2 Detecção de objetos

Depois de abrir a câmera, o código habilita a detecção de objetos da ZED utilizando:

```text
CUSTOM_BOX_OBJECTS
```

Isso significa que as bounding boxes utilizadas pelo módulo de detecção são fornecidas por um detector externo.

Nesse projeto, essas bounding boxes são provenientes do modelo YOLO.

O tracking de objetos está desabilitado na implementação atual.

---

# 12. YOLO

### Observação sobre a versão

Há informações de diferentes versões do pipeline visual nos materiais disponíveis.

O documento de treinamento da equipe descreve a utilização atual da **YOLOv8x**, enquanto o código de câmera fornecido nesta documentação utiliza explicitamente **YOLOv5** carregado através do `torch.hub`.

Portanto, a versão efetivamente utilizada no ambiente de produção deve ser confirmada antes de reproduzir a instalação.

A descrição desta seção, quando relacionada ao código abaixo, refere-se ao comportamento do código fornecido.

---

## 12.1 Carregamento do modelo no código da câmera

No `cameraProcessing.py`, o modelo é carregado através de:

```python
torch.hub.load(
    ".../yolov5",
    "custom",
    path=".../1280.pt",
    source="local"
)
```

Assim, essa implementação depende de uma cópia local do repositório YOLOv5 e de um arquivo de pesos chamado `1280.pt`.

Os caminhos presentes no código são específicos da máquina em que esse código foi desenvolvido e devem ser tratados como configuração local.

---

## 12.2 Inferência

Durante cada medição:

1. a ZED captura uma imagem;
2. a imagem é convertida para o formato utilizado pelo modelo;
3. o YOLO executa a inferência;
4. as detecções são obtidas no formato de bounding boxes;
5. cada detecção é convertida para `CustomBoxObjectData`;
6. as bounding boxes são enviadas para o SDK da ZED;
7. o SDK retorna os objetos detectados;
8. a posição e a classificação são extraídas.

O fluxo é:

```text
imagem
  │
  ▼
YOLO
  │
  ▼
bounding boxes
  │
  ▼
CustomBoxObjectData
  │
  ▼
ZED Object Detection
  │
  ▼
Objects
```

---

## 12.3 Confiança e cor

O código utiliza:

```python
NO_COLOR_THRESHOLD = 0.3
```

como limite mínimo para considerar a confiança utilizada na classificação de cor.

A implementação atual utiliza os labels:

```text
0
1
3
```

para determinar a codificação de cor.

A cor é representada numericamente:

- label `0` → valor positivo igual à confiança;
- labels `1` ou `3` → valor negativo igual à confiança;
- demais situações → `0`.

Somente detecções com `color_accuracy != 0` são adicionadas à saída.

---

# 13. Saída da ZED

Depois de processar os objetos, o `cameraProcessing.py` gera uma matriz com três valores por cone:

```text
[x, y, color]
```

No código atual, os valores utilizados para X e Y da saída são obtidos a partir de:

```python
[obj.position[2], obj.position[0]]
```

e a terceira posição armazena o valor de classificação associado à cor.

Exemplo conceitual:

```text
[
    [x1, y1, color1],
    [x2, y2, color2],
    ...
]
```

Caso nenhuma detecção válida seja obtida, a função retorna uma matriz vazia com dimensão `(0, 3)`.

---

# 14. Nó ROS 2 da ZED

O arquivo `dataProcessing.py` transforma a saída da câmera em uma publicação ROS 2.

O nó se chama:

```text
CameraNode
```

e publica:

```text
camera_cones
```

com o tipo:

```text
std_msgs/msg/Float32MultiArray
```

O nó executa o pipeline a cada:

```text
0.1 s
```

ou aproximadamente:

```text
10 Hz
```

O fluxo é:

```text
CameraNode
    │
    ▼
camera_pkg.main
    │
    ▼
ZED.measure()
    │
    ▼
detecções
    │
    ▼
Float32MultiArray
    │
    ▼
camera_cones
```

A mensagem publicada contém o resultado achatado:

```text
[x1, y1, color1, x2, y2, color2, ...]
```

---

# 15. Execução independente da ZED

O arquivo `main.py` do pacote da câmera também permite executar um teste independente do ROS 2.

A câmera é criada utilizando:

```python
ZED(30, 1000)
```

e o resultado de `measure()` é impresso continuamente.

A execução independente possui a finalidade de testar diretamente:

```text
ZED
 ↓
YOLO
 ↓
detecções
```

sem depender do restante da infraestrutura ROS 2.

> **Observação:** a resolução `1000` não está entre as resoluções explicitamente mapeadas pela implementação mostrada. Nesse caso, o próprio código utiliza `720` como valor padrão.

---

# 16. Fusão LiDAR–ZED

A fusão entre os sensores ocorre principalmente em dois momentos.

### 16.1 Coordenadas da ZED → processamento LiDAR

As posições fornecidas pela ZED são utilizadas para restringir a região da nuvem LiDAR analisada.

```text
               ZED
                │
         cone detectado
                │
           posição [x,y]
                │
                ▼
       transformação
       ZED → LiDAR
                │
                ▼
        ROI localizada
                │
                ▼
             LiDAR
                │
                ▼
          processamento
```

O parâmetro:

```python
POINT_CONE_DISTANCE
```

controla a região considerada ao redor das posições fornecidas pela câmera.

### 16.2 Cor da ZED → detecção LiDAR

Depois da classificação geométrica dos clusters, os cones detectados pelo LiDAR são comparados espacialmente com as detecções da ZED.

A associação utiliza a proximidade entre os pontos.

Quando existe correspondência, a informação de cor da ZED é utilizada na saída final.

Portanto, o papel da ZED não é simplesmente "colorir" uma detecção pronta do LiDAR. Sua posição também participa do processamento da nuvem.

---

# 17. Transformação das coordenadas

O módulo:

```text
transform.py
```

é responsável pela transformação das coordenadas fornecidas pela ZED para o referencial adotado pelo processamento.

A estrutura da função utiliza:

```text
[x, y, color]
```

como entrada e preserva a informação de cor durante a transformação.

Qualquer alteração da transformação entre os sensores deve ser registrada junto da documentação de calibração LiDAR–ZED.

---

# 18. Dados artificiais

O pacote de processamento possui fontes artificiais para testes.

## `artificial_lidar`

Carrega uma nuvem de pontos armazenada em arquivo e publica:

```text
artificialLIDAR
```

como `PointCloud2`.

---

## `framing_lidar`

Percorre uma pasta de nuvens de pontos e publica os arquivos sequencialmente em:

```text
framingLIDAR
```

Esse modo permite reproduzir casos gravados anteriormente.

---

## `artificial_zed`

Publica detecções de cones simuladas no tópico:

```text
artificial_zed
```

Esses dados podem ser utilizados para testar a integração com o pipeline LiDAR sem depender da câmera física.

---

# 19. Parâmetros importantes

Os parâmetros do processamento LiDAR estão concentrados em:

```text
constants.py
```

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

Na parte visual, parâmetros como FPS, resolução e `NO_COLOR_THRESHOLD` estão definidos no próprio código da câmera.

---

# 20. Publicações e interfaces ROS 2

As principais interfaces relevantes para a percepção são:

| Nó / componente | Tópico | Tipo | Informação |
|---|---|---|---|
| Driver LiDAR | tópico definido pelo driver | `sensor_msgs/msg/PointCloud2` | Nuvem de pontos |
| `CameraNode` | `camera_cones` | `std_msgs/msg/Float32MultiArray` | Cones detectados pela ZED |
| `lidar_node` | `final_array` | `std_msgs/msg/Float32MultiArray` | Cones finais |
| `artificial_lidar` | `artificialLIDAR` | `sensor_msgs/msg/PointCloud2` | Nuvem artificial |
| `framing_lidar` | `framingLIDAR` | `sensor_msgs/msg/PointCloud2` | Nuvens gravadas |
| `artificial_zed` | `artificial_zed` | `std_msgs/msg/Float32MultiArray` | ZED simulada |

> **Atenção:** os nomes efetivamente utilizados em `constants.py` podem variar conforme o ambiente ou o modo de teste. Verifique sempre a configuração atual antes de executar o pipeline.

---

# 21. Onde procurar no código

| Assunto | Arquivo |
|---|---|
| Execução do pipeline LiDAR | `main.py` |
| Tópicos e parâmetros | `constants.py` |
| ROI | `roi.py` |
| Remoção do chão | `MLESAC.py` |
| Clustering | `clustering.py` |
| Classificação geométrica | `geometric_filters.py` |
| Fusão LiDAR–ZED | `fusion_engine.py` |
| Transformação de coordenadas | `transform.py` |
| Processamento da câmera | `cameraProcessing.py` |
| Interface da câmera | `main.py` da câmera |
| Publicação ROS 2 da câmera | `dataProcessing.py` |
| Modelo YOLO utilizado pelo código fornecido | caminho configurado em `cameraProcessing.py` |
| Teste independente da câmera | `main.py` da câmera |

---

# 22. Sincronização temporal

A percepção utiliza sensores que observam o ambiente em momentos diferentes. Como o veículo está em movimento, diferenças temporais entre as medições podem afetar a associação entre LiDAR e ZED.

A documentação do procedimento de sincronização entre LiDAR e NVIDIA Jetson está em:

```text
06_sensor_synchronization_ptp.md
```

A sincronização temporal deve ser entendida separadamente da calibração espacial:

```text
Sincronização temporal
        │
        ▼
"Quando cada sensor observou o objeto?"

Calibração espacial
        │
        ▼
"Em que posição o sensor representa o objeto?"
```

Ambos são relevantes para uma fusão sensorial consistente.

---

# 23. Limitações e próximos passos

As limitações e possíveis melhorias conhecidas estão documentadas em:

```text
05_roadmap_future.md
```

Entre os pontos já identificados estão:

- calibração LiDAR–ZED;
- arquitetura de fusão sensorial;
- sincronização temporal;
- deskewing;
- gerenciamento do estado dos sensores;
- otimização da inferência e do processamento;
- reprodução completa do ambiente ZED/YOLO.

---

# 24. Resumo

O funcionamento da percepção pode ser resumido como:

```text
                         ZED 2i
                            │
                          imagem
                            │
                           YOLO
                            │
                   bounding boxes
                            │
                       ZED SDK
                            │
                     posição + cor
                            │
                            │
                            ▼
LiDAR ──► PointCloud2 ──► ROI localizada
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
                     cones LiDAR
                            │
                    associação ZED
                            │
                            ▼
                        [x, y, cor]
```

Em termos conceituais:

> **A ZED identifica os cones na imagem e fornece sua posição e classificação; o LiDAR utiliza essas posições para concentrar o processamento da nuvem e identificar geometricamente os objetos. As informações dos dois sensores são então associadas para produzir as detecções finais dos cones.**
