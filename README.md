# Autonomous Silobag Routing & Sampling (Agro-Pathfinding)

> **Planejamento de rotas e navegação autônoma em pátios de armazenagem de grãos utilizando algoritmos de busca em espaço de estados e heurísticas admissíveis.**

---

## 1. Visão Geral do Sistema

Este projeto implementa uma solução completa para o **Problema de Roteamento para Coleta e Amostragem de Grãos em Pátios de Silobags (Agro-TIC)**. 

No contexto da classificação e certificação agroindustrial de grãos (aferição de umidade, impurezas e avarias), um agente coletor autônomo deve planejar sua rota em um pátio logístico contendo obstáculos (maquinários, valas e drenos), visitar $K$ silobags pré-designadas para coleta e descarregar o material amostrado no Laboratório Central de Classificação no menor número de passos possível.

```text
S  .  .  .  #  .
.  #  B  .  #  .
.  #  .  .  .  .
.  .  #  #  B  .
.  .  .  .  .  .
.  #  .  .  #  L
```
*Legenda: `S` = Início, `B` = Silobag (alvo), `L` = Laboratório (meta final), `#` = Obstáculo, `.` = Livre.*

---

## 2. Modelagem Formal no Espaço de Estados

O problema foi formalizado matematicamente como um problema de busca clássica:

1. **Ambiente:** Grade bidimensional $N \times M$, estática, discreta e totalmente observável.
2. **Representação de Estados ($S$):** Tuplas imutáveis e hasheáveis `(r, c, bitmask)`:
   * `r, c`: Coordenadas atuais do agente na grade;
   * `bitmask`: Inteiro representando uma máscara binária de $K$ bits, indicando exatamente quais silobags já foram coletadas.
3. **Estado Inicial ($s_0$):** Coordenadas de partida do robô com `bitmask = 0`.
4. **Ações e Modelo de Transição:** Deslocamentos ortogonais (`CIMA`, `BAIXO`, `ESQUERDA`, `DIREITA`) para células válidas adjacentes livres de obstáculos. Ao ingressar na coordenada de uma silobag, a coleta é registrada ativando o bit correspondente via operador bitwise `OR` (`|`).
5. **Teste de Objetivo:** O agente encontra-se nas coordenadas do Laboratório ($L$) e todas as $K$ silobags foram amostradas (`bitmask == (1 << K) - 1`).
6. **Custo de Caminho ($g$):** Custo de passo unitário ($c = 1$). O custo total corresponde à quantidade de passos dados.

---

## 3. Algoritmos de Busca Implementados

### A) Busca Não Informada: Busca em Largura (BFS) em Grafo
* **Propriedades:** Em espaços de busca com custos uniformes ($c = 1$), o BFS em grafo garante completude e otimalidade estrita (encontra o caminho com o menor número absoluto de passos).
* **Mecanismo:** Exploração em níveis usando fila FIFO (`collections.deque`) e controle de estados visitados com tabela hash (`set`).
* **Gargalo:** Complexidade de espaço $O(b^d)$, sofrendo com a explosão combinatória da memória conforme o número de alvos e a dimensão da grade aumentam.

### B) Busca Informada: Algoritmo A* com Heurística Admissível e Consistente
* **Função de Avaliação:** 
  $$f(n) = g(n) + h(n)$$
* **Heurística Combinada $h(n)$:**
  $$h(n) = \min_{u \in U} d_M(\text{agente}, u) + \text{MST}(U) + \min_{u \in U} d_M(u, \text{laboratório})$$
  onde:
  * $U$ é o conjunto de silobags ainda pendentes de coleta;
  * $d_M$ é a **Distância de Manhattan** ($|\Delta r| + |\Delta c|$);
  * $\text{MST}(U)$ é o custo da **Árvore Geradora Mínima** conectando todos os alvos pendentes, calculada dinamicamente via algoritmo de Prim.
* **Garantias Teóricas:**
  * **Admissibilidade ($h(n) \le h^*(n)$):** A distância de Manhattan ignora obstáculos e a MST relaxa a restrição do caminho hamiltoniano (uma árvore geradora livre tem custo $\le$ qualquer caminho que visite todos os vértices). Logo, $h(n)$ nunca superestima o custo real.
  * **Consistência (Monotonicidade):** Satisfaz a desigualdade triangular $h(n) \le c(n, a, n') + h(n')$, garantindo que todo nó expandido já possui custo ótimo, eliminando a necessidade de reabertura de nós fechados.

---

## 4. Estrutura do Projeto

```text
├── casos_de_teste/
│   ├── caso_1_moderado.txt        # Grade 6x6, 2 silobags
│   ├── caso_2_intermediario.txt   # Grade 10x10, 4 silobags
│   └── caso_3_complexo.txt        # Grade 14x14, 6 silobags
├── src/
│   ├── __init__.py
│   ├── problem.py                 # Definição de Estado, Ações e Modelagem em Grade
│   ├── search.py                  # Implementação de Nós, BFS em Grafo e A*
│   └── heuristics.py              # Heurística de Manhattan + MST (Algoritmo de Prim)
├── main.py                        # Interface de linha de comando e visualizador de rotas
├── benchmark.py                   # Script de benchmarking estatístico e geração de gráficos
├── gerar_relatorio.py             # Gerador automatizado do relatório técnico
├── graficos_desempenho.png        # Gráficos comparativos de tempo, memória e nós
├── relatorio.pdf                  # Relatório técnico com análise empírica detalhada
└── README.md                      # Documentação técnica do projeto
```

---

## 5. Instruções de Execução

### Pré-requisitos
* Python 3.10 ou superior.
* O motor de busca e a modelagem do problema utilizam **exclusivamente a biblioteca padrão do Python** (`collections`, `heapq`, `tracemalloc`, `time`, `typing`).
* Dependências opcionais para geração de gráficos estatísticos e compilação do relatório:
  ```bash
  pip install matplotlib numpy reportlab pymupdf
  ```

### Como Executar

1. **Executar todos os cenários de teste comparando BFS vs A\*:**
   ```bash
   python main.py --todos
   ```

2. **Executar um caso específico com renderização visual da rota no terminal:**
   ```bash
   python main.py --caso casos_de_teste/caso_1_moderado.txt --visualizar
   ```

3. **Executar isoladamente um dos algoritmos:**
   ```bash
   python main.py --caso casos_de_teste/caso_2_intermediario.txt --algoritmo astar --visualizar
   python main.py --caso casos_de_teste/caso_2_intermediario.txt --algoritmo bfs
   ```

4. **Executar benchmark estatístico automatizado:**
   ```bash
   python benchmark.py
   ```
   *(Gera o arquivo `graficos_desempenho.png` comparando nós expandidos, consumo de memória e tempo de CPU).*

---

## 6. Resultados e Benchmark Comparativo

Os algoritmos foram avaliados em três cenários com diferentes ordens de complexidade:

| Cenário | Dimensão / Alvos | Algoritmo | Custo Ótimo ($g$) | Nós Expandidos | Nós Gerados | Fronteira Máx. | Tempo de Execução | Memória de Pico |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Moderado** | $6 \times 6$ ($K=2$) | BFS | 14 passos | 120 | 123 | 16 | 1,10 ms | 23,79 KB |
| | | **A\*** | **14 passos** | **14** | **29** | **15** | **0,38 ms** | **7,19 KB** |
| **Intermediário** | $10 \times 10$ ($K=4$) | BFS | 38 passos | 1.055 | 1.062 | 61 | 10,36 ms | 138,71 KB |
| | | **A\*** | **38 passos** | **176** | **236** | **60** | **4,19 ms** | **50,34 KB** |
| **Complexo** | $14 \times 14$ ($K=6$) | BFS | 60 passos | 7.665 | 7.710 | 332 | 83,22 ms | 1.230,98 KB |
| | | **A\*** | **60 passos** | **1.134** | **1.374** | **227** | **29,26 ms** | **250,69 KB** |

### Destaques de Desempenho
* **Convergência Ótima:** Ambos os algoritmos comprovaram a otimalidade formal do modelo, convergindo rigorosamente para o mesmo custo mínimo em todas as instâncias (14, 38 e 60 passos).
* **Eficiência de Poda do A\*:** A heurística baseada em MST podou **88,3%** dos nós expandidos no caso moderado e **85,2%** no caso complexo.
* **Economia de Memória:** Enquanto a fila do BFS cresceu exponencialmente até consumir 1,23 MB de memória RAM no caso complexo, o A* conteve a expansão em apenas 250 KB (redução de **79,6%** no consumo de memória).
* **Aceleração Computacional:** O A* operou de **2,5× a 2,9× mais rápido** que a busca exaustiva.
