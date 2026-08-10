#  Padrões de Código e Desenvolvimento (Percepção Driverless)

Para garantir a qualidade, facilidade de leitura e manutenção do repositório, todos os membros devem seguir estes padrões.

## 1. Nomenclatura no ROS 2
* **Nós (Nodes):** `snake_case` com sufixo `_node` (ex: `cone_detection_node`, `lidar_processing_node`).
* **Tópicos:** `snake_case` com namespace da micro-divisão (ex: `/perception/camera/cones`, `/sauva/cones_3d`).
* **Parâmetros:** `snake_case` descritivo (ex: `confidence_threshold`, `ransac_distance_threshold`).
* **Arquivos Launch:** `nome_do_pacote.launch.py` ou `funcionalidade.launch.py`.

## 2. Estrutura de Código (Python & C++)
* **Python:** Seguir **PEP 8**. Formatação automática obrigatória com `black` e checagem com `flake8`.
* **C++:** Seguir **Google C++ Style Guide**. Formatação com `clang-format`.
* **Docstrings:** Todas as funções e classes devem ter docstrings explicando entradas, saídas e propósito.

## 3. Padrão de Git e Commits
Mensagens de commit devem utilizar o padrão **Conventional Commits**:
* `feat(cone_detection): adiciona suporte ao modelo YOLOv8x em TensorRT`
* `fix(lidar_processing): corrige filtro de solo em declives`
* `docs(readme): atualiza instruções de compilação`
* `refactor(merge): simplifica associação de dados por distância euclidiana`

## 4. Padrão de Documentação no Código
Em arquivos Python de nós ROS 2, inclua o cabeçalho descritivo no topo do arquivo:
```python
"""
Node: perception_merge_node
Description: Realiza a fusão temporal e espacial das detecções de cones da câmera e do LiDAR.
Subscriptions:
    - /perception/camera/cones (custom_msgs/msg/ConeArray)
    - /perception/lidar/cones (custom_msgs/msg/ConeArray)
Publications:
    - /sauva/cones_3d (custom_msgs/msg/ConeArray)
"""