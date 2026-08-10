# Percepção Driverless — Unicamp E-Racing (SAUVA)

Bem-vindo ao repositório do subsistema de **Percepção** da plataforma autônoma **SAUVA**. Este repositório contém os pacotes ROS 2 responsáveis pela detecção, classificação e estimativa 3D dos cones delimitadores da pista.

---

## Documentação Rápida & Links Úteis

Para acessar os guias detalhados de instalação, uso e arquitetura, clique nos links abaixo:

* [**Guia de Instalação de Hardware (ZED, LiDAR, Telemetria)**](docs/01_hardware_setup.md) — *Leia este guia se estiver configurando uma nova máquina do zero.*
* [**Manual de Operação e Debug do LiDAR**](docs/02_lidar_guide.md) — *Configuração de IP, drivers, RViz e solução de problemas.*
* [**Explicação do Código e Processamento de Cones**](docs/03_code_explanation.md) — *Entenda o que recebemos dos sensores e a matemática da detecção.*
* [**Padrões de Código, Git e ROS 2**](docs/04_standards.md) — *Regras de contribuição e guia de estilo.*
* [**Roadmap e Passos Futuros**](docs/05_roadmap_future.md) — *Próximas otimizações e metas de pesquisa.*

---

## Repositórios e Recursos Oficiais de Percepção

| Recurso | Descrição | Link / Acesso |
| :--- | :--- | :--- |
| 📁 **Driver LSLiDAR (V5.0.9)** | Driver pré-configurado do fabricante para ROS 2 | [Google Drive](https://drive.google.com/drive/folders/1seugpC1GXATPf5KhsMkJyf09zj-AvZNO?usp=sharing) |
| 🐙 **Nó de Processamento LiDAR (GitHub)** | Código do Kobata para extração de cones com README explicativo | [GitHub eduardokobata](https://github.com/eduardokobata/lidar-perception-node) |
| 🦊 **Nó de Processamento LiDAR (GitLab)** | Branch oficial no repositório do SAUVA | [GitLab SAUVA](https://gitlab.com/unicamperacing/autonomous-systems/driverless/sauva/perception/repositories/lidar/processing/) |
| 📖 **Guia do LSLiDAR** | Manual de IP, comandos de launch e limpeza de build | [docs/02_lidar_guide.md](docs/02_lidar_guide.md) |

---

## Quickstart (Para quem já tem o ambiente configurado)

```bash
# 1. Clonar e compilar o workspace
cd ~/ros2_ws
colcon build --packages-up-to perception_bringup

# 2. Carregar o ambiente
source install/setup.bash

# 3. Iniciar o pipeline completo com visualização no RViz2
ros2 launch perception_bringup perception.launch.py use_rviz:=true
```

## Contato e Suporte
    Diretor de percepção: Kobata
    Membro: Campanha

    Micro-divisão de Percepção: Unicamp E-Racing Driverless