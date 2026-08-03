# Análise de Catálogo Netflix usando LLM

## Data Analyst LLM

📥 **Baixe a apresentação estilo Power BI em HTML:**
[Clique aqui para acessar o arquivo](https://drive.google.com/file/d/1yJHDrcMNhhZF1w3DsOvnSw6tNXjK1y9n/view?usp=sharing)

📥 **Baixe a apresentação executiva em HTML:**
[Clique aqui para acessar o arquivo](https://drive.google.com/file/d/1mi80BCFQZYXLAKGaq1eFTtW1X1nQgszA/view?usp=sharing)

## Problema de Negócio

### Contexto da empresa

**Stakeholder:** CEO da Netflix — visão executiva, decisão estratégica, não operacional. Isso significa que o output final precisa ser enxuto e direcionado a decisão, não um relatório exploratório extenso.

**Dor principal:** Entender quais filmes performaram bem — cruzando dois eixos de sucesso:

- Sucesso comercial/financeiro → receita, ROI (`revenue`, `budget`)
- Sucesso de captura/engajamento de audiência → alcance e recepção do público (`popularity`, `vote_count`, `vote_average`)

Isso é importante porque um filme pode "capturar audiência" (muita gente vê/avalia) sem necessariamente ser o mais lucrativo, e vice-versa — vamos precisar segmentar os filmes nesses dois eixos (ex: uma matriz tipo "sucesso de bilheteria x sucesso de audiência") em vez de tratar "sucesso" como um número único.

**Decisão a embasar:** Você mencionou "tomar uma decisão com base nos dados" — isso ainda está genérico. Como é uma análise para o CEO, geralmente esse tipo de decisão cai em uma (ou mais) dessas categorias:

- Onde investir mais (quais gêneros/países/diretores repetir)
- Onde cortar/reduzir investimento (o que não performou)
- Que tipo de conteúdo priorizar no catálogo/produção futura

## Premissas da análise

---

## Estratégia da solução

O método **Fato-Dimensão** foi usado para desenvolver a análise de dados.

### Passo 1: Resumir o contexto em uma pergunta aberta

As perguntas abertas são um tipo de demanda muito comum em análise de dados nas quais a demanda possui **N possíveis soluções** e cabe ao analista de dados avaliar as possibilidades e escolher a alternativa com o maior retorno e o menor esforço possível.

Para essa análise, foi definida a seguinte pergunta aberta:

> **Quais são os melhores filmes? Melhores diretores? Sobre o faturamento por filme, a empresa está negativa ou positiva?**

### Passo 2: Transformar pergunta aberta em fechada

As perguntas fechadas são um tipo de demanda muito comum na área de análise de dados. Essa demanda contém todos os detalhes da análise de dados e direciona o analista exatamente para o que precisa ser feito. Geralmente, a pergunta fechada é a escolha de uma solução entre todas as alternativas possíveis, feita por um profissional mais sênior da área.

Para essa análise, foi definida a seguinte pergunta fechada:

> **Pergunta Fechada:** Faça uma análise exploratória de cada aspecto que abrange os filmes do catálogo da empresa, diretores, faturamento por filme, a lei de Pareto em relação a filmes, e quais são os nichos que estão sendo negligenciados.

### Passo 3: Definição da Coluna Fato

O **Fato** é a coluna de interesse que representa o ponto focal da análise.

`revenue` — receita gerada pelo filme (em dólares). É a principal métrica de sucesso financeiro. Como a análise também cruza com engajamento, consideramos `popularity`, `vote_count` e `vote_average` como dimensões complementares para a matriz de desempenho.

### Passo 4: Identificação das Dimensões

#### Entendimento da Base de Dados

**📋 Overview da estrutura (16.000 filmes × 18 colunas)**

| Coluna | Tipo | O que representa | Significado de negócio |
| :--- | :--- | :--- | :--- |
| `show_id` | int | ID único do título | Chave primária, sem valor analítico direto |
| `type` | texto | Tipo de conteúdo (100% "Movie") | Sem variância — não discrimina nada nesta base |
| `title` | texto | Nome do filme | Identificação para relatórios/rankings |
| `director` | texto | Diretor(es) | Permite avaliar quais diretores entregam sucesso consistente (financeiro e de audiência) |
| `cast` | texto (lista) | Elenco principal | Permite avaliar "poder de atração" de atores — quem puxa audiência/receita |
| `country` | texto (lista) | País(es) de produção | Entender origem geográfica do conteúdo de sucesso — relevante para decisões de produção regional |
| `date_added` | data | Data que o título "entrou" na base/catálogo | Proxy de disponibilidade — permite análise de tendência temporal de catálogo |
| `release_year` | int | Ano de lançamento do filme | Separar "quando foi feito" de "quando entrou no catálogo" — útil para ver se filmes antigos ainda performam |
| `rating` | float (0-10) | Classificação numérica — na prática é idêntica a `vote_average` | Provavelmente redundante; preciso confirmar/depurar |
| `duration` | float | Duração em minutos | 100% nula — inutilizável |
| `genres` | texto (lista) | Gênero(s) do filme | Principal variável para segmentar "que tipo de conteúdo funciona" |
| `language` | texto | Idioma original | Entender se conteúdo em inglês domina ou se há sucesso multilíngue |
| `description` | texto | Sinopse | Só serve para NLP (não é foco aqui) |
| `popularity` | float | Score de popularidade (métrica tipo TMDB) | Proxy de "captura de audiência" — quanto o filme gerou de atenção/buzz |
| `vote_count` | int | Número de avaliações recebidas | Segundo proxy de alcance — quantas pessoas se engajaram o suficiente para avaliar |
| `vote_average` | float | Nota média das avaliações | Proxy de qualidade percebida, não de alcance |
| `budget` | int ($) | Orçamento de produção | Custo — insumo para calcular retorno |
| `revenue` | int ($) | Receita gerada | Proxy de sucesso financeiro/comercial |

---

### Passo 5: Hipóteses Analíticas

**H1 — Nem todos os gêneros geram receita proporcional ao volume de filmes produzidos**

Lógica: Gêneros como Ação/Aventura tendem a ter orçamentos maiores (efeitos especiais, elenco caro) e por isso podem gerar receita média mais alta, enquanto Drama/Comédia podem ser mais numerosos mas com retorno médio menor.
Como testar: Explodir `genres` (split por vírgula), agrupar por gênero e calcular receita média/mediana (filtrando `revenue > 0`), comparando com a contagem de filmes por gênero.

**H2 — Gêneros com maior popularidade não são necessariamente os mais bem avaliados**

Lógica: Ação/franquias podem gerar muito buzz (`popularity`, `vote_count`) mas nota mediana (`vote_average`) mais baixa que gêneros de nicho como Documentário ou Drama, que têm menos volume mas público mais "engajado com qualidade".
Como testar: Agrupar por gênero e comparar `popularity`/`vote_count` médios vs `vote_average` médio — ver se a correlação entre os dois grupos de métricas é fraca ou até negativa.

**💰 Eixo Financeiro**

**H3 — Orçamento maior não garante receita maior (retorno decrescente)**

Lógica: Existe um patamar em que aumentar o orçamento deixa de trazer ganho proporcional de receita — ou seja, ROI cai para blockbusters muito caros.
Como testar: No subconjunto com `budget > 0` e `revenue > 0`, calcular correlação entre `budget` e `revenue`, e também `revenue/budget` (ROI) por faixa de orçamento (quartis). Se ROI cair nos quartis mais altos, a hipótese se confirma.

**H4 — Um pequeno grupo de diretores/atores concentra a maior parte do sucesso financeiro**

Lógica: Segue o princípio de Pareto — poucos nomes "puxam" a maior parte da receita, sugerindo que aposta em nomes recorrentes reduz risco.
Como testar: Explodir `cast` e `director`, somar `revenue` por pessoa, ordenar decrescente e calcular % acumulado de receita pelos top 10/20% de nomes.

**H5 — País/idioma de produção influencia o teto de receita**

Lógica: Filmes em inglês / produzidos nos EUA podem ter teto de receita muito mais alto por causa do alcance de distribuição global, enquanto produções locais (outros idiomas) têm teto mais baixo mesmo quando bem avaliadas.
Como testar: Agrupar por `language` (e/ou `country` mais frequente na lista) e comparar `revenue` médio/máximo, com `vote_average` médio ao lado para ver se a diferença é de qualidade ou só de mercado/distribuição.

**👀 Eixo Captura de Audiência**

**H6 — Volume de avaliações (vote_count) é mais estável que popularidade instantânea**

Lógica: `popularity` pode ser um score volátil (picos de buzz recente), enquanto `vote_count` acumula engajamento ao longo do tempo — bons indicadores de "sucesso duradouro" vs "sucesso passageiro" podem divergir.
Como testar: Comparar o ranking de top 20 filmes por `popularity` vs top 20 por `vote_count` — quanto menor a sobreposição, mais essa hipótese se sustenta.

**H7 — Filmes mais antigos ainda concentram grande parte do engajamento (efeito "clássico")**

Lógica: Filmes de anos passados podem ter `vote_count` alto acumulado mesmo sem estarem mais "no radar" via `popularity`, sugerindo que catálogo antigo ainda sustenta audiência.
Como testar: Agrupar por `release_year`, comparar `vote_count` médio/total e `popularity` médio por ano — ver se anos mais antigos têm `vote_count` alto mas `popularity` baixo (relativo aos mais recentes).

**🔀 Eixo Cruzado (Financeiro x Audiência) — o núcleo da pergunta do CEO**

**H8 — Existem quatro perfis distintos de filme: "sucesso duplo", "hit de bilheteria sem prestígio", "joia escondida" e "fracasso duplo"**

Lógica: Sucesso financeiro e captura de audiência/qualidade não são a mesma coisa — um filme pode ter receita alta e nota baixa (blockbuster "vazio"), ou nota alta e receita baixa (cult hit sem alcance).
Como testar: Criar uma matriz 2x2 usando medianas como corte: eixo X = `revenue` (ou ROI), eixo Y = combinação de `popularity`/`vote_average`. Classificar cada filme em um dos 4 quadrantes e contar quantos caem em cada um — dá para o CEO ver a distribuição do catálogo nesses perfis.

**H9 — Nota alta (vote_average) não é o principal preditor de receita**

Lógica: Se o objetivo é entender "o que gera sucesso comercial", a hipótese é que `popularity` e `vote_count` (proxies de alcance) têm correlação mais forte com `revenue` do que a qualidade percebida (`vote_average`) — ou seja, alcance > qualidade para prever bilheteria.
Como testar: Calcular matriz de correlação entre `revenue` e as três métricas (`popularity`, `vote_count`, `vote_average`) no subconjunto financeiro válido, comparando os coeficientes.

---

### Passo 6: Critérios de Priorização

- **Critério 1:** Dados disponíveis.
- **Critério 2:** Insights acionáveis.

### Passo 7: Priorização das Hipóteses Analíticas

### ✅ Validação das hipóteses com os dados

Validou hipóteses de receita e identificou mecanismos acionáveis prioritários.

#### ✅ Resultado da validação — Etapa 4

| # | Hipótese | Nível Acionável | Resultado | Veredito |
| :--- | :--- | :--- | :--- | :--- |
| H8 | Matriz financeiro x audiência (4 perfis) | Alto | 37,2% Sucesso Duplo, 37,2% Fracasso Duplo, 12,8% Bilheteria sem Buzz, 12,8% Joia Escondida | ✅ Confirmada — a base de fato se divide em perfis distintos; quase 1 em cada 4 filmes está "descasado" entre os dois eixos |
| H1 | Receita desproporcional por gênero | Alto | Adventure (R$265M méd.), Sci-Fi (R$253M), Animation (R$233M) no topo; Documentary (R$19M) e Drama (R$54M) na base, mesmo Drama tendo 1.662 filmes (o mais produzido) | ✅ Confirmada — gêneros mais produzidos não são os mais lucrativos |
| H4 | Concentração de receita em poucos diretores | Alto | Top 10% dos diretores (230 de 2.306) concentram 72% de toda a receita do subconjunto financeiro | ✅ Fortemente confirmada — efeito Pareto muito acentuado |
| H9 | Alcance prediz receita melhor que nota | Alto | Correlação com `revenue`: `vote_count` = 0,72 (forte), `popularity` = 0,23 (fraca) | ✅ Confirmada — alcance supera qualidade para prever receita |
| H3 | Orçamento maior = ROI decrescente | Alto | Correlação `budget-revenue` = 0,75 (forte), mas ROI mediano por quartil de orçamento: Q1=1,60 → Q2=1,23 → Q3=1,45 → Q4=2,34 | ❌ Refutada — não há retorno decrescente; os orçamentos mais altos têm o melhor ROI mediano. "Apostar grande" tende a compensar mais, não menos |
| H2 | Popularidade de gênero ≠ nota de gênero | Médio | Correlação `popularidade x nota` a nível de filme = 0,07 (praticamente nula). Ex: Animation tem popularidade alta E nota alta (6,74); Action tem popularidade alta mas nota mediana (6,07) | ✅ Confirmada — buzz e qualidade percebida são fenômenos quase independentes |
| H5 | País/idioma define teto de receita | Médio | Inglês domina em volume (2.645 filmes, R$126M médio), mas idiomas como chinês (zh) aparecem no topo em receita média — com amostra pequena (n=65), sujeita a outliers | ⚠️ Inconclusiva — precisa filtro de amostra mínima e checagem de outliers antes de virar recomendação |
| H7 | Filmes antigos sustentam engajamento (efeito clássico) | Médio | `vote_count` médio caiu nos últimos anos (de ~1.000 em 2016-19 para 260 em 2024), enquanto `popularity` subiu (de ~15 para 74) | ❌ Refutada como formulada — não é que filmes antigos "sustentam" engajamento; é que filmes recentes ainda não tiveram tempo de acumular avaliações (viés de acúmulo temporal). Isso é uma limitação metodológica, não um padrão de negócio real |
| H6 | `Popularity` é mais volátil que `vote_count` | Baixo | Sobreposição entre top 20 `popularity` e top 20 `vote_count` = 0 filmes | ✅ Confirmada — são métricas completamente diferentes, mas o achado é mais diagnóstico que acionável |

### 🚩 Achados críticos que mudam a leitura para o CEO

- A crença "gastar mais dá retorno decrescente" está errada nesta base — o oposto acontece. Isso é uma informação de alto impacto para decisão de investimento.
- Nota de qualidade (`vote_average`) quase não se relaciona com receita nem com popularidade. Se o objetivo é "sucesso comercial", perseguir nota alta não é a alavanca certa — perseguir alcance (`vote_count`) sim.
- 72% da receita em 10% dos diretores é o achado de maior potencial de ação direta: uma política de "apostar em nomes recorrentes" tem respaldo forte nos dados.
- H7 revelou um viés estrutural nos dados (filmes recentes com menos tempo para acumular votos) — importante sinalizar isso como limitação na análise final, para não distorcer conclusões sobre "queda de qualidade recente".

---

## Insights da análise

## Data Analyst LLM

📥 **Baixe a apresentação estilo Power BI em HTML:**
[Clique aqui para acessar o arquivo](https://drive.google.com/file/d/1yJHDrcMNhhZF1w3DsOvnSw6tNXjK1y9n/view?usp=sharing)

📥 **Baixe a apresentação executiva em HTML:**
[Clique aqui para acessar o arquivo](https://drive.google.com/file/d/1mi80BCFQZYXLAKGaq1eFTtW1X1nQgszA/view?usp=sharing)

---

## Próximos passos

Receber feedback do CEO para saber se o seu desejo foi atendido com base na análise aqui presente, ou se será feito um aprofundamento na análise.
