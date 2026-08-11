# Guia de Operação do LiDAR

Este documento explica como configurar, executar e verificar o LiDAR LeiShen CH128X1 utilizado pelo sistema de Percepção do Driverless.

Para uma instalação completa em uma máquina nova, consulte também o [Guia de Instalação e Configuração de Hardware](01_hardware_setup.md).

---

## 1. Visão geral

O LiDAR LeiShen CH128X1 fornece uma nuvem de pontos tridimensional do ambiente ao redor do veículo.

No ROS 2, o driver do sensor funciona como um **nó** que recebe os dados do LiDAR e publica a nuvem de pontos em um tópico do tipo:

```text
sensor_msgs/msg/PointCloud2
```

O nó de processamento da percepção utiliza esse tópico como entrada para realizar a detecção dos cones.

O fluxo básico é:

```text
LiDAR
  │
  ▼
Driver LSLiDAR
  │
  │ PointCloud2
  ▼
Nó de processamento da percepção
  │
  ▼
Detecção dos cones
```

---

## 2. Pré-requisitos

Antes de executar o LiDAR, certifique-se de que:

- o driver do LiDAR está instalado;
- as dependências indicadas no `README_en.md` do driver foram instaladas;
- os pacotes foram compilados com `colcon`;
- o computador está conectado ao LiDAR por Ethernet;
- a interface Ethernet está configurada para a rede do sensor.

Para verificar a comunicação com o sensor:

```bash
ping 192.168.1.102
```

O recebimento de respostas indica que o computador consegue alcançar o LiDAR pela rede.

---

## 3. Executando o driver

Entre na pasta do driver:

```bash
cd ~/LIDAR/lslidar/LSLIDAR_ROS2_V5.0.9_250305/lslidar_driver/src
```

Carregue o ambiente ROS 2:

```bash
source install/setup.bash
```

Execute o launch do LiDAR:

```bash
ros2 launch lslidar_driver lslidar_ch_launch.py
```

O driver iniciará o nó do LiDAR e o `RViz2` utilizado para visualizar a nuvem de pontos.

---

## 4. Verificando a nuvem de pontos no RViz2

Ao executar o launch, o `RViz2` deve apresentar a nuvem de pontos produzida pelo sensor.

Essa é a primeira verificação visual de que o LiDAR está funcionando corretamente.

Observe:

- se existem pontos na visualização;
- se a nuvem corresponde ao ambiente ao redor do sensor;
- se a visualização está sendo atualizada;
- se não existem erros relacionados ao driver ou à comunicação.

---

## 5. Verificando os tópicos ROS 2

O driver publica os dados do LiDAR através de um tópico ROS 2.

Para visualizar os tópicos ativos:

```bash
ros2 topic list
```

Para verificar o tipo de uma publicação:

```bash
ros2 topic type <nome_do_topico>
```

O tópico utilizado pelo processamento deve ser do tipo:

```text
sensor_msgs/msg/PointCloud2
```

O nome exato do tópico utilizado pela equipe é configurado no pacote de processamento através da constante:

```python
LIDAR_TOPIC_TO_BE_SUBSCRIBED
```

em `constants.py`.

---

## 6. Executando o processamento dos cones

Depois de verificar que o driver está publicando a nuvem de pontos, o nó de processamento pode ser executado.

O código do processamento está disponível em:

[Nó de processamento LiDAR — GitHub](https://github.com/eduardokobata/lidar-perception-node?utm_source=chatgpt.com)

O mesmo código também está disponível no repositório oficial da equipe:

[Nó de processamento LiDAR — GitLab SAUVA](https://gitlab.com/unicamperacing/autonomous-systems/driverless/sauva/perception/repositories/lidar/processing/?utm_source=chatgpt.com)

Entre no workspace do processamento e carregue o ambiente:

```bash
source install/setup.bash
```

Execute:

```bash
ros2 run detector_pkg lidar_node
```

O nó `lidar_node` recebe a nuvem `PointCloud2`, executa o pipeline de percepção e produz as detecções de cones.

---

## 7. Dados artificiais para testes

O pacote de processamento também possui nós destinados a testes com dados artificiais.

### `artificial_lidar`

Publica uma nuvem de pontos previamente salva em arquivo `.npy` no tópico:

```text
artificialLIDAR
```

Esse nó permite testar o processamento sem depender da nuvem de pontos produzida pelo sensor físico.

### `framing_lidar`

Publica sequencialmente nuvens de pontos armazenadas em uma pasta no tópico:

```text
framingLIDAR
```

Esse modo permite reproduzir diferentes nuvens de pontos previamente registradas.

### `artificial_zed`

Publica um conjunto de detecções de cones simuladas no tópico:

```text
artificial_zed
```

Esse nó é utilizado para testar a parte da percepção que utiliza informações provenientes da câmera.

Os tópicos utilizados pelo `lidar_node` podem ser alterados no arquivo:

```text
constants.py
```

principalmente através de:

```python
LIDAR_TOPIC_TO_BE_SUBSCRIBED
CAMERA_TOPIC_TO_BE_SUBSCRIBED
```

---

## 8. Problemas comuns

### O `ping` para `192.168.1.102` não responde

Verifique:

1. se o cabo Ethernet está conectado;
2. se a interface Ethernet está ativa;
3. se o IPv4 foi configurado para a rede do LiDAR;
4. se o endereço do sensor está correto.

---

### O driver não inicia

Consulte o `README_en.md` presente no pacote do driver e verifique se todas as dependências foram instaladas.

Se o pacote foi transferido de outra máquina, tente remover os diretórios de compilação:

```bash
rm -rf build install log
```

e compile novamente:

```bash
colcon build
```

Depois:

```bash
source install/setup.bash
```

---

### O pacote funciona em um computador, mas não em outro

Os arquivos dentro de `build`, `install` e `log` podem conter artefatos gerados especificamente para a máquina em que foram compilados.

Isso é particularmente importante em **Jetsons**, cuja arquitetura é diferente da de computadores convencionais.

Ao transferir o código para outra máquina, recomenda-se recompilar os pacotes na própria máquina.

---

## 9. Checklist rápido

Antes de considerar o LiDAR operacional:

```text
[ ] Driver instalado
[ ] Dependências instaladas
[ ] Interface Ethernet configurada
[ ] Cabo Ethernet conectado
[ ] ping 192.168.1.102 funcionando
[ ] Driver iniciado
[ ] PointCloud2 sendo publicado
[ ] Nuvem visível no RViz2
[ ] detector_pkg compilado
[ ] lidar_node iniciado
[ ] Detecções de cones sendo produzidas
```