# Problem Definition — Terrain-Aware Pathfinding via Dijkstra’s Algorithm

## 1. Contexto Geral

Robôs móveis terrestres — industriais, exploratórios ou educacionais — frequentemente operam em ambientes onde o terreno apresenta variações de altura, irregularidades e inclinações. Esses robôs dependem de mapas discretizados, derivados de sensores, simulações ou topografias pré-processadas, para planejar e executar deslocamentos.

Nessa realidade, a escolha do caminho depende não apenas da distância, mas também do custo físico associado à variação de altura, que influencia o consumo energético, estabilidade mecânica e eficiência global da navegação.

## 2. Problema Central

Dado um terreno representado como uma matriz de alturas:

\[
H \in \mathbb{R}^{m \times n}
\]

onde cada entrada \(H_{i,j}\) define a altitude da célula \((i,j)\), desejamos:

**Encontrar o caminho de menor custo entre dois pontos do terreno, penalizando movimentos que impliquem grandes variações de altura.**

Operacionalmente:

> Determinar uma sequência de células adjacentes que minimize o custo acumulado derivado de deslocamento e inclinação.

## 3. Modelagem do Terreno como Grafo

Para resolver o problema, convertemos o terreno em um grafo ponderado:

- Cada célula \((i,j)\) torna-se um vértice.
- Células adjacentes são conectadas por arestas (4 ou 8 vizinhos).
- Cada aresta recebe um peso não-negativo que depende:
  - da distância plana;
  - da variação de altura entre as células;
  - de um fator multiplicador \(\alpha\).

Este grafo reflete diretamente características físicas do ambiente.

## 4. Natureza do Custo

O custo de mover-se de \((i,j)\) para \((k,l)\) é definido por:

\[
w((i,j),(k,l)) = d_{\text{plano}} \cdot \left( 1 + \alpha \cdot |\Delta h| \right)
\]

onde:

- \(d_{\text{plano}} \in \{1, \sqrt{2}\}\);
- \(\Delta h = H_{k,l} - H_{i,j}\);
- \(\alpha\) controla o impacto da inclinação;
- todos os custos são não-negativos, atendendo ao requisito do algoritmo de Dijkstra.

## 5. Formulação Formal

Dado o grafo:

\[
G = (V, E, w)
\]

e dois vértices \(s\) (start) e \(t\) (target), queremos um caminho \(P\) tal que:

\[
\text{cost}(P) = \sum_{e \in P} w(e)
\]

\[
\text{e } \text{cost}(P) = \min_{P' \in \mathcal{P}_{s \to t}} \text{cost}(P')
\]

## 6. Papel do Algoritmo de Dijkstra

O algoritmo de Dijkstra encontra o caminho de menor custo em grafos com pesos não-negativos.  
Neste projeto, ele serve como:

- baseline explicável e determinístico;
- ponto de partida para algoritmos informados (como A*);
- ferramenta educacional para conectar teoria de grafos com navegação robótica.

## 7. Relevância Educacional

Este módulo:

- ensina modelagem de grafos a partir de terreno realista;
- introduz funções de custo fisicamente motivadas;
- demonstra o impacto de inclinações no custo de navegação;
- prepara para a transição natural ao algoritmo A*.

## 8. Limites do Escopo

Este módulo **não** envolve:

- incerteza probabilística;
- ruído de sensores;
- cinemática contínua;
- otimização multiobjetivo complexa.

O objetivo é estabalecer a fundação teórica para algoritmos subsequentes.

## 9. Objetivos de Aprendizagem

Ao final deste módulo, o estudante deve ser capaz de:

- modelar um terreno como grafo ponderado;
- definir funções de custo coerentes com aplicações robóticas;
- aplicar Dijkstra corretamente;
- analisar a influência da topografia na trajetória final.

## 10. Continuidade

A conclusão deste módulo fornece a base conceitual necessária para introduzir heurísticas e avançar para o algoritmo *A\**.
