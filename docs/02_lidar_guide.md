#  Guia de Operação e Instalação do LSLiDAR (Leishen)

Este documento descreve o processo completo de configuração de rede, compilação de drivers, execução do sensor **LSLiDAR (Série CH)** e o nó de processamento para extração de cones.

---

## 1. Download dos Drivers e Dependências

Os drivers oficiais do fabricante já estão pré-configurados e salvos no Google Drive da equipe:
*  **Driver do LSLiDAR:** [Google Drive — LSLiDAR ROS2 Driver](https://drive.google.com/drive/folders/1seugpC1GXATPf5KhsMkJyf09zj-AvZNO?usp=sharing)

### Passos de Instalação de Bibliotecas:
1. Baixe a pasta `LSLIDAR_ROS2_V5.0.9_250305` do Drive e coloque no seu diretório local (exemplo: `~/LIDAR/lslidar/`).
2. Abra o arquivo `README_en.md` localizado dentro da pasta do driver para verificar se o seu sistema possui todas as bibliotecas C++ e ROS 2 necessárias instaladas.

---

## 2. Configuração de Rede (IP Estático no Ubuntu)

O LSLiDAR se comunica com o computador via rede Ethernet (pacotes UDP). O IP de fábrica do sensor é **`192.168.1.102`**.

1. Conecte o cabo Ethernet do LiDAR ao computador ou à Jetson.
2. Abra as **Configurações do Ubuntu** ➔ vá na aba **Wired (Com Fio)** ➔ clique na engrenagem da conexão.
3. Vá na aba **IPv4** e altere de *Automatic (DHCP)* para **Manual**.
4. Configure as seguintes informações:
   * **Address (IP do PC):** `192.168.1.100` *(qualquer IP na faixa 192.168.1.X diferente do sensor)*
   * **Netmask (Mascara):** `255.255.255.0`
   * **Gateway:** `192.168.1.1` (ou em branco)
5. Clique em **Apply** e reconecte a rede.
6. **Teste de Conectividade:**
   No terminal, execute o ping para o sensor:
   ```bash
   ping 192.168.1.102
Se houver resposta dos pacotes, o hardware está conectado e pronto para transmissão.
## 3. Compilação e Inicialização do Driver
    ATENÇÃO (PC x86 vs. NVIDIA Jetson ARM64):
    Códigos pré-compilados em um notebook NÃO funcionam na Jetson devido à diferença de arquitetura de processador.
    Sempre apague as pastas build/, install/ e log/ antes de recompilar o workspace ao trocar de máquina!
Procedimento de Compilação e Launch:
```bash 
    # 1. Navegar até a pasta do driver do LSLiDAR
    cd ~/LIDAR/lslidar/LSLIDAR_ROS2_V5.0.9_250305/lslidar_driver

    # 2. Limpar artefatos de compilação antigos (Obrigatório)
    rm -rf build install log

    # 3. Compilar via colcon build
    colcon build

    # 4. Carregar o ambiente compilado
    source install/setup.bash

    # 5. Executar o nó do driver (Abre o RViz2 automaticamente com a nuvem de pontos)
    ros2 launch lslidar_driver lslidar_ch_launch.py
``` 
Após o lançamento, o nó do ROS 2 começará a publicar a nuvem de pontos tridimensional bruta no formato sensor_msgs/msg/PointCloud2.
## 4. Nó de Processamento de Cones (Lidar Perception Node)
Para converter a nuvem de pontos bruta em detecção e extração tridimensional dos cones da pista, utilizamos o nó de processamento da equipe:

    Repositório GitHub (Com README detalhado): eduardokobata/lidar-perception-node

    Repositório GitLab SAUVA: GitLab Perception LiDAR Processing
Como Rodar o Processamento de Cones:
1. Clone o repositório dentro do seu workspace do ROS 2:
```bash
cd ~/ros2_ws/src
git clone [https://github.com/eduardokobata/lidar-perception-node.git](https://github.com/eduardokobata/lidar-perception-node.git)
```
2. Limpe builds antigos e compile:
```bash
cd ~/ros2_ws
rm -rf build install log
colcon build --packages-select lidar_perception_node
source install/setup.bash
```
3. Execute o nó enquanto o driver do LSLiDAR estiver rodando no outro terminal:
```bash
ros2 run lidar_perception_node lidar_perception_node
```