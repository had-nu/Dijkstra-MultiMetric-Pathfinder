# Theory — Graphs, Shortest Paths and the Dijkstra Algorithm

## 1. Introdução

O problema de caminho mínimo consiste em determinar a rota de menor custo entre dois vértices em um grafo ponderado. É um tema fundamental em algoritmos, redes, logística e navegação robótica.

Neste módulo, focamos no caso clássico em que:

- todos os pesos das arestas são não-negativos;
- o grafo é estático;
- busca-se o menor custo acumulado.

Esse é o domínio ideal para o algoritmo de *Dijkstra*. Mas antes de prosseguir, vamos dar um passo atrás e revisar os fundamentos teóricos.

## 2. A Arte de Resolver Problemas

Vamos usar um problema real como ponto de partida. Imagine-se de férias na Roménia, na cidade de Arad, com um bilhete de avião não reembolsável para partir de Bucareste no dia seguinte. À sua frente, placas indicam estradas para Sibiu, Timisoara e Zerind. Qual é o melhor caminho a seguir? O mais rápido? O que tem menos paragens? Este é um desafio de planeamento que todos nós enfrentamos, seja ao traçar uma rota no GPS ou ao organizar tarefas complexas. 

Para um ser humano, a decisão pode envolver intuição e experiência. Mas como podemos traduzir este mapa mental de cidades e estradas para a linguagem fria e lógica de um computador? Este é o ponto de partida da nossa jornada, a transformação de um problema do mundo real num modelo que uma máquina pode compreender, analisar e, por fim, resolver de forma ótima. 

Para ilustrar este processo, vamos usar um exemplo clássico dos manuais de Inteligência Artificial: o problema de encontrar a melhor rota através de um mapa simplificado da Roménia.

### 2.1. Modelando o Mundo: Estados e Ações

A primeira etapa para que um computador possa "raciocinar" sobre o nosso problema é impor ordem e estrutura. Em Inteligência Artificial, formalizamos este processo com dois conceitos-chave:

- Estado: Uma representação de uma situação específica no problema. Na nossa viagem, um estado é simplesmente a cidade onde nos encontramos (por exemplo, o estado é "estar em Arad").
- Ação: Uma operação que nos transporta de um estado para outro. No nosso caso, uma ação é "seguir pela estrada de Arad para Sibiu", o que resulta num novo estado: "estar em Sibiu".

Ao definir todos os estados possíveis (todas as cidades) e todas as ações que os conectam (todas as estradas), criamos uma representação abstrata do problema. Essencialmente, construímos um mapa estruturado que o computador pode ler.

Este "mapa" não é apenas um desenho; é uma estrutura matemática poderosa e versátil que está no coração da ciência da computação e da IA. É a fundação que nos permite aplicar estratégias sistemáticas para encontrar a solução, seja ela o caminho mais curto, o mais barato ou o que melhor se adequa a qualquer outro critério. Essa estrutura fundamental é chamada de *grafo*.

Com o nosso problema agora modelado em termos de estados e ações, precisamos de uma linguagem formal para representar este mapa de possibilidades. É aqui que os grafos entram em cena, fornecendo a estrutura sobre a qual os algoritmos de busca irão operar.

## 3. Apresentando Grafos: Definição Formal do Mapa de Todos os Problemas

> *Um grafo é uma coleção de vértices (ou nós) e arestas (ou ligações) que conectam esses vértices. Cada vértice representa um estado, e cada aresta representa uma ação que conecta dois estados.*

De forma intuitiva, um grafo é exatamente o que parece: um mapa de pontos conectados por linhas. Esta estrutura simples é a linguagem universal que a computação usa para descrever relacionamentos. Dominá-la é como aprender o alfabeto da resolução de problemas algorítmica.

Um grafo $$G$$ é um par de conjuntos, $$G = (V, E)$$, onde $$V$$ é o conjunto de todos os vértices (as nossas cidades) e $$E$$ é o conjunto de todas as arestas (as nossas estradas). Essa estrutura elegante serve para modelar não apenas mapas, mas também redes sociais (pessoas como vértices, amizades como arestas), a internet (páginas como vértices, links como arestas) e, claro, os espaços de busca para problemas de IA.

### 3.1. Os Componentes Essenciais de um Grafo

Para usar os grafos de forma eficaz, precisamos de conhecer os seus componentes e as suas variações.

| Conceito | Explicação e Analogia |
|---|---|
| **Vértice (ou Nó)** | Um ponto individual no grafo. Representa um estado, uma entidade ou um local. <br> **Analogia:** Uma cidade no mapa, uma pessoa numa rede social. |
| **Aresta (ou Ligação)** | Uma conexão entre dois vértices. Representa uma relação, uma transição ou uma ação. <br> **Analogia:** Uma estrada que liga duas cidades, uma amizade entre duas pessoas. |
| **Grafo** | A coleção completa de vértices e arestas. A estrutura que modela o problema. <br> **Analogia:** O mapa completo de todas as cidades e estradas, a rede social inteira. |
| **Grafo Ponderado** | Um grafo onde as arestas possuem pesos (custos). <br> **Analogia:** Mapa onde cada estrada tem uma distância ou pedágio específico. |
| **Grafo Direcionado** | As conexões têm direção única (de A para B). <br> **Analogia:** Ruas de sentido único (mão única). |
| **Grafo Não-Direcionado** | As conexões não têm direção ou são bidirecionais. <br> **Analogia:** Ruas de mão dupla. |
| **Peso** | O custo ou distância associada a uma aresta. Representa a dificuldade ou custo de uma transição. <br> **Analogia:** A distância em quilómetros ou o tempo de viagem numa estrada. |
| **Caminho** | Uma sequência de vértices conectados. Representa uma sequência de estados ou ações. <br> **Analogia:** Uma rota que liga duas cidades, uma série de amizades. |
| **Custo de Caminho** | A soma dos pesos de todas as arestas em um caminho. Representa o custo total de uma sequência de estados ou ações. <br> **Analogia:** O total de quilómetros percorridos ou tempo gasto numa rota. |
| **Caminho Mínimo** | O caminho com o menor custo de caminho. Representa a solução ótima do problema. <br> **Analogia:** A rota mais curta ou mais rápida entre duas cidades, a melhor sequência de amizades. |

## 4. Grafos Ponderados

Formalmente, descrevemos um grafo $$G$$ ponderado como:

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

Tomemos o nosso exemplo inicial como um grafo: o problema de viagem na Roménia é um exemplo perfeito de um grafo ponderado e não-direcionado:

- As cidades são os vértices.
- As estradas são as arestas.
- As estradas são de mão dupla, tornando o grafo não-direcionado.
- Cada estrada tem uma distância associada (um custo), tornando o grafo ponderado.

Agora que temos o mapa do nosso problema formalmente representado como um grafo, a questão que se segue é óbvia: como encontramos o caminho? Precisamos de uma estratégia para navegar nesta estrutura, o que nos leva diretamente ao mundo dos algoritmos de busca.

## 5. Custo de Caminho

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

## 6. Pesos Não-Negativos

O algoritmo de Dijkstra exige:

$$
w(u,v) \ge 0
$$

Razões:

1. A fila de prioridade garante que o vértice extraído tem a menor distância possível.
2. Pesos negativos poderiam alterar caminhos já processados.
3. A prova de corretude depende da monotonicidade dos custos.

A modelagem de terreno naturalmente garante $$\(w \ge 0\)$$.

## 7. Navegando no Grafo: Algoritmos de Busca Básicos
### 7.1. A Estratégia de Exploração

Um algoritmo de busca é uma receita sistemática para explorar um grafo, partindo de um estado inicial, em busca de um estado objetivo. A estratégia baseia-se em manter e expandir o que chamamos de fronteira (frontier): o conjunto de todos os nós que já foram alcançados, mas cujos "vizinhos" ainda não foram explorados.

A forma como o algoritmo escolhe qual nó da fronteira expandir a seguir define a sua natureza e as suas propriedades. Vamos analisar as nossas duas primeiras estratégias, conhecidas como Buscas Não-Informadas, pois não utilizam nenhuma informação sobre a "distância" até o objetivo.

### 7.2. Duas Estratégias Fundamentais

**Busca em Largura (Breadth-First Search - BFS)**

Vamos analisar a nossa primeira estratégia, a Busca em Largura. Pensem nela como o explorador mais cauteloso e sistemático que existe.

- Lógica: A BFS explora o grafo "camada por camada". Imagine derramar um balde de tinta no nó inicial. A BFS explora o grafo da mesma forma que a tinta se espalharia: cobrindo uniformemente todos os pontos à mesma "distância" antes de avançar para a próxima camada. É metódica, justa e exaustiva.

- Estrutura de Dados: Utiliza uma fila (FIFO - First-In, First-Out). Ao usar uma fila, o algoritmo é forçado a expandir os nós na ordem em que foram descobertos. Isso garante que todos os nós a uma distância de 1 sejam processados antes de qualquer nó a uma distância de 2, e assim por diante, forçando a exploração camada por camada.

- Garantia: Encontra o caminho com o menor número de arestas (passos) até o objetivo.

**Busca em Profundidade (Depth-First Search - DFS)**

A Busca em Profundidade é o oposto: é uma exploradora agressiva e obstinada, que mergulha de cabeça num caminho antes de considerar alternativas.

- Lógica: A partir de um nó, explora o mais fundo possível por um dos seus ramos. Apenas quando chega a um "beco sem saída" (ou ao objetivo), ele retrocede (backtracking) para explorar um ramo alternativo.

- Estrutura de Dados: Utiliza uma pilha (LIFO - Last-In, First-Out). Com uma pilha, o último nó descoberto é o primeiro a ser explorado. Isso faz com que o algoritmo siga um caminho até ao seu extremo antes de retroceder para explorar os vizinhos do nó anterior, resultando numa exploração em profundidade.

- Garantia: Pode encontrar uma solução muito rapidamente se ela estiver num ramo profundo. No entanto, não há garantia de que a solução encontrada seja a melhor (a mais curta).

### 7.3. Comparando as Estratégias

A escolha entre BFS e DFS depende inteiramente dos requisitos do problema.

| Critério | Busca em Largura (BFS) | Busca em Profundidade (DFS) |
|---|---|---|
| **Completude** | Sim. Se uma solução existe, o BFS irá encontrá-la. | Não, em grafos com ramos infinitos. No entanto, para grafos finitos, é completo, mas pode ser extremamente ineficiente e não é garantido que termine se houver ciclos, a menos que se mantenha um registo dos nós visitados. |
| **Otimização** | Condicionalmente. Sim, mas apenas se o custo de todas as arestas for idêntico. É por isso que encontra o caminho com o menor número de passos, mas não necessariamente o caminho mais barato num grafo ponderado. | Não. A primeira solução que encontra pode não ser a melhor (a mais curta). |
| **Complexidade de Tempo** | $$O(b^d)$$ | $$O(b^m)$$ |
| **Complexidade de Espaço** | $$O(b^d)$$ | $$O(bm)$$ |

> Legenda: $$b$$ é o fator de ramificação (número médio de vizinhos), $$d$$ é a profundidade da solução mais rasa, e $$m$$ é a profundidade máxima do grafo.

A Busca em Largura deu-nos o caminho com menos paragens. Mas será esse sempre o melhor caminho? E se uma rota com mais cidades for, na verdade, centenas de quilómetros mais curta? Precisamos de um algoritmo que leve os pesos das arestas em consideração.

## 8. Encontrando o Menor Custo com Dijkstra
### 8.1 O Custo do Caminho

Em problemas do mundo real, como a nossa viagem pela Roménia, o que realmente importa é o custo total do caminho — a soma das distâncias (pesos) das estradas que compõem a rota. A Busca em Largura (BFS) encontraria o caminho 'Arad -> Sibiu -> Fagaras -> Bucareste' (3 paragens), mas o seu custo total é de 140 + 99 + 211 = 450 km. Será que existe um caminho mais curto, mesmo que passe por mais cidades?
Para resolver isso, precisamos de um algoritmo que priorize os caminhos de menor custo acumulado, e não apenas o menor número de passos.

### 8.2 A Intuição do Algoritmo de Dijkstra

Este algoritmo é amplamente conhecido na ciência da computação como Algoritmo de Dijkstra. No contexto dos livros de referência de IA, como Artificial Intelligence: A Modern Approach, ele é classificado como um tipo de Busca de Custo Uniforme (Uniform-Cost Search), que por sua vez é uma instância de uma busca best-first (que prioriza o "melhor" nó a expandir).

A sua lógica é elegantemente simples e gananciosa (greedy):

> A cada passo, expanda o nó na fronteira que possui o menor custo total acumulado desde o ponto de partida.

Aqui reside uma das mais belas ideias da algoritmia. A Busca em Largura (BFS) usa uma fila normal, onde a regra é "primeiro a entrar, primeiro a sair". O Algoritmo de Dijkstra simplesmente troca essa fila por uma "fila de prioridade", onde a regra passa a ser "o de menor custo acumulado, o próximo a sair". Essa pequena mudança na estrutura de dados transforma um algoritmo que encontra o caminho com menos passos num algoritmo que encontra o caminho com menor custo, uma evolução poderosa.

### 8.3. Dijkstra em Ação

Vamos aplicar a lógica de Dijkstra para os primeiros passos da nossa viagem de Arad para Bucareste, usando os custos do mapa da Roménia.

1. Início:
   - Nó atual: Arad (custo total = 0).
   - Expandindo Arad: Adicionamos os seus vizinhos à fronteira (fila de prioridade) com os seus respetivos custos acumulados.
   - Fronteira: {Zerind(75), Timisoara(118), Sibiu(140)}
2. Passo 1:
   - Próximo nó: Zerind é escolhido, pois tem o menor custo (75).
   - Expandindo Zerind: Adicionamos o seu vizinho Oradea à fronteira. O custo para chegar a Oradea é o custo até Zerind mais a distância de Zerind a Oradea (75 + 71 = 146).
   - Fronteira: {Timisoara(118), Sibiu(140), Oradea(146)}
3. Passo 2:
   - Próximo nó: Timisoara é escolhido (custo 118).
   - Expandindo Timisoara: Adicionamos o seu vizinho Lugoj à fronteira (custo 118 + 111 = 229).
   - Fronteira: {Sibiu(140), Oradea(146), Lugoj(229)}
4. Passo 3:
   - Próximo nó: Sibiu é o próximo (custo 140).
   - Expandindo Sibiu: Os seus vizinhos são adicionados: Rimnicu Vilcea (140 + 80 = 220) e Fagaras (140 + 99 = 239).
   - Fronteira: {Oradea(146), Rimnicu Vilcea(220), Lugoj(229), Fagaras(239)}

O algoritmo continuaria este processo, sempre escolhendo o nó de menor custo total na fronteira, até que o nó objetivo (Bucareste) seja selecionado para expansão. Nesse momento, temos a garantia de ter encontrado o caminho de menor custo total.

### 8.4. A Condição de Ouro

E aqui reside a genialidade do algoritmo de Dijkstra: com uma regra gananciosa tão simples, ele nos dá a garantia de ouro de encontrar o caminho de custo mínimo absoluto. No entanto, esta garantia depende de uma restrição fundamental: o algoritmo só funciona *se não houver arestas com pesos negativos*. A lógica do algoritmo baseia-se na premissa de que, uma vez que um nó é expandido, o custo para chegar até ele é o menor possível. Um peso negativo poderia quebrar essa garantia, pois permitiria que um caminho mais longo (em número de passos) se tornasse, subitamente, mais "barato" no futuro, invalidando as decisões gananciosas já tomadas.

Partimos de um problema de viagem, o modelamos como um grafo e, em seguida, aplicamos um algoritmo rigoroso e eficiente para descobrir o melhor caminho. Esta jornada da abstração à otimização ilustra a essência da resolução de problemas em Inteligência Artificial.

### 8.5. Ideia Central do Algoritmo

Dijkstra segue uma estratégia gananciosa:

- mantém uma estimativa da menor distância até cada vértice;
- usa relaxamento para atualizar estimativas;
- extrai repetidamente o vértice de menor custo estimado;
- garante que distâncias finalizadas são ótimas.

A fila de prioridade (min-heap) torna a abordagem eficiente.

### 8.6. Complexidade

Usando min-heap:

$$
O((V + E) \log V)
$$

Esta é a versão adotada aqui, pois reflete boas práticas e é adequada a grafos derivados de terrenos.

### 8.7. Versão Abstrata do Algoritmo

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

## 9. Correção do Algoritmo (Resumo)

A prova de corretude baseia-se em três fatos:

1. A fila de prioridade processa vértices em ordem não-decrecente de custo.
2. Em grafos com pesos não-negativos, a primeira vez que um vértice é extraído ele já tem seu custo ótimo.
3. Relaxamentos posteriores não podem melhorar vértices já finalizados.

Referência formal: *CLRS*, Capítulo 24.

## 10. Conexão com Navegação Robótica

Dijkstra é importante em robótica porque:

- transforma mapas discretizados em grafos navegáveis;
- permite definir custos energéticos, riscos ou dificuldades;
- gera trajetórias determinísticas e explicáveis;
- serve como baseline para algoritmos informados como o A\*.

## 11. Ponte com o Algoritmo A*

O A\* é uma extensão direta de Dijkstra:

$$
f(n) = g(n) + h(n)
$$

onde $$\(h(n)\)$$ é uma heurística admissível.

Quando $$\(h(n) = 0\)$$, A\* reduz-se exatamente a Dijkstra.  
Por isso, dominar Dijkstra é pré-requisito conceitual para entender A\*.

## 12. Bibliografia

- Russell & Norvig, *Artificial Intelligence: A Modern Approach*  
- Cormen, Leiserson, Rivest e Stein, *Introduction to Algorithms* (CLRS), Capítulos 23–24  
- Kleinberg & Tardos, *Algorithm Design*  
- Steven M. LaValle, *Planning Algorithms*  
- MIT OCW 6.006 e 6.034  
- UC Berkeley CS188
