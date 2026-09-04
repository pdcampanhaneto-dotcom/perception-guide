# Guia de Instalação e Configuração de Hardware

Este documento apresenta os procedimentos necessários para preparar uma máquina para trabalhar com os sensores utilizados pela microdivisão de Percepção do Driverless.

---

## 1. LiDAR LeiShen CH128X1

O LiDAR utilizado pela equipe é o **LeiShen CH128X1**. A comunicação com o computador é realizada por Ethernet, utilizando uma conexão de rede local entre o computador e o sensor.

### 1.1 Driver

O driver ROS 2 do LiDAR pode ser obtido diretamente no [repositório do fabricante](https://github.com/Lslidar/Lslidar_ROS2_driver/tree/LS-S1_V1.0). A equipe também mantém uma cópia dos arquivos do driver no Google Drive:

[Driver LSLiDAR — Google Drive da equipe](https://drive.google.com/drive/folders/1seugpC1GXATPf5KhsMkJyf09zj-AvZNO?usp=sharing&utm_source=chatgpt.com)

O pacote utilizado pela equipe contém o driver específico para o LiDAR CH128X1.

> **Importante:** o driver possui um arquivo `README_en.md` com as instruções de instalação e das dependências. Essas instruções devem ser consultadas durante a configuração de uma máquina nova.

### 1.2 Configuração da interface Ethernet

O LiDAR transmite os dados pela interface Ethernet. Para que o computador consiga se comunicar com o sensor, sua interface de rede deve estar configurada na mesma sub-rede do LiDAR.

No Ubuntu:

1. Abra **Configurações → Rede → Wired**.
2. Crie ou edite um perfil de conexão Ethernet.
3. Configure manualmente o IPv4 para a rede utilizada pelo LiDAR.
4. Conecte o LiDAR ao computador através do cabo Ethernet.

No ambiente atualmente utilizado pela equipe, o LiDAR é acessado pelo endereço:

```text
192.168.1.102
```

A configuração exata do endereço IP do computador deve ser mantida de acordo com a configuração utilizada pela equipe.

### 1.3 Teste de comunicação

Depois de conectar o sensor e configurar a interface Ethernet, verifique a comunicação executando:

```bash
ping 192.168.1.102
```

Se o computador receber respostas do endereço `192.168.1.102`, existe comunicação de rede com o LiDAR.

Caso o `ping` não funcione, o problema deve ser investigado na configuração da interface Ethernet, no cabo ou na conexão com o sensor antes de prosseguir para o ROS 2.

---

## 2. Compilação dos pacotes ROS 2

Os drivers do LiDAR são pacotes ROS 2 e precisam ser compilados no ambiente em que serão executados.

Após instalar as dependências indicadas pelo `README_en.md` de cada pacote, compile os pacotes utilizando `colcon build`.

Durante a configuração de uma máquina nova, pode ser necessário remover os diretórios de compilação anteriores:

```bash
rm -rf build install log
```

e então executar:

```bash
colcon build
```

Depois da compilação:

```bash
source install/setup.bash
```

### 2.1 Por que limpar `build`, `install` e `log`?

A equipe encontrou esse procedimento como uma etapa importante principalmente ao transferir o ambiente entre máquinas diferentes.

Isso é especialmente relevante em computadores **Jetson**, cuja arquitetura é diferente da de um computador convencional. Arquivos previamente compilados em outra máquina podem não ser compatíveis com a arquitetura de destino.

Por isso, ao transferir os pacotes para outra máquina, especialmente uma Jetson, a recomendação é remover os arquivos de compilação existentes e recompilar os pacotes na própria máquina.

---

## 3. Câmera ZED 2i

A Percepção utiliza uma câmera **Stereolabs ZED 2i** para a detecção visual dos cones.

O código utilizado pela equipe está mantido no repositório da percepção no GitLab.

A documentação de instalação da ZED deve distinguir entre:

- **ambiente da câmera:** ZED SDK e suas dependências;
- **modelo de detecção:** YOLO e seus pesos;
- **integração com o código da equipe:** `cameraProcessing.py`, `main.py` e `dataProcessing.py`.

### 3.1 Estado atual da instalação

O código da câmera utilizado pela equipe já foi identificado e testado, porém o procedimento completo para reproduzir o ambiente de execução a partir de uma máquina limpa ainda não está formalizado nesta documentação.

Por isso, não devem ser considerados oficiais comandos de instalação obtidos apenas de tutoriais externos sem verificar sua compatibilidade com o ambiente utilizado pela equipe.

### 3.2 Componentes utilizados

O funcionamento da câmera depende dos seguintes componentes:

```text
ZED 2i
   │
   ▼
ZED SDK
   │
   ▼
Python API (`pyzed.sl`)
   │
   ▼
YOLO
   │
   ▼
Código da câmera
   │
   ▼
ROS 2
```

A configuração específica desses componentes deve ser registrada assim que o procedimento utilizado pela equipe puder ser reproduzido integralmente.

### 3.3 Verificação

Depois de configurado o ambiente, a câmera deve ser testada antes da integração com o restante da percepção.

A validação deve verificar:

- se a ZED é reconhecida;
- se imagens podem ser capturadas;
- se o modelo de detecção é carregado;
- se os cones são detectados;
- se o resultado possui o formato esperado pelo restante do sistema.

### 3.4 Integração com ROS 2

O nó responsável pela publicação das detecções da câmera publica:

```text
camera_cones
```

com o tipo:

```text
std_msgs/msg/Float32MultiArray
```

As detecções são representadas como grupos:

```text
[x, y, color]
```

O funcionamento detalhado da câmera, do YOLO e da publicação ROS 2 está documentado em `03_code_explanation.md`.

> **Nota para futuros membros:** este documento deve ser atualizado sempre que o procedimento oficial de instalação de um sensor mudar. Evite copiar instruções de versões antigas do sistema sem verificar a configuração atualmente utilizada pela equipe.