# Terrain Modeling and Cost Functions

Este documento detalha a metodologia de conversão de dados topográficos brutos em uma estrutura de grafo ponderado, adequada para algoritmos de caminho mínimo como Dijkstra e A*.  

A modelagem ultrapassa a simples conectividade de grade, integrando restrições físicas e custos energéticos para simular a navegação de um agente sobre um relevo não-uniforme.

---

## 1. Representação do Terreno (Reconstrução via Odometria)

No contexto de navegação robótica baseada em inerciais, não partimos de um mapa topográfico pré-existente (como imagens de satélite). O terreno é reconstruído progressivamente a partir da odometria estimada pelo modelo TinyML. Portanto, a representação do terreno não é estática, mas derivada da trajetória histórica do agente.

### 1.1. Representação Formal: Da Trajetória ao Grid

Seja $$T$$ a sequência de poses estimadas pelo modelo de odometria no tempo $$t$$:

$$
T={(x_t,y_t,z_t)∣t=0,…,T}
$$

O terreno é modelado como uma matriz de ocupação e elevação $$H \in (\mathbb{R} \cup \{\emptyset\})^{m \times n}$$, onde cada célula representa uma região do espaço físico discretizado. Diferente de imagens comuns, esta matriz deve ser capaz de representar áreas que ainda não foram exploradas (ou seja, áreas desconhecidas).

$$
H = \begin{bmatrix}
h_{0,0} & \dots & h_{0,n-1} \\
\vdots & \ddots & \vdots \\
h_{m-1,0} & \dots & h_{m-1,n-1}
\end{bmatrix}
$$

Inicialmente, todo o conhecimento do mundo é nulo (terra incognita):

$$
\forall (i,j), H_{i,j} = \emptyset
$$

A matriz é preenchida pela projeção da trajetória $$T$$ no plano 2D. Para uma pose $$(xt​,yt​,zt​)$$, a célula correspondente (it​,jt​) é atualizada:

$$
H_{it,jt} \leftarrow f(H_{it,jt}, z_t)
$$

Onde $$f$$ é uma função de fusão de dados(média móvel ou último valor) que lida com visitas repetidas à mesma célula (loop closure).

### 1.2. Resolução Espacial e Escala $$\(\delta\)$$

A matriz é *adimensional* em seus índices. Para conferir significado físico, definimos a resolução espacial $$\(\delta\)$$ (metros/pixel). A resolução define o nível de detalhe e o tamanho dos obstáculos detectáveis:

- $$\(\delta\)$$ muito grande: Perda de fidelidade (obstáculos pequenos desaparecem, aliasing topográfico).

- $$\(\delta\)$$ muito pequeno: Explosão combinatória no número de nós do grafo ($$N=m\times n$$), elevando o custo computacional do Dijkstra/A*.

### 1.3. Mapeamento Índice ↔ Espaço Físico

É importante distinguir entre a posição no array (memória) e a posição no mundo (física). Para converter as coordenadas do mundo real $$(x,y)$$ para os índices da matriz $$(i,j)$$, utilizamos uma transformação linear que considera a resolução e a origem do sistema:

$$
\begin{cases}
j=⌊\frac{\delta_x-x_{min}}{\delta}⌋\\
i=⌊\frac{\delta_y-y_{max}-y}{\delta}⌋
(se\ a\ origem\ da\ imagem\ for\ top-left)\\
\end{cases}
$$

> Nota: O uso de $$x_{min}$$ e $$y_{max}$$ é necessário para garantir que coordenadas negativas da odometria (ex.: o robô andou para trás do ponto de partida) sejam mapeadas corretamente para índices positivos de array.

### 1.4. Tipologia dos Dados de Entrada

Os experimentos neste projeto utilizam três categorias de dados para validar a robustez do algoritmo::

**a. Dados Sintéticos (Procedurais):**

- Gerados via Ruído de Perlin ou Simplex Noise.

- Característica: Gradientes suaves, contínuos e diferenciáveis. Ótimos para testar a estabilidade do algoritmo em encostas progressivas.

**b. Dados Determinísticos (Hard-coded):**

- Matrizes desenhadas à mão (ex: rampas perfeitas, escadas, "walls").

- Característica: Descontinuidades abruptas e geometria euclidiana perfeita. Usados para testes unitários e validação de corner cases.

**c. Dados Reais (Odometria Inercial / Dataset MAGF-ID):**

Neste projeto, os dados reais derivam de sistemas de navegação baseados em IMU (Inertial Measurement Unit), utilizando o dataset MAGF-ID. Diferente de uma varredura direta (LiDAR), a topografia aqui é inferida através da reconstrução de trajetória e atitude do agente:

- Fonte: Acelerômetros, giroscópios e magnetômetros.
- Processo: A matriz $$H$$ é gerada pela projeção da pose estimada $$\left(x_t,y_t,z_t\right)$$ no grid 2D.
- Características Críticas:
    - Deriva (Drift): Ao contrário do erro local do LiDAR, o erro do IMU é cumulativo. Pequenos desvios na estimativa de inclinação ($$\theta$$) podem gerar grandes discrepâncias de altura ($$\Delta z$$) ao longo de trajetos longos.
    - Suavidade vs. Ruído Mecânico: O IMU capta vibrações do chassi que não correspondem ao relevo. O mapa resultante exige filtragem para distinguir o que é "rampa" do que é "trepidação".
    - Esparsidade: Dados de odometria geram "trilhas". Para obter a matriz $$H^{m\times n}$$ completa, é necessário aplicar técnicas de interpolação espacial, como Kriging ou Inverse Distance Weighting, nas áreas não visitadas, ou limitar o grafo apenas às regiões navegadas.

Importante:
Ao usar dados de IMU para gerar o terreno H, precisamos ter cuidado redobrado com o eixo $$Z$$.

- O problema da "Gravidade Fantasma": Sensores IMU de baixo custo muitas vezes confundem aceleração lateral (curva) com inclinação (gravidade), o que pode criar "morros virtuais" no mapa onde o terreno é, na verdade, plano.

- Consistência Global: Se o dataset MAGF-ID consistir em vários loops sobre a mesma área, a altura $$z$$ no início e no fim do loop pode não bater devido ao drift (o robô "acha" que subiu ou desceu, mas voltou ao mesmo lugar).

- (Pré-processamento): Talvez seja necessário uma etapa explícita de "nivelamento" ou correção de loop-closure se o mapa parecer inclinado artificialmente.

---

## 2. Topologia e Vizinhança (Grafo de Grade)

Uma vez definida a matriz $$H$$, o passo seguinte é convertê-la em uma estrutura topológica navegável: um grafo $$G=(V,E)$$. Esta conversão determina os "graus de liberdade" cinemáticos do agente dentro da simulação discreta.

### 2.1. Definição de Vértices (V)

Nem toda célula da matriz torna-se um nó do grafo. Em mapas gerados por odometria (esparsos), distinguimos células válidas de células vazias. O conjunto de vértices $$V$$ é definido apenas pelas células contendo dados de elevação válidos:

$$
V={v_{i,j}\mid H_{i,j}\neq NaN}
$$

Isso implica que o grafo resultante não é necessariamente uma grade regular perfeita; ele pode conter "ilhas", "túneis" ou componentes desconexos, dependendo da qualidade da trajetória de entrada.

### 2.2. Definição de Arestas ($E$) e Conectividade

As arestas definem as transições permitidas. Existem duas topologias padrão para grids quadrados:

**a. 4-Vizinhos (Von Neumann):**

Conecta apenas células ortogonais (Cima, Baixo, Esquerda, Direita).
- Métrica induzida: Distância Manhattan ($L_1$).
- Problema: O robô não pode fazer curvas suaves; para ir à diagonal, precisa fazer um "zigue-zague" (staircasing), superestimando a distância real em fator de $\sqrt{2} \approx 1.41$ no pior caso.

**b. 8-Vizinhos (Moore) - Adotado neste Projeto**

Conecta células ortogonais e diagonais.

$$
N_8(i,j) = \{ (k,l) \in V \mid \max(|k-i|, |l-j|) = 1 \}
$$

- Métrica induzida: Distância Chebyshev ($L_\infty$), aproximando-se da Euclidiana ($L_2$) quando os custos são ponderados.
- Vantagem Robótica: Permite aproximações de movimentos de 45°, essenciais para suavizar trajetórias de veículos reais.

### 2.3. Condição de Existência da Aresta

Uma aresta $e = (u, v)$ só existe no conjunto $E$ se ambas as condições forem atendidas: 

1. Adjacência Geométrica: $v$ está no conjunto $N_8(u)$. 
2. Validade dos Dados: Tanto $H_u$ quanto $H_v$ são números reais válidos (não são NaN).

> Nota sobre "Cutting Corners": Em uma topologia de 8-vizinhos, mover-se na diagonal entre dois obstáculos (ou paredes) pode ser geometricamente possível no grafo, mas fisicamente impossível para um robô com largura > 0. Para este estudo algorítmico, assumimos o robô como um ponto material. Em um sistema de produção, seria necessário inflar os obstáculos (Costmap Inflation) antes de gerar o grafo.

### 2.4. Resumo da Estrutura

O grafo resultante $G$ possui as seguintes propriedades:
- Não-direcionado (inicialmente): Se posso ir de A para B, a aresta existe de B para A.
- Grau Máximo: 8 (em áreas abertas).
- Grau Mínimo: 1 (em pontas de corredores sem saída/dead-ends da odometria).

---

## 3. Geometria do Movimento (Distância 3D)



---

## 4. Análise de Inclinação e Transversalidade

Antes de calcular o custo, avaliamos a viabilidade física da aresta. Um agente possui limites mecânicos (torque, atrito, centro de massa).

Definimos a inclinação $$\(\Theta\)$$ (em radianos) da aresta como:

$$
\Theta=arctan(dxy\mid\Delta h\mid)
$$

### 4.1. Corte de Transversalidade (Hard Constraint)

Se a inclinação excede o limite crítico do robô ($$\(\Theta_{max}\)$$), a aresta é considerada intransponível (obstáculo):

$$
se \(\Theta>\Theta_{max}\)⟹w(u,v)=∞
$$

Isso remove efetivamente a aresta do conjunto $$\(E\)$$, criando barreiras naturais baseadas na topografia.

---

## 5. Função de Custo Composta

Se a aresta é transponível $$(\Theta\leq\Theta_{max})$$, o custo w(u,v) é modelado como uma função do esforço estimado. Utilizamos uma abordagem híbrida que combina distância e penalidade por esforço.

$$
w(u,v)=d_{3D}(u,v)\cdot(1+K_p\cdot P(\Theta))
$$

Onde:

    - $$d_{3D}$$: O custo base (caminho mais curto geométrico).

    - $$K_p$$: Coeficiente de penalidade (ajuste de engenharia, "peso do esforço").

    - $$P(\Theta)$$: Função de penalidade baseada na inclinação.

### 5.1 Modelos de Penalidade $$(P(\Theta))$$

Existem duas abordagens implementadas para o cálculo da penalidade:

#### A. Modelo Linear (Simplificado)

Assume que o custo aumenta proporcionalmente à inclinação. Bom para validação algorítmica simples.

$$
P(\Theta)=d_{xy}\mid\Delta h\mid=tan(\Theta)
$$

#### B. Modelo Exponencial (Realista)

Simula o aumento drástico de consumo energético ou risco de deslizamento conforme a inclinação se aproxima do limite crítico.

$$
P(\Theta)=e^{\beta\cdot\Theta_{max}\cdot\Theta^{-1}}
$$

Onde $$\(\beta\)$$ controla a "agressividade" da curva de custo.

---

## 6. Assimetria de Custo (Direcionalidade)

Em cenários de alta fidelidade, o grafo é direcionado ($$w(u,v)\neq w(v,u)$$).

- Subida ($$\Delta h>0$$): Custo alto (trabalho contra a gravidade).

- Descida ($$\Delta h<0$$):

    - Caso Simples: Custo reduzido (ajuda da gravidade).

    - Caso Robótico: Custo moderado (necessidade de frenagem ativa para não perder controle).

Para este projeto, adotamos uma Penalização Simétrica Conservadora (assumindo que descer uma ladeira íngreme é tão perigoso/custoso quanto subi-la), garantindo robustez e simplificando a heurística para o A*.

---

## 7. Propriedades Formais do Grafo Resultante

O processo de modelagem resulta em um grafo $$G=(V,E,w)$$ com as seguintes garantias para o algoritmo de Dijkstra:

1. Não-Negatividade: $$w(e)\geq 0$$ para todo $$e\in E$$. Isso é garantido pois $$d_{3D}>0$$ e os termos de penalidade são positivos.

2. Conectividade Variável: A matriz densa torna-se um grafo esparso dependendo dos obstáculos ($$\theta>\theta_{max}$$).

3. Determinismo: O mesmo input $$H$$ sempre gera o mesmo grafo $$G$$.

---

## 8. Considerações de Implementação

Ao construir a matriz de adjacência ou lista de adjacência:

- Normalização: É recomendável normalizar as altitudes para a escala do grid se H vier de dados reais (ex: GeoTIFF), evitando que $$\Delta h$$ domine completamente $$d_{xy}$$.

- NaN Handling: Células com valor $$\text{NaN}$$ no input representam buracos ou áreas desconhecidas e são isoladas (grau 0).

---
Esta modelagem física serve como a "verdade" (ground truth) para os custos de movimentação. No próximo módulo (Pathfinding Algorithms), utilizaremos este grafo para comparar a eficiência de exploração do Dijkstra versus a busca direcionada do A*, onde a escolha da heurística $$h(n)$$ precisará ser consistente com a função de custo $$w(u,v)$$ aqui definida.

---

## 9. Preparação para os Experimentos

Os arquivos em `data/` incluem:

- terrenos sintéticos simples para validação;
- terrenos mais variados para análise visual.

Esses arquivos serão usados no documento:

- `05-experiments-and-results.md`

e no notebook:

- `01_exploratory_terrain_and_paths.ipynb`.

---

## 10. Continuidade com A\*

Esta modelagem será reutilizada no próximo módulo (A\*), portanto, a qualidade desta fase impacta diretamente o módulo seguinte.

