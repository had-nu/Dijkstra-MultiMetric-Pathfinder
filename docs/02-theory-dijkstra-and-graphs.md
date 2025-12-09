# Theory — Graphs, Shortest Paths and the Dijkstra Algorithm

## 1. Introdução

O problema de caminho mínimo consiste em determinar a rota de menor custo entre dois vértices em um grafo ponderado.  
É um tema fundamental em algoritmos, redes, logística e navegação robótica.

Neste módulo, focamos no caso clássico em que:

- todos os pesos das arestas são não-negativos;
- o grafo é estático;
- busca-se o menor custo acumulado.

Esse é o domínio ideal para o algoritmo de *Dijkstra*.

## 2. Grafo Ponderado: Definição Formal

Um grafo ponderado é representado por:

$$
G = (V, E, w)
$$

onde:

- $$\(V\)$$: vértices,
- $$\(E\)$$: arestas,
- $$\(w: E \to \mathbb{R}\)$$: pesos.

No contexto do projeto:

- vértices = células do terreno,
- arestas = vizinhança (4 ou 8),
- pesos = custo de deslocamento penalizado por inclinação.

## 3. Custo de Caminho

Um caminho $$\(P\)$$ é uma sequência de vértices conectados:

$$
P = \langle v_0 = s, v_1, \dots, v_k = t \rangle
$$

Seu custo total é:

$$
\text{cost}(P) = \sum_{i=0}^{k-1} w(v_i, v_{i+1})
$$

O problema é encontrar o caminho $$\(P^\*\)$$ tal que:

$$
\text{cost}(P^\*) = \min_{P}
$$

## 4. Pesos Não-Negativos

O algoritmo de Dijkstra exige:

$$
w(u,v) \ge 0
$$

Razões:

1. A fila de prioridade garante que o vértice extraído tem a menor distância possível.
2. Pesos negativos poderiam alterar caminhos já processados.
3. A prova de corretude depende da monotonicidade dos custos.

A modelagem de terreno naturalmente garante $$\(w \ge 0\)$$.

## 5. Ideia Central do Algoritmo

Dijkstra segue uma estratégia gananciosa:

- mantém uma estimativa da menor distância até cada vértice;
- usa relaxamento para atualizar estimativas;
- extrai repetidamente o vértice de menor custo estimado;
- garante que distâncias finalizadas são ótimas.

A fila de prioridade (min-heap) torna a abordagem eficiente.

## 6. Complexidade

Usando min-heap:

$$
O((V + E) \log V)
$$

Esta é a versão adotada aqui, pois reflete boas práticas e é adequada a grafos derivados de terrenos.

## 7. Versão Abstrata do Algoritmo

1. Inicializar:
   - $$\(d[s] = 0\)$$,
   - $$\(d[v] = \infty\) para \(v \ne s\)$$.

2. Inserir todos os vértices na fila de prioridade.

3. Enquanto a fila não estiver vazia:
   - extrair o vértice $$\(u\)$$ com menor $$\(d[u]\)$$;
   - relaxar todas as arestas $$\((u,v)\)$$.

O relaxamento é:

$$
\text{se } d[v] > d[u] + w(u,v):
$$
$$
d[v] = d[u] + w(u,v)
$$
$$
\pi[v] = u
$$

4. Ao final, $$\(d[t]\)$$ é o custo ótimo e $$\(\pi\)$$ define o caminho.

## 8. Correção do Algoritmo (Resumo)

A prova de corretude baseia-se em três fatos:

1. A fila de prioridade processa vértices em ordem não-decrecente de custo.
2. Em grafos com pesos não-negativos, a primeira vez que um vértice é extraído ele já tem seu custo ótimo.
3. Relaxamentos posteriores não podem melhorar vértices já finalizados.

Referência formal: *CLRS*, Capítulo 24.

## 9. Conexão com Navegação Robótica

Dijkstra é importante em robótica porque:

- transforma mapas discretizados em grafos navegáveis;
- permite definir custos energéticos, riscos ou dificuldades;
- gera trajetórias determinísticas e explicáveis;
- serve como baseline para algoritmos informados como o A\*.

## 10. Ponte com o Algoritmo A*

O A\* é uma extensão direta de Dijkstra:

$$
f(n) = g(n) + h(n)
$$

onde $$\(h(n)\)$$ é uma heurística admissível.

Quando $$\(h(n) = 0\)$$, A\* reduz-se exatamente a Dijkstra.  
Por isso, dominar Dijkstra é pré-requisito conceitual para entender A\*.

## 11. Referências

- Russell & Norvig, *Artificial Intelligence: A Modern Approach*  
- Cormen, Leiserson, Rivest e Stein, *Introduction to Algorithms* (CLRS), Capítulos 23–24  
- Kleinberg & Tardos, *Algorithm Design*  
- Steven M. LaValle, *Planning Algorithms*  
- MIT OCW 6.006 e 6.034  
- UC Berkeley CS188
