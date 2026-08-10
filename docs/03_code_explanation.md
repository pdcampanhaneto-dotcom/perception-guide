# Pipeline do Nó de Processamento LiDAR (`lidar-perception-node`)

O pacote de percepção LiDAR realiza a filtragem, segmentação, restauração geométrica e classificação dos cones de pista. A fusão sensorial é feita combinando as métricas 3D do LiDAR com a classificação de cor da câmera ZED 2i.

---

## 1. Máquina de Estados e Sincronização Sensorial

Como a taxa de atualização da câmera ZED 2i e do LiDAR **LeiShen CH128X1** são distintas, o sistema executa um loop de decisão assíncrono baseado em temporizador.

Dada a hora atual $t_{\text{curr}}$ e os carimbos de data/hora das últimas mensagens recebidas ($t_{\text{lidar}}$ e $t_{\text{camera}}$), definimos um limiar de frescor máximo $\Delta t_{\text{max}}$:

$$Valid_{\text{lidar}} = \begin{cases}  \text{True}, & \text{se } (t_{\text{curr}} - t_{\text{lidar}}) < \Delta t_{\text{max}} \\  \text{False}, & \text{caso contrário}  \end{cases}$$

$$Valid_{\text{camera}} = \begin{cases}  \text{True}, & \text{se } (t_{\text{curr}} - t_{\text{camera}}) < \Delta t_{\text{max}} \\  \text{False}, & \text{caso contrário}  \end{cases}$$

O sistema transiciona dinamicamente entre quatro estados $S$:

$$S = \begin{cases} \text{BOTH}, & \text{se } Valid_{\text{lidar}} \land Valid_{\text{camera}} \\ \text{LIDAR ONLY}, & \text{se } Valid_{\text{lidar}} \land \neg Valid_{\text{camera}} \\ \text{CAMERA ONLY}, & \text{se } \neg Valid_{\text{lidar}} \land Valid_{\text{camera}} \\ \text{NO SENSOR}, & \text{se } \neg Valid_{\text{lidar}} \land \neg Valid_{\text{camera}} \end{cases}$$

---

## 2. Etapas do Pipeline de Processamento (6 Estágios)

```text
[Entrada PointCloud2] 
       │
       ▼
[1. ROI Filter (Prisma + Square)]
       │
       ▼
[2. Floor Removal (MLESAC)]
       │
       ▼
[3. Fast Euclidean Clustering (DBSCAN)]
       │
       ▼
[4. Restauração Cilíndrica]
       │
       ▼
[5. Filtros Geométricos (Score Gaussiano)]
       │
       ▼
[6. Coloração & Publicação (Float32MultiArray)]
```
### Passo 1: Filtragem da Região de Interesse (ROI)

Dado o conjunto de pontos bruto $P$, aplicamos um filtro de prisma retangular gerando o conjunto reduzido $F$:
$$F = \{ (x_i, y_i, z_i) \in P \mid x_{\min} \le x_i \le x_{\max} ; y_{\min} \le y_i \le y_{\max} ; z_{\min} \le z_i \le z_{\max} \}$$

Quando há detecções da câmera ZED 2i disponíveis, o ROI é refinado: para cada cone estimado pela visão, retém-se apenas os pontos dentro de um quadrado de lado $POINT\_CONE\_DISTANCE$ centrado na posição do cone.

### Passo 2: Remoção do Solo com MLESAC

Utiliza-se o MLESAC (Maximum Likelihood Estimation Sample Consensus) em uma versão simplificada da nuvem de pontos $F'$ (obtida via amostragem por Voxel Grid).Para cada plano candidato $\theta_i$, avalia-se a verossimilhança dos pontos inliers através de uma distribuição Gaussiana:
$$L_{\text{inlier}}(D \mid \theta_i) = \frac{1}{\sqrt{2\pi \sigma^2}} \exp\left(-\frac{\text{dist}^2}{2\sigma^2}\right)$$
O plano do solo ideal $\theta_{\text{best}}$ é o que maximiza o log da verossimilhança. Os pontos com distância do plano menor que o limiar estipulado são removidos.

### Passo 3: Clusterização Euclidiana Rápida
Aplica-se uma variação do algoritmo DBSCAN. Definindo o parâmetro minPts = 1, o algoritmo torna-se equivalente à clusterização por conectividade espacial euclidiana pura.

### Passo 4: Restauração Cilíndrica
Cones que perderam pontos na base durante a etapa de remoção do solo são restaurados. Para cada centroide de cluster $(C_x, C_y, C_z)$, busca-se na nuvem original $F'$ os pontos que estão em um volume cilíndrico vertical:$$R = \{ (x_i, y_i, z_i) \in F' \mid \sqrt{(C_x - x_i)^2 + (C_y - y_i)^2} \le \text{XY}_{\text{THRESHOLD}} \land Z_{\min} \le C_z \le Z_{\max} \}$$

### Passo 5: Filtros Geométricos e Pontuação Gaussiana
Cada cluster é avaliado segundo as dimensões regulamentares de cones da Formula Student (altura, diâmetro, razão altura/base, elongação AABB, comprimento da diagonal AABB, quantidade de pontos e altura do centroide).

Para cada métrica, calcula-se um score Gaussiano usando médias e desvios padrão calibrados. Os scores são somados para definir a probabilidade de o objeto ser um cone, aplicando tolerâncias ajustadas para cones distantes.

### Passo 6: Coloração e Publicação
Associa-se cada cluster LiDAR à cor do cone da câmera mais próximo por distância euclidiana. A saída final é publicada em uma mensagem do tipo std_msgs/msg/Float32MultiArray contendo o vetor sequencial $[x_1, y_1, \text{color}_1, x_2, y_2, \text{color}_2, \dots]$.