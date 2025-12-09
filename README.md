# Dijkstra-MultiMetric-Pathfinder  
**IA Clássica Aplicada à Navegação em Terreno Não-Uniforme**

Este repositório integra uma série de quatro projetos educativos sobre Inteligência Artificial Clássica, estruturados para estudantes, iniciantes em IA e recrutas técnicos que desejam compreender algoritmos fundamentais por meio de problemas concretos e implementações reproduzíveis.

Cada projeto foca em um algoritmo canónico — Dijkstra, A*, Minimax e Algoritmos Genéticos — sempre acompanhado de documentação didática, modelagem formal e aplicações práticas conectadas à robótica, otimização e tomada de decisão.

---

## 1. Sobre Esta Série de Projetos

Esta série foi concebida como caderno de estudos para ensinar IA clássica de forma aplicada, combinando:

- **Teoria estruturada**, com explicações progressivas e referências formais.  
- **Aprendizagem baseada em problemas (PBL)**: cada repositório parte de um problema realista.  
- **Implementações claras e modulares**, adequadas para estudo e manutenção.  
- **Documentação acessível**, escrita para quem precisa compreender o algoritmo e não apenas reproduzir código.  
- **Contexto prático**, especialmente voltado a sistemas robóticos e engenharia aplicada.

Os quatro repositórios seguem a mesma arquitetura documental, permitindo estudo incremental e comparativo entre algoritmos.

---

## 2. Finalidade Educacional

Este repositório tem como objetivo ajudar estudantes e profissionais a:

- compreender profundamente como o algoritmo de Dijkstra opera;  
- relacionar o algoritmo com problemas reais de navegação e grafos;  
- modelar terrenos físicos como grafos ponderados;  
- analisar funções de custo sob a perspectiva de robótica móvel;  
- visualizar caminhos otimizados em terrenos com variações de inclinação.

A abordagem privilegia clareza pedagógica sem sacrificar rigor matemático.

---

## 3. Estrutura do Repositório
```
Dijkstra-MultiMetric-Pathfinder/
├── README.md
├── docs/
│   ├── 01-problem-definition.md
│   ├── 02-theory-dijkstra-and-graphs.md
│   ├── 03-terrain-modeling-and-cost-functions.md
│   ├── 04-implementation-notes.md
│   └── 05-experiments-and-results.md
├── data/
│   ├── terrain_demo_01.csv
│   └── terrain_demo_02.csv
├── src/
│   ├── __init__.py
│   ├── graph_model.py
│   ├── dijkstra.py
│   └── terrain_loader.py
├── notebooks/
│   └── 01_exploratory_terrain_and_paths.ipynb
└── tests/
    └── test_dijkstra_basic.py
```

**Função das principais pastas:**

- **docs/** – documentação técnica detalhada (teoria, modelagem, experimentos).  
- **src/** – implementação modular do algoritmo e estruturas auxiliares.  
- **data/** – terrenos sintéticos usados como base para experimentos.  
- **notebooks/** – exploração visual e experimentação interativa.  
- **tests/** – testes mínimos para validar a corretude do algoritmo.

---

## 4. Guia Teórico (Resumo)

O algoritmo de *Dijkstra* encontra o caminho de menor custo em grafos com pesos não-negativos, utilizando relaxamento iterativo e uma fila de prioridade para eficiência.

### Por que isso importa em IA?

- É a base para algoritmos mais sofisticados, como o *A\**.  
- Serve como modelo para otimização de caminhos, custos e recursos.  
- É um exemplo clássico de algoritmo ganancioso com prova formal de optimalidade.  

### Por que isso importa em robótica?

Robôs móveis executam continuamente:

- planeamento de trajetória;  
- escolha de caminhos de menor custo energético;  
- decisões sob topografia variável;  
- navegação em ambientes discretizados.

Dijkstra oferece um baseline robusto antes de aplicar heurísticas (como em A\*).

---

## 5. O Problema Abordado (PBL)

Este repositório aborda, portanto, o problema de:

*Encontrar o caminho de menor custo energético para um robô terrestre atravessar um terreno representado por uma matriz de alturas, onde cada movimento possui penalidade proporcional à inclinação.*

Modelagem:

- cada célula do terreno é um vértice;  
- células adjacentes formam arestas;  
- o custo da aresta depende da variação de altura (Δh) e da distância plana;  
- subidas íngremes geram custos maiores.

Isso cria um cenário aplicado e interpretável, diretamente inspirado em navegação de UGVs, AMRs e rovers.

---

## 6. O Que Este Repositório Ensina

- Conceitos fundamentais de grafos ponderados.  
- Funcionamento e prova de corretude de Dijkstra.  
- Construção de uma função de custo fisicamente motivada.  
- Conversão de terrenos em grafos espaciais.  
- Visualização do caminho encontrado e do custo acumulado.  
- Comparação entre diferentes perfis de terreno.  
- Relações entre geometria, energia e planejamento de trajetórias.

---

## 7. Como Executar (Resumo)

Instruções completas no `README` técnico secundário.  
Fluxo geral:

1. Instalar dependências.  
2. Carregar um terreno de exemplo.  
3. Executar o algoritmo.  
4. Visualizar o caminho e métricas associadas.  

O foco aqui é estudo, não performance.

---

## 8. Estrutura dos Documentos

Os principais documentos encontram-se em `docs/`:

- **01-problem-definition.md** → descrição formal do problema e contexto robótico.  
- **02-theory-dijkstra-and-graphs.md** → fundamentos teóricos e referências.  
- **03-terrain-modeling-and-cost-functions.md** → modelagem do terreno, funções de custo e justificativas.  
- **04-implementation-notes.md** → arquitetura da implementação.  
- **05-experiments-and-results.md** → visualizações, análises e comparação de resultados.

---

## 9. Referências Canónicas

- *Artificial Intelligence: A Modern Approach* — Russell & Norvig  
- *Introduction to Algorithms* — Cormen, Leiserson, Rivest e Stein (CLRS)  
- Material clássico de grafos, busca e otimização  
- Artigos e notas sobre navegação robótica e modelagem de terreno

---

## 10. Licença

Projeto educativo de uso livre.

