# Padrões e Organização — Percepção Driverless

Este documento reúne as convenções utilizadas no desenvolvimento e manutenção do subsistema de **Percepção** do SAUVA.

O objetivo não é impor um padrão genérico de desenvolvimento de software, mas registrar as práticas que devem ser preservadas para que os próximos membros consigam entender, testar e modificar o código existente.

---

## 1. Antes de modificar o código

Antes de alterar qualquer parte do pipeline, o membro deve:

1. Entender qual é a função do arquivo que será modificado.
2. Verificar quais outros módulos dependem dele.
3. Identificar quais tópicos ROS 2 entram e saem daquela etapa.
4. Verificar as constantes utilizadas no processamento.
5. Executar o pipeline antes da alteração, quando possível.
6. Testar novamente após a alteração e comparar o comportamento.

A percepção é composta por várias etapas encadeadas. Uma alteração aparentemente pequena em uma etapa pode modificar a quantidade ou a distribuição dos pontos recebidos pelas etapas seguintes.

---

## 2. Organização do código

O processamento do LiDAR é dividido em módulos responsáveis por diferentes partes do pipeline.

De maneira geral:

```text
detector_pkg/
├── main.py
├── constants.py
├── clustering.py
├── geometric_filters.py
├── mlesac.py
├── roi.py
├── transform.py
├── fusio_engine.py
├── artificial_lidar.py
├── artificial_framing_lidar.py
├── artificial_zed.py
└── auxiliar_tools.py
```

A divisão dos arquivos deve ser preservada sempre que possível.

Por exemplo:

- `roi.py` → filtragem da região de interesse;
- `mlesac.py` → remoção do plano do solo;
- `clustering.py` → formação dos clusters e cálculo dos centroides;
- `geometric_filters.py` → avaliação geométrica dos clusters;
- `transform.py` → transformação dos dados da ZED para o referencial utilizado pelo LiDAR;
- `fusio_engine.py` → operações relacionadas à fusão LiDAR–câmera;
- `constants.py` → parâmetros utilizados pelo pipeline;
- `main.py` → nó ROS 2 principal.

Ao criar uma nova etapa de processamento, prefira criar um módulo separado em vez de concentrar toda a lógica em `main.py`.

---

## 3. Parâmetros e constantes

Os principais parâmetros do processamento são mantidos em `constants.py`.

Exemplos:

```python
XRMIN = -1.5
XRMAX = 1.5

YRMIN = 1.2
YRMAX = 7

POINT_CONE_DISTANCE = 0.5

RADIUS_THRESHOLD = 0.16
RADIUS_RESTORATION = 0.18
```

Esses valores fazem parte do comportamento do detector.

Portanto, ao alterá-los, deve-se registrar:

- qual parâmetro foi alterado;
- qual era o valor anterior;
- qual passou a ser o novo valor;
- qual problema motivou a alteração;
- em quais condições a alteração foi testada.

Evite colocar valores de calibração diretamente no meio das funções quando eles puderem ser mantidos como constantes.

---

## 4. Tópicos ROS 2

Os nomes dos tópicos utilizados pelo pipeline são definidos em `constants.py`.

Atualmente, o processamento utiliza:

```python
LIDAR_TOPIC_TO_BE_SUBSCRIBED = 'framingLIDAR'
CAMERA_TOPIC_TO_BE_SUBSCRIBED = 'artificial_zed'
```

Durante a utilização com o LiDAR real, o tópico deve corresponder ao tópico publicado pelo driver do sensor.

O nome do tópico não deve ser alterado em um único arquivo sem verificar os demais nós que publicam ou consomem essa informação.

Para descobrir os tópicos disponíveis no ambiente:

```bash
ros2 topic list
```

Para verificar o tipo de uma mensagem:

```bash
ros2 topic type <nome_do_topico>
```

Para observar as mensagens:

```bash
ros2 topic echo <nome_do_topico>
```

---

## 5. Dados utilizados pelo pipeline

O LiDAR publica uma nuvem de pontos utilizando a mensagem ROS 2:

```text
sensor_msgs/msg/PointCloud2
```

O processamento converte essa informação para uma representação NumPy contendo pontos no formato:

```text
[x, y, z]
```

A câmera, por sua vez, fornece informações de cones contendo:

```text
[x, y, color]
```

A etapa de fusão utiliza a posição dos cones detectados pela câmera para auxiliar o processamento do LiDAR e, posteriormente, associar a cor aos cones detectados.

---

## 6. Estrutura das funções

As funções de processamento devem deixar claro:

- qual é a entrada;
- qual é a saída;
- em qual referencial os dados estão;
- qual transformação ou filtragem é realizada.

Por exemplo, funções que trabalham com nuvens de pontos devem indicar que recebem uma estrutura contendo pontos:

```text
[[x1, y1, z1],
 [x2, y2, z2],
 ...
 [xn, yn, zn]]
```

e informar se retornam:

- outra nuvem de pontos;
- clusters;
- centroides;
- classificações booleanas;
- ou os cones finais.

Isso é especialmente importante porque várias etapas do pipeline utilizam estruturas semelhantes.

---

## 7. Alterações no pipeline

Ao modificar uma etapa do pipeline, deve-se verificar pelo menos:

```text
Entrada
   ↓
Etapa modificada
   ↓
Saída da etapa
   ↓
Etapa seguinte
```

Sempre que possível, utilize RViz2 ou outras ferramentas de visualização para verificar o resultado intermediário.

Uma alteração no ROI, por exemplo, pode:

- remover pontos que seriam utilizados posteriormente;
- alterar o resultado da remoção do solo;
- modificar os clusters encontrados;
- alterar as características geométricas;
- alterar a quantidade final de cones.

Por isso, não é suficiente verificar apenas se o programa continua executando sem erros.

---

## 8. Testes com dados artificiais

O pacote possui nós auxiliares para testar o processamento sem necessariamente utilizar os sensores físicos.

Entre eles estão:

- `artificial_lidar`;
- `framing_lidar`;
- `artificial_zed`.

Esses nós permitem publicar dados previamente armazenados ou simulados nos mesmos tópicos utilizados pelo pipeline.

Isso é útil para:

- testar alterações no algoritmo;
- reproduzir casos específicos;
- depurar uma etapa isoladamente;
- trabalhar quando o sensor físico não está disponível.

Ao criar novos casos de teste, mantenha os dados utilizados e registre sua origem e finalidade.

---

## 9. Compilação

Após alterações no pacote ROS 2, o workspace deve ser recompilado.

Exemplo:

```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

Quando houver problemas relacionados a arquivos compilados anteriormente, especialmente ao transportar o workspace entre máquinas ou arquiteturas diferentes, pode ser necessário remover os diretórios gerados anteriormente:

```bash
rm -rf build install log
colcon build --symlink-install
```

Isso é particularmente relevante ao trabalhar com a Jetson, cuja arquitetura é diferente da de computadores convencionais.

---

## 10. Documentação de alterações

Alterações importantes no comportamento da percepção devem ser documentadas.

Uma descrição útil deve responder:

```text
O que foi alterado?
Por que foi alterado?
Qual problema existia?
Como a alteração foi testada?
Qual foi o resultado?
```

Isso permite que os próximos membros entendam não apenas o estado atual do código, mas também as decisões tomadas anteriormente.

---

## 11. Princípio geral

A prioridade da documentação é permitir que um novo membro consiga responder às seguintes perguntas sem depender exclusivamente de outro membro da equipe:

1. Como faço o sensor funcionar?
2. Como verifico se o sensor está funcionando?
3. Que dados o sensor publica?
4. Como esses dados entram no pipeline?
5. O que cada etapa do pipeline faz?
6. Quais parâmetros controlam o comportamento?
7. Como testo uma alteração?
8. Onde encontro os próximos pontos que precisam ser melhorados?

Caso uma alteração torne alguma dessas respostas obsoleta, a documentação correspondente também deve ser atualizada.