# Terrain Modeling and Cost Functions

Este documento descreve como o terreno é convertido em grafo ponderado e como são definidos os pesos (custos) das arestas.  
A modelagem é um ponto central do projeto, pois determina como o algoritmo de Dijkstra interpreta e optimiza deslocamentos sobre um relevo não-uniforme.

---

## 1. Representação do Terreno

Um terreno discreto é representado como uma matriz de alturas:

$$
H \in \mathbb{R}^{m \times n}
$$

onde:

- $$\(H_{i,j}\)$$ é a altitude da célula localizada na linha $$\(i\)$$ e coluna $$\(j\)$$;
- cada célula corresponde a uma posição física no espaço 2D;
- a escala vertical e horizontal é arbitrária, mas consistente.

Esta matriz pode ser:

- definida manualmente;
- gerada proceduralmente;
- derivada de dados reais (ex.: mapas digitais de elevação);
- simulada com valores sintéticos.

---

## 2. Discretização e Vizinhança

Cada célula $$\((i,j)\)$$ torna-se um vértice do grafo:

$$
v_{i,j} \in V
$$

As arestas conectam células adjacentes. Há duas convenções possíveis:

### **2.1. 4-vizinhos (Von Neumann)**

Conecta apenas células ortogonais:

$$
N_4(i,j) = \{(i+1,j),\,(i-1,j),\,(i,j+1),\,(i,j-1)\}
$$

Características:

- topologia mais simples;
- sem movimentos diagonais;
- custo geométrico mais fácil de interpretar.

### **2.2. 8-vizinhos (Moore)**

Conecta ortogonais e diagonais:

$$
N_8(i,j) = N_4(i,j) \cup \{(i+1,j+1),(i+1,j-1),(i-1,j+1),(i-1,j-1)\}
$$

Características:

- permite caminhos mais suaves;
- aproxima melhor o espaço contínuo;
- exige cuidado ao definir a distância plana.

Este projeto adota *N_8* por ser mais realista para navegação robótica.

---

## 3. Distância Plana

O custo base do movimento depende da geometria da vizinhança.

$$
d_{\text{plano}} =
\begin{cases}
1 & \text{movimento ortogonal} \\
\sqrt{2} & \text{movimento diagonal}
\end{cases}
$$

Esta definição aproxima a distância euclidiana entre centros de células adjacentes.

---

## 4. Inclinação Local e Desníveis

Ao mover-se de uma célula $$\((i,j)\)$$ para uma vizinha $$\((k,l)\)$$, a variação de altura local é:

$$
\Delta h = H_{k,l} - H_{i,j}
$$

Interpretação física:

- $$\(\Delta h > 0\)$$: subida;  
- $$\(\Delta h < 0\)$$: descida;  
- $$\(|\Delta h|\)$$: magnitude do desnível independentemente do sentido.

Robôs terrestres raramente possuem simetria perfeita entre consumo energético em subidas e descidas, mas para fins educacionais adotamos penalização simétrica.

---

## 5. Função de Custo

O custo total para mover-se da célula $$\((i,j)\)$$ para $$\((k,l)\)$$ é:

$$
w((i,j),(k,l)) = d_{\text{plano}} \cdot \left( 1 + \alpha \cdot |\Delta h| \right)
$$

Onde:

- $$\(d_{\text{plano}}\)$$ representa o custo básico geométrico;
- $$\(|\Delta h|\)$$ representa a dificuldade da transição vertical;
- $$\(\alpha \in \mathbb{R}_{\ge 0}\)$$ ajusta a sensibilidade à inclinação.

### **Interpretação de $$\(\alpha\)$$:**

- $$\(\alpha = 0\)$$: terreno plano → o problema vira “apenas distância”;  
- $$\(\alpha > 0\)$$: inclinação passa a influenciar fortemente o custo;  
- $$\(\alpha \gg 1\)$$: o algoritmo evita drasticamente regiões inclinadas.

$$\(\alpha\)$$ funciona como um parâmetro de engenharia.

---

## 6. Justificativa para Custos Não-Negativos

O algoritmo de Dijkstra **exige que todos os pesos sejam não-negativos**.  
A função acima garante isto porque:

- $$\(d_{\text{plano}} > 0\)$$;
- $$\(|\Delta h| \ge 0\)$$;
- $$\(\alpha \ge 0\)$$.

Isso preserva a monotonicidade das estimativas de custo e garante a correção formal do algoritmo.

---

## 7. Normalização Opcional (para terrenos reais)

Terrenos complexos podem ter:

- variações abruptas de altura;  
- ranges muito diferentes entre dimensões X/Y/Z;  
- gradientes extremamente amplos.

Para manter o custo numérico estável, pode-se usar uma normalização opcional:

$$
\Delta h_{\text{norm}} = \frac{\Delta h}{\max |H| }
$$

Ou usar uma versão delimitada por quantis.  
Esta normalização não altera a estrutura teórica, apenas facilita experimentação.

---

## 8. Construção do Grafo

O grafo final é definido como:

$$
G = (V, E, w)
$$

onde:

- $$\(V = \{ v_{i,j} \mid 0 \le i < m,\, 0 \le j < n \}\)$$
- $$\(E = \{ (v_{i,j}, v_{k,l}) \mid (k,l) \in N_8(i,j) \}\)$$
- $$\(w\)$$ segue a função definida acima.

Características:

- Grafo quase-regular (cada vértice tem até 8 vizinhos).
- Ponderado por propriedades físicas simples.
- Completamente determinístico.

---

## 9. Mapa de Custos e Interpretação

A função de custo induz uma paisagem de energia onde:

- regiões com desníveis acentuados tornam-se “montes de custo”;  
- vales e planícies tornam-se corredores de baixo custo;  
- movimentos diagonais são mais longos, mas nem sempre mais caros;  
- caminhos “visualmente curtos” podem se tornar energeticamente desfavoráveis.

A análise da trajetória final permite observar como a geometria e a topografia moldam a solução.

---

## 10. Relação com Planeamento Robótico

Esta modelagem é uma aproximação válida para:

- rovers terrestres;  
- AMRs industriais;  
- UAVs em navegação 2.5D;  
- robôs educativos que percorrem mapas discretos.

Embora simplificada, ela captura os elementos essenciais:

- deslocamento planar;
- variação de altitude;
- custo energético aproximado;
- busca por trajetórias fisicamente coerentes.

---

## 11. Preparação para os Experimentos

Os arquivos em `data/` incluem:

- terrenos sintéticos simples para validação;
- terrenos mais variados para análise visual.

Esses arquivos serão usados no documento:

- `05-experiments-and-results.md`

e no notebook:

- `01_exploratory_terrain_and_paths.ipynb`.

---

## 12. Continuidade com A\*

Esta modelagem será reutilizada no próximo módulo (A\*):

- mesma matriz de alturas;
- mesma função de custo;
- adição de heurísticas admissíveis;
- análise comparativa entre Dijkstra e A\*.

A qualidade desta fase impacta diretamente o módulo seguinte.

