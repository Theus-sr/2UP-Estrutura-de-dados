# Workflow Engine — Orquestrador de Agentes Autônomos

Motor de execução responsável por resolver a ordem em que milhares de tarefas dependentes devem ser executadas, com detecção preventiva de deadlocks.

---

## Estrutura do Projeto

```
workflow_engine/
├── main.py                  # Ponto de entrada — executa os 3 cenários
├── src/
│   ├── graph.py             # RF01 — Grafo Direcionado (Lista de Adjacência)
│   ├── heap.py              # RF04 — Max-Heap de Prioridade
│   ├── algorithms.py        # RF02 (Kahn) + RF03 (DFS)
│   ├── engine.py            # Orquestrador principal + relatórios
│   └── generate_data.py     # Gerador de dados (básico / avançado / estresse)
├── data/                    # JSONs gerados (criados ao executar)
├── reports/                 # Relatórios de execução (criados ao executar)
└── tests/
    └── test_engine.py       # 22 testes unitários (RF01–RF04)
```

---

## Pré-requisitos

- Python 3.10 ou superior
- Nenhuma biblioteca externa — apenas stdlib (`random`, `uuid`, `json`, `time`)

---

## Como Executar

### 1. Executar todos os cenários (gera dados + executa pipeline)

```bash
python main.py
```

### 2. Executar apenas um cenário

```bash
python main.py --scenario basico
python main.py --scenario avancado
python main.py --scenario estresse
```

### 3. Reutilizar JSONs já gerados

```bash
python main.py --no-generate
```

### 4. Gerar apenas os dados de teste

```bash
cd src
python generate_data.py --output-dir ../data
```

### 5. Rodar os testes unitários

```bash
python tests/test_engine.py
```

---

## Requisitos Funcionais Implementados

### RF01 — Grafo Direcionado com Lista de Adjacência (`src/graph.py`)

Estrutura `DirectedGraph` com representação interna:

| Campo | Tipo | Descrição |
|---|---|---|
| `_nodes` | `dict[str, Task]` | Mapa `id → Task` |
| `_adj` | `dict[str, list[str]]` | Lista de adjacência de saída |
| `_radj` | `dict[str, list[str]]` | Lista de adjacência de entrada (grafo reverso) |
| `_in_deg` | `dict[str, int]` | Grau de entrada (usado em Kahn) |

Complexidades:
- `add_task`: **O(1)**
- `add_dependency`: **O(1)** amortizado
- `successors / predecessors`: **O(1)**
- Espaço: **O(V + E)**

---

### RF03 — Detecção de Ciclos via DFS (`src/algorithms.py` — `CycleDetector`)

Algoritmo de 3-coloração (BRANCO / CINZA / PRETO):

- **BRANCO**: nó não visitado
- **CINZA**: nó em visita (na pilha atual) → se um sucessor for CINZA, há *back-edge* → **ciclo detectado**
- **PRETO**: nó totalmente processado

Implementação **iterativa** (pilha explícita) para suportar grafos com 10.000+ nós sem `RecursionError`.

Ao detectar ciclo:
1. Registra a aresta `(u, v)` que fecha o ciclo
2. Reconstrói o caminho do ciclo via pilha
3. O `WorkflowEngine` **aborta a execução** imediatamente

Complexidade: **O(V + E)**

---

### RF02 — Ordenação Topológica / Kahn (`src/algorithms.py` — `TopologicalSorter`)

Algoritmo de Kahn com fila de prioridade integrada (RF04):

```
1. Calcula grau de entrada de todos os nós (snapshot — não muta o grafo)
2. Insere nós com grau 0 na MaxHeap (por urgência)
3. Loop:
   a. Extrai o nó de maior urgência da heap
   b. Emite o nó na sequência de execução
   c. Para cada sucessor, reduz seu grau de entrada em 1
   d. Se grau chegou a 0 → insere na heap
4. Se ao final sobram nós com grau > 0 → há ciclo (detecção de efeito colateral)
```

Complexidade:
- Com prioridade (MaxHeap): **O(V log V + E)**
- Sem prioridade (FIFO): **O(V + E)**

---

### RF04 — Fila de Prioridade Max-Heap (`src/heap.py`)

Implementação manual completa sem `heapq` ou qualquer biblioteca externa.

Array-based com índices base 0:
- `pai(i) = (i - 1) // 2`
- `filho_esq(i) = 2*i + 1`
- `filho_dir(i) = 2*i + 2`

| Operação | Complexidade |
|---|---|
| `push` | O(log N) — `_sift_up` |
| `pop` | O(log N) — `_sift_down` |
| `peek` | O(1) |
| `build` (Floyd heapify) | **O(N)** |

Desempate por FIFO (contador de inserção `insert_seq`) quando duas tarefas têm mesma urgência.

---

## Metas de Estresse (Rubrica)

| Arquivo | Tarefas | Dependências | Ciclo oculto |
|---|---|---|---|
| `input_basico.json` | 20 | 30 | Não |
| `input_avancado.json` | 200 | 500 | Não |
| `input_estresse.json` | **10.000** | **25.001** | **Sim (profundidade 15)** |

O ciclo do estresse é uma *back-edge* inserida entre dois nós distantes (15 camadas de profundidade), validando que o RF03 o detecta mesmo em grafos massivos.

---

## Prova de Carga — Tempos Medidos

| Cenário | Carga do grafo | DFS (RF03) | Kahn (RF02) | Total |
|---|---|---|---|---|
| Básico | 0.2 ms | 0.1 ms | 0.1 ms | 0.4 ms |
| Avançado | 1.2 ms | 0.3 ms | 0.8 ms | 1.3 ms |
| Estresse | 79.5 ms | 19.3 ms | — (abortado) | ~100 ms |

No cenário de estresse, a DFS detecta o ciclo profundo em **< 20 ms** com 10.000 nós e 25.001 arestas, abortando antes de executar Kahn — comportamento correto conforme RF03.

---
