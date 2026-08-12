# Sincronização Temporal entre LiDAR e NVIDIA Jetson utilizando PTP

Este documento descreve o procedimento utilizado pela equipe para sincronizar temporalmente o LiDAR LeiShen CH128X1 com a NVIDIA Jetson utilizando PTP.

A sincronização temporal é importante para a percepção porque LiDAR e câmera observam o ambiente em instantes diferentes. Em um veículo em movimento, uma diferença de tempo entre as medições pode resultar em posições aparentes diferentes para o mesmo objeto.

---

## 1. Conceito

O Precision Time Protocol (PTP) é utilizado para sincronizar relógios através de uma rede.

No procedimento documentado pela equipe, o objetivo é utilizar o relógio do LiDAR como referência e sincronizar o relógio da NVIDIA Jetson a ele.

O CH128X1 possui suporte a sincronização por gPTP. O manual do fabricante descreve a seleção da fonte de relógio como `PTP` e informa que, nesse modo, os timestamps passam a utilizar nanossegundos e são sincronizados com o sinal fornecido pelo master PTP.

> **Observação de nomenclatura:** os documentos da equipe utilizam o termo "PTP L2". O manual do fabricante do CH128X1 v1.0.6 descreve essa funcionalidade como **gPTP**.

---

## 2. Arquitetura utilizada

```text
       LSLiDAR CH128X1
              │
              │ PTP
              ▼
       Interface Ethernet
              │
              ▼
       NVIDIA Jetson
              │
              ▼
      CLOCK_REALTIME
```

O objetivo é que a referência temporal utilizada pelo computador esteja alinhada com a referência temporal do LiDAR.

---

## 3. Dependência

O procedimento utiliza o pacote `linuxptp`.

A documentação original da equipe indica a utilização de uma versão **4.4 ou superior**.

---

## 4. Configuração

O arquivo utilizado pela equipe é:

```text
ptp4l_slave.cfg
```

Conteúdo:

```ini
[global]

clientOnly 1
priority1 255
priority2 255
domainNumber 0
twoStepFlag 1
network_transport L2
delay_mechanism P2P
ignore_transport_specific 1
inhibit_announce 1
ignore_source_id 1
BMCA noop

[eth0]
```

A interface `eth0` deve ser substituída caso a interface Ethernet utilizada pela máquina possua outro nome.

---

## 5. Executar o driver do LiDAR

Primeiro, inicie o driver normalmente:

```bash
ros2 launch lslidar_driver lslidar_ch_launch.py
```

Consulte o [Guia de Operação do LiDAR](02_lidar_guide.md) caso seja necessário configurar ou iniciar o sensor.

---

## 6. Verificar o serviço de configuração temporal

No workspace que contém os arquivos do LiDAR, pode-se localizar o serviço `TimeMode.srv`:

```bash
cat $(find . -name "TimeMode.srv")
```

A implementação utilizada pela equipe define os modos temporais do serviço.

No procedimento documentado pelo Kobata:

```text
0 → GPS
1 → PTP L2
2 → NTP
```

Assim, o valor `1` é utilizado para solicitar o modo PTP.

---

## 7. Configurar o LiDAR para PTP

Com o driver em execução:

```bash
ros2 service call /ch/time_mode \
lslidar_msgs/srv/TimeMode \
"{time_mode: 1, ntp_ip: ''}"
```

A execução bem-sucedida deve produzir uma mensagem semelhante a:

```text
PTP L2 node setting successful
```

> **Importante:** esse comando faz parte do procedimento utilizado pela equipe. A disponibilidade exata do serviço depende da versão do driver instalada.

O manual do fabricante confirma que o CH128X1 possui um modo de seleção de clock no qual `PTP` corresponde ao valor `1`.

---

## 8. Iniciar o `ptp4l`

Em outro terminal, execute o `ptp4l` utilizando o arquivo de configuração:

```bash
sudo <caminho-do-linuxptp>/ptp4l \
    -f /etc/linuxptp/ptp4l_slave.cfg \
    -i eth0 \
    -m
```

Exemplo:

```bash
sudo ~/Downloads/linuxptp/ptp4l \
    -f /etc/linuxptp/ptp4l_slave.cfg \
    -i eth0 \
    -m
```

O processo deve ser acompanhado pelo terminal para verificar o estado da sincronização.

---

## 9. Sincronizar o relógio do sistema

Depois de iniciar o `ptp4l`, execute:

```bash
sudo phc2sys -s eth0 -c CLOCK_REALTIME -w -m
```

Esse processo realiza a sincronização entre o relógio da interface de rede e o relógio do sistema operacional.

---

## 10. Relação com a fusão sensorial

A sincronização temporal é complementar à calibração espacial.

Para associar corretamente as informações:

```text
           ZED
            │
       timestamp
            │
            ▼
        percepção
            ▲
            │
       timestamp
            │
          LiDAR
```

é necessário que as referências temporais dos sensores sejam suficientemente próximas.

Mesmo com uma transformação espacial correta entre LiDAR e câmera, uma diferença temporal significativa pode produzir uma associação espacial incorreta quando o veículo está em movimento.

---

## 11. Relação com o deskewing

O CH128X1 fornece timestamps que permitem considerar o instante de aquisição das medições. O manual do fabricante descreve inclusive a forma de calcular o tempo preciso de cada ponto a partir dos timestamps dos pacotes.

Isso é relevante para futuras melhorias relacionadas a **deskewing**, nas quais a distorção causada pelo movimento do veículo durante a aquisição de uma varredura pode ser compensada.

---

## 12. Diagnóstico

Caso a sincronização não funcione:

1. verifique se o driver do LiDAR está executando;
2. confirme se a interface Ethernet utilizada no `ptp4l` está correta;
3. confirme se o modo temporal do LiDAR foi configurado;
4. verifique as mensagens apresentadas pelo `ptp4l`;
5. verifique as mensagens apresentadas pelo `phc2sys`;
6. confirme que `linuxptp` está instalado na versão esperada.

O procedimento deve ser executado somente após o funcionamento básico do LiDAR estar confirmado.