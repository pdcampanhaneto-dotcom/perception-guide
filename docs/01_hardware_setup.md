# Guia de Instalação e Configuração de Hardware

Este documento apresenta os procedimentos necessários para preparar uma máquina para trabalhar com os sensores utilizados pela microdivisão de Percepção do Driverless.

---

## 1. LiDAR LeiShen CH128X1

O LiDAR utilizado pela equipe é o **LeiShen CH128X1**. A comunicação com o computador é realizada por Ethernet, utilizando uma conexão de rede local entre o computador e o sensor.

### 1.1 Driver

O driver ROS 2 do LiDAR pode ser obtido diretamente no repositório do fabricante. A equipe também mantém uma cópia dos arquivos do driver no Google Drive:

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

O endereço utilizado pelo LiDAR é:

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

## 3. Outros sensores

A configuração da câmera ZED 2i e dos demais equipamentos da percepção deve ser documentada conforme os procedimentos efetivamente utilizados pela equipe.

> **Nota para futuros membros:** este documento deve ser atualizado sempre que o procedimento oficial de instalação de um sensor mudar. Evite copiar instruções de versões antigas do sistema sem verificar a configuração atualmente utilizada pela equipe.