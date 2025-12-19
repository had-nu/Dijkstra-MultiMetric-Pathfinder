# Terrain Modeling and Cost Functions

Este documento detalha a metodologia de conversão de dados topográficos brutos em uma estrutura de grafo ponderado, adequada para algoritmos de caminho mínimo como Dijkstra e A*.  

A modelagem ultrapassa a simples conectividade de grade, integrando restrições físicas e custos energéticos para simular a navegação de um agente sobre um relevo não-uniforme.

---

## 1. Representação do Terreno

O terreno é discretizado como um Grid de Elevação (Digital Elevation Model - DEM), representado formalmente pela matriz $$\(H\)$$:

$$
H \in \mathbb{R}^{m \times n}
$$

Onde $$\(H_{i,j}\)$$ representa a cota (altitude) absoluta da célula nas coordenadas discretas $$\(i, j\)$$. Ou seja, na linha $$\(i\)$$ e coluna $$\(j\)$$. 

Propriedades do Espaço:

- Resolução $$\(\delta\)$$: A distância física entre os centros de duas células adjacentes (ex: 1 metro/pixel).
- Consistência: Assume-se que a resolução é uniforme nos eixos X e Y.

---

## 2. Topologia e Vizinhança

Para converter a matriz em grafo $$G=(V,E)$$, cada célula $$\((i,j)\)$$ é mapeada para um vértice $$\(v_{i,j}\)$$. A conectividade é definida pela topologia de Moore (8-vizinhos), essencial para permitir trajetórias mais orgânicas e evitar o efeito "Manhattan" (zigue-zague excessivo).

$$
N8(i,j)={(k,l):max(|k-i|,|l-j|)=1}
$$

Isso resulta em um grafo onde o grau máximo de qualquer vértice é $$8\(deg(v)\leq 8\)$$. 

---

## 3. Geometria do Movimento (Distância 3D)

Diferente de abordagens simplistas que consideram apenas a distância planar, este modelo calcula a Distância Euclidiana 3D real percorrida pelo agente.

Ao mover-se de $$\((i,j)$$ para $$\((k,l)$$, a distância planar base $$\(d_{xy}\)$$ é:

$$
d_{xy}(u,v) = \delta \cdot \sqrt{(k-i)^2 + (l-j)^2}
$$

A distância física real $$\(d_{3D}\)$$, considerando o desnível $$\(\Delta h = H_v - H_u\)$$, é dada por pitágoras em 3 dimensões:

$$
d_{3D}(u,v) = \sqrt{d_{xy}^2 + \Delta h^2}
$$

Nota: Utilizar $$\(d_{3D}\)$$ penaliza naturalmente caminhos íngremes simplesmente porque eles são mais longos do que a sua projeção no plano.

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

