#  Guia de Instalação e Configuração de Hardware

Este guia descreve o processo passo a passo para configurar os sensores e dispositivos de comunicação do subsistema de Percepção do zero em uma nova máquina (Ubuntu 22.04 LTS + ROS 2 Humble).

---

## 1. Câmera Stereolabs ZED 2i

### Pré-requisitos
* GPU NVIDIA com drivers proprietários instalados (`nvidia-smi` funcionando).
* CUDA Toolkit (versão recomendada pelo SDK da Stereolabs para Jetson/PC).

### Passo a Passo de Instalação
1. Baixar o SDK da ZED:
    ```bash
    wget [https://download.stereolabs.com/zedsdk/4.0/cu118/ubuntu22](https://download.stereolabs.com/zedsdk/4.0/cu118/ubuntu22) -O ZED_SDK.run
    chmod +x ZED_SDK.run
    ./ZED_SDK.run
2. Configurar Regras de Permissão USB (udev rules):
    ```Bash
    sudo usermod -aG dialout $USER
3. Instalar o ROS 2 Wrapper da ZED:
    ```Bash
    cd ~/ros2_ws/src
    git clone --recursive [https://github.com/stereolabs/zed-ros2-wrapper.git](https://github.com/stereolabs/zed-ros2-wrapper.git)
    cd ~/ros2_ws
    rosdep install --from-paths src --ignore-src -r -y
    colcon build --symlink-install --packages-up-to zed_wrapper
4. Validação:
    Conecte a ZED 2i em uma porta USB 3.0 (Azul/USB-C) e execute:
    ZED_Explorer (para testar vídeo/imagem) ou ZED_Diagnostic (para verificar sensores inerciais).

## 2. Sensor LiDAR
### Configuração da Interface de Rede (IP Estático)
Os sensores LiDAR transmitem pacotes UDP via Ethernet. A placa de rede do computador/Jetson deve estar na mesma sub-rede do sensor.
1. Identificar a interface Ethernet:
ip a ou nmcli device (exemplo: eth0 ou enp3s0).
2. Configurar IP Estático na Máquina:
IP da Máquina: 192.168.1.100 (exemplo)
Máscara: 255.255.255.0 (/24)
Gateway: Deixar em branco ou 192.168.1.1
3. Instalar Driver ROS 2 do LiDAR:
    ```Bash 
    cd ~/ros2_ws/src
    # Clonar o repositório do driver do LiDAR (exemplo para Velodyne/Hesai/Ouster)
    git clone [https://github.com/ros-drivers/velodyne.git](https://github.com/ros-drivers/velodyne.git) -b humble-devel
    cd ~/ros2_ws
    colcon build --packages-select velodyne
## 3. Antenas e Telemetria (Serial / USB)
1. Mapeamento de Portas Seriais:

    Para evitar que a porta do rádio/antena mude entre /dev/ttyUSB0 e /dev/ttyUSB1, configure uma regra de udev:
    ```Bash
    # Identificar o vendorID e productID do dispositivo
    lsusb
    # Criar o arquivo de regra personalizada
    echo 'SUBSYSTEM=="tty", ATTRS{idVendor}=="XXXX", ATTRS{idProduct}=="YYYY", SYMLINK+="telemetry_antenna"' | sudo tee /etc/udev/rules.d/99-telemetry.rules
    sudo udevadm control --reload-rules && sudo udevadm trigger
