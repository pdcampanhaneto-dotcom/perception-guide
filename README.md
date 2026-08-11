# Percepção Driverless — Unicamp E-Racing

Bem-vindo à documentação do subsistema de **Percepção** do SAUVA.

A percepção é responsável por obter informações dos sensores do veículo e produzir as detecções utilizadas pelos demais sistemas do Driverless, incluindo a detecção e classificação dos cones da pista.

---

## Documentação

| Documento | Conteúdo |
|---|---|
| [01 — Hardware Setup](docs/01_hardware_setup.md) | Instalação e configuração da ZED 2i, LiDAR e demais dispositivos utilizados pela percepção. |
| [02 — LiDAR Guide](docs/02_lidar_guide.md) | Configuração de rede, instalação do driver, execução do LiDAR, visualização no RViz2 e diagnóstico básico. |
| [03 — Code Explanation](docs/03_code_explanation.md) | Funcionamento do pipeline de processamento do LiDAR e sua integração com a ZED 2i. |
| [04 — Standards](docs/04_standards.md) | Organização do código, parâmetros, testes e práticas para manutenção do projeto. |
| [05 — Roadmap](docs/05_roadmap_future.md) | Limitações conhecidas e possíveis linhas de desenvolvimento futuro. |

---

## Repositórios e recursos

### Driver do LiDAR

O driver do LeiShen CH128X1 é disponibilizado pelo fabricante para ROS 2.

Uma cópia do driver utilizado pela equipe está disponível no Google Drive da equipe.

### Processamento do LiDAR

O processamento responsável pela extração dos cones está disponível em:

- **GitHub:** `lidar-perception-node`
- **GitLab da equipe:** repositório oficial do processamento de LiDAR

O GitHub contém uma explicação mais detalhada do funcionamento do pipeline.

---

## Visão geral

De maneira simplificada:

```text
              ┌──────────────┐
              │    LiDAR     │
              └──────┬───────┘
                     │
               PointCloud2
                     │
                     ▼
              ┌──────────────┐
              │   Pipeline   │
              │    LiDAR     │
              └──────┬───────┘
                     │
              Cones detectados
                     │
                     ▼
              ┌──────────────┐
              │    ZED 2i    │
              │     Fusão    │
              └──────┬───────┘
                     │
                     ▼
             Informação dos cones
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