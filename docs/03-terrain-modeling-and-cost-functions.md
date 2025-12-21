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

A matriz é preenchida pela projeção da trajetória $$T$$ no plano 2D. Para uma pose $$(x_t,y_t,z_t)$$, a célula correspondente ($$i_t$$,$$j_t$$) é atualizada:

$$
H_{i_t,j_t} \leftarrow f(H_{i_t,j_t}, z_t)
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

## 3. Geometria do Movimento (Distância Euclidiana 3D)

Diferente de abordagens clássicas de grid-world que consideram apenas a distância planar (Manhattan ou Euclidiana 2D), este projeto calcula a distância física real percorrida pelo agente no espaço tridimensional. Isso é fundamental para a Odometria Inercial, pois o erro do sensor acumula-se sobre a trajetória percorrida ($$d_{3D}$$), e não sobre sua sombra projetada no chão ($$d_{xy}$$).

### 3.1. Distância Planar Base ($$d_{xy}$$)

Seja uma aresta conectando o nó atual $u=(i,j)$ a um vizinho $v=(k,l)$. A distância horizontal entre os centros das células, considerando a resolução espacial $\delta$, é:

$$
d_{xy}(u,v) = \delta \cdot \sqrt{(k-i)^2 + (l-j)^2}
$$

Dada a topologia de 8-vizinhos (Moore), isso simplifica para dois casos discretos:
- Movimento Ortogonal: $d_{xy} = \delta$
- Movimento Diagonal: $d_{xy} = \delta \sqrt{2} \approx 1.414 \cdot \delta$

### 3.2. Diferença de Cota ($\Delta h$)

A variação de altura entre os nós é obtida diretamente da matriz de elevação:

$$
\Delta h = H_{k,l} - H_{i,j}
$$

### 3.3. Distância Real Percorrida ($d_{3D}$)

Aplicando o Teorema de Pitágoras em 3 dimensões, obtemos o comprimento físico da aresta. Este será o peso base ($w_{base}$) do grafo antes de aplicarmos penalidades de esforço:

$$
d_{3D}(u,v) = \sqrt{d_{xy}^2 + \Delta h^2}
$$

### 3.4. Impacto no Pathfinding

A utilização de $$d_{3D}$$ introduz uma penalização geométrica implícita:

- Caminho Mínimo Real: O algoritmo priorizará, *ceteris paribus*, terrenos planos em vez de terrenos acidentados, simplesmente porque a hipotenusa de uma rampa é mais longa que o cateto base.

- Consistência com a Física: Se $$\Delta h$$ for muito grande (um muro vertical), $$d_{3D}$$ aumenta significativamente, mas não o suficiente para impedir o movimento sozinho. Por isso, precisaremos das restrições de inclinação na próxima seção.

**Caso: "Rampa vs. Degrau" e Navegação em Escombros**

Contexto: Uma empresa especializada em robôs de resgate foi contratada pela Defesa Civil para implantar robôs de resgate em escombros. O robô (TinyML Rover) utiliza apenas odometria inercial para mapear o ambiente, pois a fumaça bloqueia câmeras e LiDARs.

O Problema: O robô está na posição *A* e precisa chegar em *B*. O sistema detectou duas rotas possíveis no grid reconstruído. O algoritmo precisa decidir qual aresta explorar primeiro baseando-se no custo físico real.

Dados de Entrada:

- Resolução ($\delta$): $0.5$ metros (cada célula tem $50 \times 50$ cm).
- Matriz de Elevação Local ($H$):

$$
H = \begin{bmatrix}
 0.0 & 0.0 & \mathbf{2.0} \\
 0.0 & 0.0 & 0.5 \\
 \mathbf{0.0} & 0.0 & 0.0
 \end{bmatrix}
 $$
 
 (Onde $H_{2,0}$ é a posição atual do robô $A$ e ele avalia vizinhos).
 
 Cenário 1: O "Atalho" Vertical (Obstáculo)
 O robô avalia mover-se para a célula $(0,2)$ (canto superior direito), onde há um escombro de $2.0m$ de altura.

 - Deslocamento Planar ($d_{xy}$): Digamos que a distância no grid seja 2 células (1.0m).
 - Desnível ($\Delta h$): $2.0m - 0.0m = 2.0m$.
 - Custo Real ($d_{3D}$):
 
 $$
 d_{3D} = \sqrt{1.0^2 + 2.0^2} = \sqrt{5} \approx \mathbf{2.23\,m}
 $$
 
 (Nota: O caminho mais que dobrou de tamanho devido à altura).
 
 Cenário 2: A Rampa SuaveO robô avalia mover-se para a célula $(1,2)$ (meio direito), onde a altura é $0.5m$.
 
 - Deslocamento Planar ($d_{xy}$): A mesma distância de grade (1.0m).
 Desnível ($\Delta h$): $0.5m - 0.0m = 0.5m$.
 - Custo Real ($d_{3D}$):
 
 $$
 d_{3D} = \sqrt{1.0^2 + 0.5^2} = \sqrt{1.25} \approx \mathbf{1.11\,m}
 $$
 
 > Conclusão do Algoritmo:
 > Embora visualmente no mapa 2D ("top-down") as células pareçam equidistantes, o cálculo da Geometria 3D revela que o Cenário 1 custa 100% mais energia/distância para ser transposto do que o Cenário 2.O algoritmo de Dijkstra, alimentado por esses pesos, naturalmente evitará o escombro alto, direcionando o robô para a rampa suave, mimetizando o comportamento de um operador humano experiente.

---

## 4. Análise de Inclinação e Transversalidade

Agora chegamos ao momento de "vida ou morte" para o algoritmo. Enquanto a Seção 3 calculava o esforço, a Seção 4 decide a possibilidade. Se não implementarmos isso, o Dijkstra pode encontrar um caminho "ótimo" que envolve subir uma parede de 90 graus só porque é o trajeto mais curto.

Antes de atribuir um custo monetário ou energético a uma aresta, é necessário determinar a sua viabilidade física. Robôs possuem limites mecânicos intransponíveis ditados pelo torque dos motores, coeficiente de atrito das rodas e centro de massa. Nesta etapa, o grafo sofre uma "poda" baseada em geometria, transformando arestas geometricamente existentes em conexões logicamente nulas.

### 4.1. Cálculo da Inclinação Local (θ)

Para cada aresta potencial conectando $$u$$ a $$v$$, calculamos o ângulo de inclinação $$\(\theta\)$$ em relação ao plano horizontal.

Dada a diferença de altura $$\(|\Delta h|\)$$ e a distância planar $$d_{xy}$$:

$$
\theta = arctan(d_{xy} \cdot |\Delta h|)
$$

Onde $$\(\theta \in \left[0, \frac{\pi}{2}\right)\)$$.

A inclinação é definida por:

$$
m = \frac{\Delta y}{\Delta x}
$$

onde:
- $\Delta y$ é o *rise*
- $\Delta x$ é o *run*

![Slope diagram](slope.png)


### 4.2. Critério de Transversalidade (Hard Constraint)

Definimos um ângulo crítico $$\theta_{max}$$ (ou $$θ_{crit}$$), que representa o limite operacional do agente. A função de transversalidade da aresta é binária:

$$
\text{Transponível}(u, v) =
\begin{cases}
\text{True},  & \text{se } \theta \le \theta_{\max} \\
\text{False}, & \text{se } \theta > \theta_{\max}
\end{cases}
$$

Se $$\text{Transponível}(u, v)$$ for *False*, o peso da aresta torna-se infinito $$(w = \infty)$$, ou, mais eficientemente, a aresta é removida da lista de adjacência.

> Nota: Em sistemas de odometria inercial, picos de ruído no eixo Z podem criar "agulhas" falsas no terreno que excedem $$\theta_{max}$$. É comum aplicar uma tolerância ($$\epsilon$$) ou um filtro de suavização antes deste passo para evitar que o robô fique preso em "paredes de ruído".

**Caso: Decisão de Segurança**

Contexto: Continuando a operação de resgate, o robô está diante das mesmas opções, mas agora o algoritmo possui as especificações técnicas do chassi:

- Limite de Tração ($$\theta_{max}$$): $$45^\circ$$ ($$\frac{\pi}{4}$$ rad). Acima disso, o robô capota para trás.
- Resolução ($$\delta$$): 1.0 metro (para simplificar o cálculo).

Revisão do Cenário 1: O "Atalho" Vertical (Escombros)

- Dados: $$\Delta h=2.0m$$, $$d_{xy}=1.0m$$.
- Cálculo do Ângulo:

$$
\theta = \arctan\!\left(\frac{2.0}{1.0}\right) \approx 63.4^\circ
$$

Verificação: $$63.4^\circ > 45^\circ$$.

Decisão: VIOLAÇÃO DE SEGURANÇA. A aresta é removida. Para o grafo, o caminho direto através do escombro não existe, mesmo sendo o caminho mais curto em metros.

Revisão do Cenário 2: A Rampa Suave

Dados: $$\Delta h=0.5m$$, $$d_{xy}=1.0m$$.

Cálculo do Ângulo:

$$
\theta = \arctan\!\left(\frac{0.5}{1.0}\right) \approx 26.5^\circ
$$

Verificação: $$26.5^\circ \leq 45^\circ$$.

Decisão: APROVADO. A aresta permanece no grafo e seu custo será calculado (na próxima seção).

Impacto na Navegação: Graças a isso, o algoritmo de Dijkstra nem sequer considerará escalar o escombro. Ele será forçado a contornar o obstáculo ou pegar a rampa, garantindo não apenas a otimização da rota, mas a sobrevivência do robô.

---

## 5. Função de Custo Composta (Custo de Esforço)

Se a Seção 3 definiu a distância e a Seção 4 a viabilidade, a Seção 5 define a preferência. Aqui transformamos o grafo de uma representação geométrica para uma representação de esforço. Para um sistema de odometria e robótica, a distância mais curta nem sempre é a melhor. Muitas vezes, um desvio longo pelo plano economiza mais bateria (e oferece menos risco de derrapagem) do que uma subida curta e íngreme.

Uma vez que a aresta $$(u,v)$$ foi validada como transponível $$(\theta\leq\theta_{max})$$, precisamos atribuir um peso numérico $$w(u,v)$$ que guiará a busca do algoritmo de Dijkstra.

O peso não representa apenas distância (metros), mas sim o Custo Generalizado de Travessia, que pode ser interpretado como consumo de energia, tempo ou risco acumulado.

### 5.1. A Equação Geral do Custo

Definimos o custo da aresta como a distância física $$d_{3D}$$ modulada por um fator de penalidade associado à inclinação:

$$
w(u,v)=d_{3D}(u,v)\cdot(1+\alpha\cdot P(\theta))
$$

Onde:

- $$d_{3D}$$: Custo base geométrico (calculado na Seção 3).
- $$\alpha$$ (Alpha): Fator de sensibilidade (parâmetro de calibração).
    - Se $$\alpha=0$$, o terreno é tratado como se fosse plano (apenas distância importa).
    - Se $$\alpha≫1$$, o robô evitará inclinações a todo custo.
- $$P(\theta)$$: Função de penalidade normalizada baseada no ângulo de inclinação.

### 5.2. Modelos de Penalidade $$(P(\theta))$$

Dependendo da física do robô e da natureza dos dados (IMU ruidoso vs. LiDAR limpo), adotamos estratégias diferentes:

#### A. Modelo Linear (Soft Penalty)

Adequado para robôs com torque sobrando ou terrenos onde a aderência é constante. O custo cresce proporcionalmente ao ângulo.

$$
P_{linear}(\theta)=\frac{\theta_{max}}{\theta}
$$

#### B. Modelo Exponencial (Hard Penalty - Realista)

Adequado para simular o comportamento real de motores elétricos (onde a corrente sobe exponencialmente perto do limite de torque) ou risco de deslizamento em terrenos soltos (cascalho/areia).

$$
P_{exp}(\theta)=\exp\left(\beta\cdot\frac{\theta_{max}}{\theta}\right)-1
$$

Onde $$\beta$$ controla a agressividade da curvatura do custo.

### 5.3. Tratamento de Ruído (Zona Morta)

Considerando que os dados provêm de Odometria Inercial (MAGF-ID), pequenas oscilações de $$\theta$$ (ex: 1∘ a 3∘) são frequentemente ruído de sensor ou vibração, e não relevo real.

Para evitar que o algoritmo zigue-zagueie tentando evitar "ruído", aplicamos uma Zona Morta ($$\theta_{min}$$):

$$
P(\theta)=0\quad\text{se }\theta<\theta_{min}\approx5^\circ
$$

Sem isso, o mapa inercial cheio de micro-tremidas faria o custo flutuar loucamente, e o caminho resultante seria errático.

**Caso: Batalha por Eficiência**

Contexto: O robô rejeitou a parede vertical. Agora, ele precisa escolher entre duas rotas válidas para chegar ao destino final. O parâmetro de calibração do sistema é $$\alpha=10.0$$ (o robô é pesado e odeia subir ladeiras).

Opção A: A Rampa Direta (Curta mas Íngreme)

- Dados: Distância $$d_{3D}=1.11m$$, Inclinação $$\theta=26.5^\circ$$ ($$\approx0.46$$ rad).
- Limite: $$\theta_{max}=45^\circ$$.
- Razão: $$\frac{\theta_{max}}{\theta}=\frac{45}{26.5}\approx0.59$$.
- Custo (Linear): 

$$
w_A=1.11\cdot(1+10\cdot0.59)=1.11\cdot(6.9)\approx7.66
$$

Opção B: O Contorno Longo (Longo mas Plano) 

Existe um caminho alternativo pelo chão plano que dá a volta no obstáculo.

- Dados: Distância $$d_{3D}=3.50m$$, Inclinação $$\theta\approx0^\circ$$.
- Penalidade: $$P(0)=0.$$
- Custo: 

$$
w_B=3.50\cdot(1+0)=3.50
$$

> Decisão do Algoritmo: 
> Embora a Opção A (1.11m) seja 3x mais curta fisicamente que a Opção B (3.50m), o custo energético calculado (7.66 vs 3.50) indica que subir a rampa é energeticamente proibitivo.

O Dijkstra selecionará a Opção B. O robô fará o caminho mais longo, economizando bateria e reduzindo o desgaste mecânico. Isso demonstra a "inteligência" emergente da função de custo bem modelada.

### 5.4. Garantia de Correção (Não-Negatividade)

Para que o Dijkstra funcione, é imperativo que $$w(u,v)≥0$$. A nossa formulação garante isso estruturalmente:

- $$d_{3D}\geq\delta>0$$ (sempre positivo).
- $$\alpha\geq0$$ e $$P(\theta)\geq0$$ (penalidades aditivas).

Logo, não existem arestas de peso negativo, impedindo ciclos infinitos de redução de custo.

---

## 6. Assimetria de Custo (Direcionalidade e Gravidade)

Em topografias acidentadas, o custo energético é fundamentalmente anisotrópico devido à ação da gravidade. Subir uma encosta exige trabalho contra o vetor gravitacional (aumento de energia potencial), enquanto descer pode ser auxiliado por ele, embora exija controle (frenagem).

Isso implica que o grafo gerado deve ser Direcionado: a aresta $$e_{uv}$$ (ida) possui um peso diferente da aresta $$e_{vu}$$ (volta).

### 6.1. Definição do Sentido ($$sgn(\Delta h)$$)

Para determinar o regime de movimento entre $$u$$ e $$v$$, observamos o sinal do desnível:

$$
\Delta h_{u\to v}=H_v-H_u
$$

- Subida ($$\Delta h>0$$): O motor precisa superar a resistência ao rolamento + componente da gravidade.
- Descida ($$\Delta h<0$$): A gravidade auxilia o movimento, mas pode exigir consumo energético para frenagem ativa (segurança).
- Plano ($$\Delta h\approx0$$): Apenas resistência ao rolamento.

### 6.2. Fatores de Penalidade Distintos

Refinamos a função de custo da Seção 5 dividindo o coeficiente $$\alpha$$ em dois componentes distintos:

$$
w(u,v)=d_{3D}\cdot(1+K_{dir}\cdot P(\theta))
$$

Onde $$K_{dir}$$ é selecionado condicionalmente:

$$
K_{dir}=\begin{cases}
\alpha_{subida}&\text{se }\Delta h>0\text{(Ex: 10.0)}\\
\alpha_{descida}&\text{se }\Delta h<0\text{(Ex: 2.0)}
\end{cases}
$$

Interpretação Física:

- Geralmente, $$\alpha_{subida}≫\alpha_{descida}$$.
- Mesmo na descida, mantemos um $$\alpha_{descida}>0$$ (pequeno) ao invés de zero ou negativo. Isso simula a necessidade de cautela e evita que o robô se jogue "morro abaixo" descontroladamente.

### 6.3. Restrição Algorítmica (Pesos Negativos)

É tentador modelar a descida com custos negativos (recuperação de energia/freio regenerativo). No entanto, o algoritmo de Dijkstra não suporta arestas com pesos negativos, pois isso invalida a propriedade de "ganância" (greedy) da exploração.

Para garantir a estabilidade matemática:

$$
\forall(u,v),w(u,v)\geq d_{3D}>0
$$

Mesmo que a descida seja "fácil", ela nunca deve custar menos que a própria distância física percorrida.

**Caso: Ida e Volta**

Contexto: Vamos supor que, ao invés de dar a volta, como na seção anterior, o robô subiu a rampa (opção A) e agora precisa voltar para a base. Vamos comparar o custo de subir vs. descer a mesma rampa de 26.5∘.

Parâmetros:

- $$d_{3D}=1.11m$$
- $$P(θ)≈0.59$$ (fator de inclinação calculado na seção anterior).
- $$α_{subida}=10.0$$ (Muito custoso).
- $$α_{descida}=1.0$$ (Pouco custoso, apenas atrito).

Custo da Ida (Subida A→B):

$$
w_{A\to B}=1.11\cdot(1+10.0\cdot0.59) \approx 7.66
$$

Custo da Volta (Descida B→A):

$$
w_{B\to A}=1.11\cdot(1+1.0\cdot0.59) \approx 1.76
$$

> O caminho de volta é 4.3x "mais barato" para o algoritmo do que o caminho de ida. Isso cria um comportamento interessante: para ir até o alvo, o robô pode dar uma volta enorme pelo plano (para evitar a subida de custo 7.66). Mas para voltar, ele pode escolher descer a rampa direta (custo 1.76), criando rotas assimétricas típicas de trilhas reais.

---

## 7. Propriedades Formais do Grafo Resultante

Após aplicar as regras de discretização, geometria e custo, o terreno original é convertido em um grafo dirigido ponderado $$G=(V,E,w)$$. Isso significa que o objeto matemático construído ($$G$$) cumpre os pré-requisitos teóricos para que o algoritmo de Dijkstra funcione corretamente e termina a modelagem antes de entrarmos na implementação.

A análise das propriedades deste grafo é fundamental para prever o comportamento computacional dos algoritmos de busca.

### 7.1. Definição Formal

O grafo é definido pela tupla $$G=(V,E,w)$$ onde:

- $$V$$ (Vértices): O conjunto de todas as células visitadas/conhecidas:

$$
V={(i,j)\in Z^2\mid H_{i,j}\neq NaN}
$$

- $$E$$ (Arestas): O conjunto de transições válidas sob a restrição de inclinação:

$$
E={(u,v)\in V\times V\mid v\in N_8(u)\land\theta(u,v)\leq\theta_{max}}
$$

- $$w$$ (Pesos): A função de custo assimétrica definida na Seção 6.

$$
w:E\to R^+
$$

### 7.2. Propriedades Imprescindíveis para o Algoritmo

**A. Não-Negatividade (Condição de Dijkstra)**

Para que o algoritmo de Dijkstra garanta a otimalidade sem reavaliar nós fechados (propriedade greedy), não podem existir arestas negativas. Nossa modelagem garante:

$$
\forall e\in E, w(e)\geq d_{3D}\geq\delta>0
$$

Isso assegura que o custo do caminho é estritamente monotônico crescente, prevenindo loops infinitos e garantindo a convergência.

**B. Direcionalidade (Digrafo)**

Devido à física da gravidade (Seção 6), o grafo é Dirigido (Directed Graph). A matriz de adjacência resultante não é simétrica:

$$
w(u,v)\neq w(v,u)\quad\text{(exceto em terreno perfeitamente plano)}
$$

Isso implica que, ao implementar a busca, devemos iterar sobre os sucessores de u, e o caminho de volta requer um recálculo total, não sendo apenas o inverso do caminho de ida.

**C. Esparsidade e Conectividade (O Efeito "Arquipélago")

Diferente de grids sintéticos que são geralmente conexos, grafos derivados de odometria inercial frequentemente apresentam Componentes Desconexos. Se o robô mapeou a "Sala A", foi desligado e transportado para a "Sala B", a matriz $$H$$ conterá duas ilhas de dados válidos separadas por um mar de $$NaN$$.

- Consequência: Se o ponto de partida $$S$$ estiver na "Sala A" e o alvo $$T$$ na "Sala B", o algoritmo de Dijkstra explorará toda a componente conexa de $$A$$ e retornará "Caminho Inexistente". Isso é um comportamento esperado e correto, não um bug.

**D. Fator de Ramificação (Branching Factor)

O grau de saída máximo (\Delta_out) de qualquer vértice é limitado pela topologia de Moore:

$$
deg_out(v)\leq8
$$

Na prática, devido às bordas da trajetória e obstáculos ($$\theta>\theta_{max}$$), o grau médio é frequentemente $$3\leq\bar{deg}\leq6$$. Isso classifica o grafo como esparso, favorecendo implementações baseadas em Listas de Adjacência em vez de Matrizes de Adjacência para economia de memória em TinyML.

---

## 8. Considerações de Implementação

A transição das equações matemáticas para código performático exige escolhas arquiteturais específicas, especialmente considerando a natureza esparsa dos dados de odometria.

### 8.1. Estrutura de Dados: Matriz vs. Lista de Adjacência

Embora o terreno seja representado visualmente como uma matriz $$H$$ (Grid), o grafo $$G$$ não deve ser armazenado como uma Matriz de Adjacência $$(N\times N)$$, simplesmente por ser inviável.Para um mapa de 100×100 células (104 nós), a matriz de adjacência teria 108 entradas. Como cada nó tem no máximo 8 vizinhos, 99.9% dessa memória seria desperdiçada com zeros.

**Lista de Adjacência (Hash Map): Abordagem recomendada**
- Python: `Dict[Node, Dict[Neighbor, Weight]]`.
- C++ (TinyML): `std::vector` de structs ou mapas estáticos.
- Armazena apenas as transições válidas ($$\theta\leq\theta_{max}$$), economizando memória e acelerando a iteração dos vizinhos no algoritmo de Dijkstra.

### 8.2. Otimização via Vetorização e Máscaras

A construção do grafo é, computacionalmente, a etapa mais custosa antes da busca em si. Uma abordagem ingênua — iterar sobre cada célula $$(i,j)$$ com laços `for` aninhados para verificar seus 8 vizinhos — introduz um *overhead* proibitivo em Python, devido à natureza interpretada da linguagem. Para viabilizar experimentos rápidos com múltiplos mapas e parâmetros, adotamos uma estratégia de Vetorização (Array Operations) utilizando NumPy.

**A Técnica de Deslocamento de Matrizes (Array Shifting):**
Ao invés de calcular a diferença de altura $$\Delta h$$ vizinho por vizinho, processamos o grid inteiro de uma vez. Para calcular o custo de movimento para a "Direita", por exemplo, subtraímos a matriz original de uma versão dela mesma deslocada (shifted) em uma coluna. Isso nos permite calcular os custos de milhões de arestas simultaneamente, aproveitando as otimizações de baixo nível (C/Fortran) do NumPy.

**Aplicação de Restrições via Máscaras Booleanas:**
A validação de transversalidade (Seção 4) não utiliza condicionais `if/else` lentos. Em vez disso, calculamos a matriz de inclinações $$\theta$$ para todo o terreno e geramos uma *máscara booleana* onde $$\theta>\theta_{max}$$. Essa máscara é aplicada multiplicativamente ou via indexação direta para invalidar arestas em lote.

> Nota: Esta otimização é específica para a fase de prototipagem em Python. Na implementação final embarcada (TinyML/C++), a memória é escassa e não permite duplicar matrizes para vetorização; lá, a iteração baseada em ponteiros será a escolha mais eficiente.

### 8.3. Precisão Numérica e Tolerâncias

Ao lidar com *floats* derivados de sensores inerciais:

- Nunca verificar `if custo == 0`. Usar `if custo < epsilon`.
- O custo total do caminho é uma soma de *floats*. Em trajetos muito longos (> 1km), a precisão simples (float32) é suficiente para robótica, mas deve-se estar atento a erros de arredondamento se os custos das arestas tiverem ordens de magnitude muito díspares (ex: andar no plano vs. penalidade exponencial extrema).

---

## 9. Preparação para os Experimentos

A validação desta modelagem será conduzida através de experimentos comparativos, utilizando os dados armazenados na pasta `data/`.

### 9.1. Datasets de Entrada

O algoritmo consumirá dois tipos de artefatos gerados pelo notebook `01_exploratory_terrain_and_paths.ipynb`:

**A. Terrenos Sintéticos (data/synthetic/)**

- `ramp_perfect.npy` : Uma rampa lisa com inclinação constante. Usado para validar se o custo calculado bate com a fórmula teórica.
- `obstacle_course.npy` : Um corredor plano bloqueado por um muro alto. Usado para testar o "Hard Constraint" ($$\theta_{max}$$).

**B. Dados Reais MAGF-ID (data/processed/)**

- `magf_sample_01.npy` : Matriz esparsa reconstruída a partir de uma caminhada de 50 metros em ambiente de escritório (chão plano + escada).

Desafio: Contém ruído de vibração e células $$NaN$$. O algoritmo deve ser capaz de encontrar o caminho contornando os "buracos" da odometria.

### 9.2. Integração com o Próximo Módulo

Este documento define a classe `TerrainGraph` que servirá de input direto para o próximo módulo:

- Documento: `04-dijkstra-implementation.md`
- Objetivo: O algoritmo de Dijkstra não precisará saber o que é "altura" ou "inclinação". Ele receberá apenas o grafo abstrato $$G$$ com pesos $$w$$ definidos aqui.

Essa separação de responsabilidades (Decoupling) é importante, pois se decidirmos mudar a física do robô (ex: mudar de rodas para lagartas), alteramos apenas a função de custo neste módulo, sem tocar em uma linha de código do algoritmo de busca.

---

## 10. Continuidade com A\*

Esta modelagem será reutilizada no próximo módulo (A\*), portanto, a qualidade desta fase impacta diretamente o módulo seguinte.

