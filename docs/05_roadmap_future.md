# Roadmap — Percepção Driverless

Este documento registra limitações conhecidas do sistema de percepção e possíveis linhas de desenvolvimento para as próximas gerações da equipe.

O objetivo é preservar o conhecimento acumulado durante o desenvolvimento do pipeline e evitar que problemas já identificados precisem ser redescobertos.

As propostas abaixo não devem ser interpretadas como uma lista obrigatória de tarefas. Antes de implementar uma melhoria, o membro responsável deve avaliar o estado atual do código e verificar se a solução continua adequada.

---

## 1. Estado atual

O pipeline atual realiza, de maneira geral:

```text
LiDAR PointCloud2
       ↓
ROI
       ↓
Remoção do solo
       ↓
Clusterização
       ↓
Restauração dos clusters
       ↓
Filtros geométricos
       ↓
Associação com câmera
       ↓
Cones detectados
```

O sistema também possui mecanismos para trabalhar com dados artificiais, permitindo testar o processamento sem depender continuamente dos sensores físicos.

Apesar de funcional, existem limitações conhecidas no processamento, na integração entre sensores e na infraestrutura de software.

---

# 2. Melhorias no processamento LiDAR

## 2.1 Deskewing da nuvem de pontos

Durante a movimentação do veículo, diferentes pontos de uma mesma varredura do LiDAR são adquiridos em instantes diferentes.

Em velocidades mais altas, esse intervalo pode produzir distorções na nuvem de pontos.

Uma possível melhoria é implementar **deskewing**, compensando o movimento do veículo durante a aquisição da varredura.

### Objetivo

Reduzir a distorção da nuvem e melhorar a posição e geometria dos objetos detectados durante o movimento.

### Pontos a investigar

- timestamp dos pontos;
- frequência de aquisição do LiDAR;
- estimativa de movimento do veículo;
- integração com odometria/IMU;
- impacto da correção na detecção dos cones.

---

## 2.2 Melhoria da remoção do solo

O pipeline utiliza MLESAC para estimar o plano do solo.

Os parâmetros utilizados atualmente são calibrados empiricamente em `constants.py`.

Uma possível linha de melhoria é avaliar o comportamento do algoritmo em diferentes condições da pista e do veículo.

Devem ser investigados casos como:

- pista com inclinação;
- irregularidades do solo;
- mudanças de superfície;
- movimento do veículo;
- presença de objetos próximos ao solo.

O objetivo é evitar que pontos pertencentes aos cones sejam removidos junto com o chão ou que partes relevantes da pista sejam classificadas incorretamente.

---

## 2.3 Clusterização

A clusterização atual utiliza DBSCAN com:

```python
min_samples = 1
```

e um raio definido por:

```python
RADIUS_THRESHOLD
```

Essa configuração é utilizada como uma forma de realizar clusterização por conectividade espacial.

Uma possível melhoria é estudar métodos de clusterização mais adequados para nuvens de pontos de LiDAR, principalmente considerando a variação da densidade dos pontos com a distância.

---

# 3. Melhoria dos filtros geométricos

A identificação dos cones atualmente utiliza diversas características geométricas, como:

- diâmetro;
- extensão vertical;
- razão altura/base;
- desvio padrão em Z;
- elongação;
- diagonal;
- altura do centroide;
- número de pontos;
- omnivariance.

Os parâmetros dessas características são definidos manualmente em `geometric_filters.py`.

Uma possível linha de desenvolvimento é tornar essa classificação mais robusta através de:

- novos dados de treinamento;
- análise estatística das características;
- avaliação sistemática de falsos positivos e falsos negativos;
- comparação com classificadores alternativos.

Alterações devem ser avaliadas considerando diferentes distâncias do LiDAR, pois a quantidade e a distribuição dos pontos em um cone mudam conforme a distância.

---

# 4. Fusão LiDAR–ZED

A fusão entre LiDAR e câmera é uma das partes mais importantes do sistema.

Atualmente, as informações da câmera são utilizadas para auxiliar a localização dos cones e para fornecer sua classificação de cor.

A transformação entre os referenciais é realizada pelo módulo `transform.py`.

No estado atual documentado, a transformação implementada é uma transformação no plano XY:

```text
ZED
 ↓
transform.py
 ↓
LiDAR
```

Uma limitação conhecida é a precisão da estimativa da transformação extrínseca entre os sensores.

Uma transformação imprecisa pode fazer com que a posição de um cone detectado pela câmera não coincida corretamente com os pontos correspondentes no LiDAR.

---

## 4.1 Calibração extrínseca

Uma possível melhoria é desenvolver um procedimento mais preciso e reproduzível para estimar a transformação entre:

```text
referencial da ZED
        ↕
referencial do LiDAR
```

O objetivo é permitir uma associação espacial mais precisa entre os sensores.

Antes de alterar o algoritmo de fusão, deve-se verificar:

- posição física dos sensores;
- orientação dos sensores;
- definição dos referenciais;
- matriz de transformação utilizada;
- método utilizado para obter essa matriz;
- precisão necessária para a aplicação.

---

## 4.2 Associação câmera–LiDAR

O sistema atual utiliza uma região ao redor da posição estimada pela câmera para restringir os pontos do LiDAR.

O parâmetro:

```python
POINT_CONE_DISTANCE
```

define essa região.

Uma futura arquitetura de fusão pode substituir ou complementar essa abordagem por uma associação espacial baseada em uma transformação extrínseca mais precisa e em uma projeção consistente entre os sensores.

---

# 5. Gerenciamento dos estados dos sensores

Outro ponto de desenvolvimento é tornar mais robusto o gerenciamento do estado dos sensores.

O sistema deve ser capaz de lidar com situações como:

```text
LiDAR funcionando + câmera funcionando
LiDAR funcionando + câmera indisponível
LiDAR indisponível + câmera funcionando
Nenhum sensor disponível
```

Uma arquitetura de gerenciamento de estado mais profissional pode utilizar os recursos do próprio ROS 2 para monitorar:

- disponibilidade dos nós;
- frequência das mensagens;
- atrasos;
- perda de comunicação;
- estado dos sensores.

O objetivo é evitar que uma falha ou atraso de um sensor comprometa desnecessariamente todo o sistema de percepção.

---

# 6. Migração para C++

Uma possível melhoria de longo prazo é a migração de partes do processamento para C++.

O pipeline atual possui implementação em Python utilizando principalmente NumPy e Scikit-learn.

Uma implementação em C++ pode ser investigada especialmente para operações intensivas sobre nuvens de pontos.

Bibliotecas especializadas em point clouds, como PCL, também podem ser avaliadas.

A migração não deve ser feita apenas por trocar a linguagem. Antes disso, deve-se identificar quais etapas realmente são gargalos de processamento e medir o desempenho atual.

---

# 7. Testes e avaliação quantitativa

Uma melhoria importante para as próximas gerações é estabelecer uma forma sistemática de avaliar o detector.

Além de observar os resultados visualmente no RViz2, podem ser registrados indicadores como:

- número de cones detectados;
- falsos positivos;
- falsos negativos;
- precisão da posição;
- desempenho em diferentes distâncias;
- tempo de processamento;
- frequência do pipeline.

Também é interessante manter conjuntos de dados representativos de diferentes situações da pista para que alterações no algoritmo possam ser comparadas utilizando as mesmas entradas.

---

# 8. Organização dos dados de teste

O projeto já possui mecanismos de publicação de dados artificiais, como os nós:

```text
artificial_lidar
framing_lidar
artificial_zed
```

Esses recursos podem ser utilizados como base para uma estrutura mais organizada de testes.

Uma evolução possível é manter conjuntos de dados representando casos específicos, por exemplo:

```text
data/
├── cones_proximos/
├── cones_distantes/
├── multiplos_cones/
├── casos_falsos_positivos/
└── casos_problematicos/
```

Cada conjunto deve possuir uma descrição indicando:

- origem dos dados;
- condições de aquisição;
- objetivo do teste;
- comportamento esperado.

---

# 9. Prioridade para futuras gerações

A ordem das melhorias deve ser determinada de acordo com os problemas encontrados durante os testes e com as necessidades da competição.

Uma forma possível de organizar as tarefas é:

### Curto prazo

- documentar e preservar os procedimentos de instalação;
- organizar os dados de teste;
- medir o desempenho atual;
- identificar falsos positivos e falsos negativos;
- melhorar a calibração dos parâmetros existentes.

### Médio prazo

- melhorar a calibração LiDAR–ZED;
- avaliar melhorias na fusão dos sensores;
- estudar a robustez da remoção do solo;
- estudar a clusterização em diferentes distâncias;
- desenvolver testes quantitativos.

### Longo prazo

- deskewing;
- gerenciamento mais robusto dos estados dos sensores;
- investigação de uma implementação em C++;
- avaliação de bibliotecas especializadas para point clouds;
- reformulação da arquitetura de fusão sensorial, caso necessário.

---

# 10. Como utilizar este documento

Este documento deve servir como ponto de partida para novos membros.

Antes de implementar uma das melhorias listadas, procure:

1. O código atual relacionado à funcionalidade.
2. Os parâmetros utilizados.
3. Os dados de teste existentes.
4. Os resultados obtidos pelas versões anteriores.
5. Discussões ou decisões registradas pela equipe.

Depois da implementação, registre:

```text
Problema identificado
        ↓
Hipótese
        ↓
Alteração realizada
        ↓
Teste
        ↓
Resultado
        ↓
Conclusão
```

Dessa forma, o conhecimento adquirido durante cada temporada permanece disponível para as próximas gerações.