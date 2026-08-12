# Percepção Driverless — Unicamp E-Racing

Bem-vindo à documentação do subsistema de **Percepção** do SAUVA.

A percepção é responsável por obter informações dos sensores do veículo e produzir as detecções utilizadas pelos demais sistemas do Driverless, incluindo a detecção e classificação dos cones da pista.

---

## Documentação

| Documento | Conteúdo |
|---|---|
| [01 — Hardware Setup](docs/01_hardware_setup.md) | Instalação e configuração da ZED 2i, LiDAR e demais dispositivos. |
| [02 — LiDAR Guide](docs/02_lidar_guide.md) | Configuração, execução e diagnóstico do LiDAR. |
| [03 — Code Explanation](docs/03_code_explanation.md) | Funcionamento do pipeline LiDAR, ZED e YOLO. |
| [04 — Standards](docs/04_standards.md) | Organização do código e práticas de manutenção. |
| [05 — Roadmap](docs/05_roadmap_future.md) | Limitações e possíveis desenvolvimentos futuros. |
| [06 — PTP](docs/06_sensor_synchronization_ptp.md) | Sincronização temporal entre LiDAR e Jetson. |

---

## Repositórios e recursos

### Driver do LiDAR

O driver do LeiShen CH128X1 é disponibilizado pelo fabricante para ROS 2.

Uma cópia do driver utilizado pela equipe está disponível no Google Drive da equipe.

### Processamento do LiDAR

O processamento responsável pela extração dos cones está disponível em:

- **GitHub:** [lidar-perception-node](https://github.com/eduardokobata/lidar-perception-node/tree/main)
- **GitLab da equipe:** [repositório oficial do processamento de LiDAR](https://gitlab.com/unicamperacing/autonomous-systems/driverless/sauva/perception/repositories/lidar)

O GitHub contém o código e uma documentação técnica desenvolvida
durante o projeto. Esta documentação do repositório da equipe complementa
essas informações com procedimentos de instalação, operação, arquitetura,
desenvolvimento e histórico do sistema.

### ZED 2i e YOLO

A documentação da câmera ZED 2i e do modelo YOLO utilizado na percepção
está sendo consolidada a partir do código e dos materiais mantidos no GitLab.

Os detalhes do treinamento e da implantação do modelo são apresentados
no `03_code_explanation.md`.

---

## Visão geral

De maneira simplificada:

```text
               ┌─────────────┐
               │    LiDAR    │
               └──────┬──────┘
                      │
                 PointCloud2
                      │
                      ▼
               ┌─────────────┐
               │   Pipeline  │◄──────────────┐
               │    LiDAR    │               │
               └──────┬──────┘               │
                      │                      │
               cones detectados              │
                      │                      │
                      ▼                      │
               ┌─────────────┐        coordenadas
               │    Fusão    │◄──────────────┘
               └──────┬──────┘
                      │
                      ▼
               cones + cor
                      ▲
                      │
                 posição + cor
                      │
               ┌──────┴──────┐
               │    ZED 2i   │
               │   + YOLOv8  │
               └─────────────┘
```

Para configurar uma máquina nova, comece pelo **01 — Hardware Setup**.

Para colocar o LiDAR em funcionamento, consulte o **02 — LiDAR Guide**.

Para entender o funcionamento do algoritmo, consulte o **03 — Code Explanation**.

---

## Desenvolvimento

Antes de modificar o pipeline, recomenda-se ler o documento de explicação do código e verificar os procedimentos de teste.

Alterações relevantes no comportamento do sistema devem ser registradas na documentação para que possam ser compreendidas pelas próximas gerações da equipe.

---

## Micro-divisão

**Unicamp E-Racing — Driverless**  
**Perception**