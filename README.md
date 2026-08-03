# Análise de Entregas iFood usando LLM

## Data Analyst LLM

📥 **Baixe a apresentação estilo Power BI em HTML:**
[Clique aqui para acessar o arquivo](https://drive.google.com/file/d/1yJHDrcMNhhZF1w3DsOvnSw6tNXjK1y9n/view?usp=sharing)

📥 **Baixe a apresentação executiva em HTML:**
[Clique aqui para acessar o arquivo](https://drive.google.com/file/d/1mi80BCFQZYXLAKGaq1eFTtW1X1nQgszA/view?usp=sharing)

## Problema de Negócio

### Contexto da empresa

**Consumidor da análise:** CEO — ou seja, a análise precisa ser executiva: poucas métricas-chave, conclusões diretas, e recomendações acionáveis (não um relatório técnico extenso). Gráficos e números devem responder "o que fazer" mais do que "como calculamos".

**Dor / pergunta central:**

A operação de entrega está com atrasos, e não está claro se o gargalo é o restaurante (demora para preparar/liberar o pedido), o entregador (demora na coleta/deslocamento), ou o modelo de entrega/rota (design operacional — modal, tipo de rota, distância). Também não se sabe se isso se concentra em restaurantes específicos, horários de pico, ou regiões.

Isso se traduz em 3 perguntas operacionais:

- **Quem** — quais restaurantes puxam a operação para baixo?
- **Quando/onde** — existe padrão por horário, dia da semana ou geografia (proxy: distância, já que não há coluna de região explícita — vale confirmar se isso é uma limitação da base)?
- **Por quê** — a causa raiz é do restaurante, do entregador, ou do modelo/rota de entrega?

**Decisões que a análise precisa embasar:**

- Diagnóstico dos principais pontos de falha na operação.
- Métricas de comparação de performance entre restaurantes (um scorecard/ranking).
- Recomendações práticas baseadas em evidência para reduzir atrasos.

## Premissas da análise

---

## Estratégia da solução

O método **Fato-Dimensão** foi usado para desenvolver a análise de dados.

### Passo 1: Resumir o contexto em uma pergunta aberta

As perguntas abertas são um tipo de demanda muito comum em análise de dados nas quais a demanda possui **N possíveis soluções** e cabe ao analista de dados avaliar as possibilidades e escolher a alternativa com o maior retorno e o menor esforço possível.

Para essa análise, foi definida a seguinte pergunta aberta:

> **Como está sendo a velocidade de entrega dos pedidos do iFood, sendo eles entregues pela própria empresa, e sobre os gargalos referentes a atrasos nos pedidos, e saber quais são as causas?**

### Passo 2: Transformar pergunta aberta em fechada

As perguntas fechadas são um tipo de demanda muito comum na área de análise de dados. Essa demanda contém todos os detalhes da análise de dados e direciona o analista exatamente para o que precisa ser feito. Geralmente, a pergunta fechada é a escolha de uma solução entre todas as alternativas possíveis, feita por um profissional mais sênior da área.

Para essa análise, foi definida a seguinte pergunta fechada:

> **Pergunta Fechada:** Faça uma análise exploratória de cada aspecto que abrange a entrega dos produtos ao cliente, qual a margem de segurança das entregas? Quem normalmente mais faz atrasar as entregas do iFood? As empresas são eficientes no tempo? Ou o iFood não está sendo eficiente?

### Passo 3: Definição da Coluna Fato

O **Fato** é a coluna de interesse que representa o ponto focal da análise.

`flag_atraso_pedido` — indica se o pedido atrasou (booleano). É a principal métrica de resultado da operação de entregas, pois responde diretamente à pergunta do CEO sobre a experiência do cliente. Como a análise também cruza com diagnóstico de causa, consideramos `flag_atraso_restaurante`, `flag_atraso_entregador`, `atraso_pedido_seg` e `atraso_entregador_seg` como dimensões complementares para identificar a origem do problema.

### Passo 4: Identificação das Dimensões

#### Overview da estrutura

| # | Coluna | Tipo | O que representa | Significado de negócio |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `numero_pedido` | ID | Identificador único do pedido | Chave primária — 1 linha = 1 pedido |
| 2 | `hora_data_pedido` | datetime | Timestamp de criação do pedido | Permite análise por hora do dia, dia da semana, picos de demanda |
| 3 | `modelo_entrega` | categórica | Quem opera a entrega: OWN_DELIVERY (frota própria do iFood), MARKET_PLACE_DELIVERY (entregador do próprio restaurante), ON_DEMAND, HYBRID | Define quem é o responsável operacional pela entrega — essencial para separar responsabilidade do iFood vs. do restaurante |
| 4 | `tipo_entrega` | categórica | Só tem o valor ORDINARY | Sem variância — não discrimina nada, pode ser descartada |
| 5 | `tipo_rota` | categórica | SINGLE_PICK_SINGLE_DROP (uma coleta, uma entrega) vs SINGLE_PICK_MULTI_DROP (uma coleta, múltiplas entregas) | Rotas multi-drop tendem a ser mais lentas por pedido — é uma variável de desenho operacional |
| 6 | `modal` | categórica | Veículo do entregador: moto, bike, e-bike | Afeta velocidade/capacidade — moto domina (92%) |
| 7 | `tipo_loja` | categórica | Restaurante vs mercado | Perfis de preparo muito diferentes (mercado pode ter picking mais lento, restaurante tem tempo de cozinha) |
| 8 | `id_loja` | ID | Identificador da loja | Chave para o scorecard de performance por restaurante que o CEO pediu |
| 9 | `id_rota` | ID | Identificador da rota do entregador | Quase 1:1 com pedido (149.508 rotas p/ 153.604 pedidos) — múltiplos pedidos podem compartilhar rota em multi-drop |
| 10 | `tempo_real_ate_restaurante_seg` | numérica | Tempo real do entregador até chegar ao restaurante | Mede a etapa de deslocamento do entregador até a coleta |
| 11 | `tempo_estimado_ate_restaurante_seg` | numérica | Estimativa da plataforma para essa etapa | Benchmark de comparação (real vs. promessa) |
| 12 | `tempo_real_ate_coleta_seg` | numérica | Tempo real total até o entregador coletar o pedido no restaurante | Inclui deslocamento + tempo de espera no restaurante — é aqui que aparece o atraso de preparo |
| 13 | `tempo_entrega_real_seg` | numérica | Tempo real do pedido até a entrega no cliente | KPI-mestre de experiência do cliente |
| 14 | `tempo_entrega_estimado_seg` | numérica | Promessa de entrega total dada ao cliente | Benchmark para medir cumprimento de SLA |
| 15 | `flag_atraso_pedido` | booleana | Se o pedido atrasou (fim a fim) em relação à promessa | Métrica-chave de resultado — o que o cliente sente |
| 16 | `atraso_pedido_seg` | numérica | Quantos segundos de atraso no pedido | Magnitude do atraso final |
| 17 | `flag_atraso_entregador` | booleana | Se houve atraso atribuído ao entregador | Diagnóstico de causa — etapa de deslocamento/coleta |
| 18 | `atraso_entregador_seg` | numérica | Magnitude desse atraso | — |
| 19 | `flag_atraso_restaurante` | booleana | Se houve atraso atribuído ao restaurante | Diagnóstico de causa — etapa de preparo/liberação |
| 20 | `atraso_restaurante_seg` | numérica | Magnitude desse atraso | — |
| 21 | `distancia_restaurante_cliente_km` | numérica | Distância da entrega (loja→cliente) | Fator estrutural que explica parte do tempo de entrega |
| 22 | `distancia_ate_restaurante_km` | numérica | Distância do entregador até a loja no momento do aceite | Fator estrutural da etapa de coleta |

---

### Passo 5: Hipóteses Analíticas

**H1 — O atraso não é distribuído igualmente entre restaurantes; poucos "restaurantes-problema" concentram a maior parte do impacto (padrão 80/20).**

Lógica: já vimos que 76% dos atrasos finais vêm do componente restaurante. Se isso for um problema generalizado (todo restaurante atrasa um pouco), a solução é sistêmica. Se for concentrado em poucas lojas, a solução é pontual (intervenção direcionada).
Como testar: agrupar por `id_loja`, calcular taxa de atraso (`flag_atraso_pedido`) e atraso médio (`atraso_restaurante_seg`) por loja, ordenar decrescente e construir curva de Pareto (% acumulado de atrasos vs. % de lojas). Filtrar por volume mínimo de pedidos para evitar ruído estatístico em lojas com poucos pedidos.

**H2 — Restaurantes com maior volume de pedidos têm taxa de atraso pior (efeito de sobrecarga operacional na cozinha).**

Lógica: cozinhas com muitos pedidos simultâneos podem ter gargalo de preparo, gerando atraso do restaurante desproporcional ao volume.
Como testar: correlacionar volume de pedidos por loja (count por `id_loja`) com taxa de `flag_atraso_restaurante` por loja. Testar também dentro do mesmo dia — picos de volume horário por loja vs. atraso naquele horário.

**Bloco B — Quando/onde: padrões temporais e estruturais**

**H3 — O atraso é maior em horários de pico (almoço e jantar) do que em horários de baixa demanda.**

Lógica: picos de demanda pressionam tanto a cozinha (mais pedidos simultâneos) quanto a frota de entregadores (mais rotas disputando os mesmos entregadores disponíveis), então atraso deveria subir nesses períodos.
Como testar: extrair hora do dia de `hora_data_pedido`, agrupar taxa de atraso e atraso médio por faixa horária (ex: bins de 1h ou períodos: manhã/almoço/tarde/jantar/noite). Visualizar como série por hora.

**H4 — Existe padrão por dia da semana (ex: fins de semana atrasam mais que dias úteis).**

Lógica: fim de semana costuma ter volume de pedidos maior em delivery, o que pode saturar cozinhas e frota da mesma forma que H3, mas numa escala diária.
Como testar: extrair dia da semana de `hora_data_pedido`, comparar taxa de atraso e atraso médio entre dias. Atenção: a base cobre só 7 dias (6-12 abr/2020), então dá para comparar dias individualmente, mas não generalizar padrão sazonal com robustez estatística.

**H5 — Distância maior (loja→cliente e entregador→loja) está associada a mais atraso.**

Lógica: hipótese estrutural óbvia — trajetos mais longos aumentam a chance de atraso por trânsito, imprevistos, etc. Mas também testa se a plataforma já compensa isso corretamente na estimativa.
Como testar: correlação entre `distancia_restaurante_cliente_km` / `distancia_ate_restaurante_km` e `atraso_pedido_seg`. Segmentar por faixas de distância (curta/média/longa) e comparar taxa de atraso. Importante: comparar contra o tempo estimado também — se a estimativa já é proporcional à distância, o efeito no atraso pode ser pequeno mesmo com trajetos longos.

**Bloco C — Por quê: causa raiz (restaurante x entregador x modelo)**

**H6 — O modelo de entrega (`modelo_entrega`) influencia a taxa de atraso — MARKET_PLACE_DELIVERY (entregador do próprio restaurante) atrasa mais que OWN_DELIVERY (frota do iFood).**

Lógica: a frota própria do iFood tende a ter processos padronizados e otimização de roteirização; entregadores do restaurante podem ter menos eficiência logística, gerando mais atraso por entregador.
Como testar: comparar `flag_atraso_pedido`, `flag_atraso_entregador` e `atraso_pedido_seg` médio entre as categorias de `modelo_entrega`. Atenção ao desbalanceamento (OWN_DELIVERY é 91% da base) — usar médias e proporções, não só contagem absoluta.

**H7 — Rotas multi-drop (SINGLE_PICK_MULTI_DROP) têm atraso maior por pedido do que rotas single-drop.**

Lógica: ao entregar vários pedidos numa rota só, o entregador naturalmente demora mais para cada entrega individual, o que pode inflar o tempo de entrega real e gerar mais atraso — mesmo sendo mais eficiente em custo agregado.
Como testar: comparar `tempo_entrega_real_seg` e `flag_atraso_pedido` entre `tipo_rota`. Ver se `tempo_entrega_estimado_seg` já é ajustado para multi-drop (ou seja, se a estimativa é realista) ou se a promessa não reflete a complexidade da rota.

**H8 — O modal do entregador (moto vs. bike) afeta o atraso, especialmente combinado com distância.**

Lógica: bikes são mais lentas e mais sensíveis a distâncias longas; se restaurantes com entregas de bike tiverem distâncias parecidas às de moto, o atraso deveria ser sistematicamente maior nesse modal.
Como testar: comparar atraso médio e taxa de atraso por modal, controlando por faixa de distância (para não confundir "bike atrasa mais" com "bike é usada em regiões mais distantes", por exemplo).

**H9 — A estimativa de tempo (`tempo_entrega_estimado_seg`) está mal calibrada — sistematicamente otimista — o que gera atraso "artificial" mesmo com operação estável.**

Lógica: se o tempo real médio é consistentemente maior que o estimado, o problema não é só operacional, é também de política de SLA/promessa — a plataforma pode estar prometendo prazos irreais.
Como testar: calcular a diferença sistemática `tempo_entrega_real_seg - tempo_entrega_estimado_seg` (média e distribuição) para o total da base e por segmento (loja, modelo de entrega, horário). Se a média for consistentemente positiva mesmo em operação "normal" (baixo volume, curta distância), é evidência de estimativa mal calibrada, não de falha operacional pontual.

**H10 — O atraso do restaurante "silencioso" (que não vira atraso final, os 30% que vimos na etapa 2) mascara um problema de preparo mais amplo do que os KPIs de atraso sugerem.**

Lógica: se o buffer da estimativa absorve boa parte do atraso do restaurante, o KPI oficial de atraso pode estar subestimando o real problema de eficiência operacional das cozinhas — relevante porque o CEO quer diagnosticar causa raiz, não só o sintoma final.
Como testar: calcular a taxa de `flag_atraso_restaurante = True` independente de `flag_atraso_pedido`, e comparar o tamanho desse "buffer absorvido" (`atraso_restaurante_seg` quando `flag_atraso_pedido = False`) entre lojas — lojas que dependem muito desse buffer podem estar mais expostas a risco de atraso real se o volume aumentar.

---

### Passo 6: Critérios de Priorização

- **Critério 1:** Dados disponíveis.
- **Critério 2:** Insights acionáveis.

### Passo 7: Priorização das Hipóteses Analíticas

### ✅ Validação com os dados

**✅ H1 — CONFIRMADA (forte).** Os 20% de restaurantes com pior desempenho concentram 79% de todos os atrasos da base — um padrão de Pareto quase de manual. Isso confirma que o problema é de concentração, não generalizado, e justifica ação direcionada em vez de uma política igual para todas as lojas.

**❌ H2 — REFUTADA.** Correlação entre volume de pedidos por loja e taxa de atraso do restaurante é praticamente zero (0,008). Comparando por quartil de volume, a taxa de atraso fica estável (~36–40%) independente do tamanho da loja. Volume não explica o atraso — o problema está em processo/gestão da loja, não em sobrecarga.

**✅ H3 — CONFIRMADA.** Existe um padrão horário claro: picos de atraso no almoço (11h–12h, ~11%) e jantar (19h–20h, ~11–12%), com vale entre 14h–17h (~7%). Confirma que o problema se intensifica nos horários de maior demanda.

**⚠️ H4 — PARCIALMENTE CONFIRMADA.** Sexta (11,5%) e domingo (11,4%) têm as maiores taxas; sábado (8,3%) e segunda (8,0%) as menores. Padrão existe, mas com apenas 7 dias de dados não dá para afirmar que é sazonal e não uma variação pontual dessa semana específica — recomendo tratar como hipótese a confirmar com mais dados históricos.

**❌ H5 — REFUTADA.** Correlação entre distância e atraso é quase nula (0,037 e 0,06). Mesmo comparando por faixa de distância (curta a longa), a taxa de atraso varia pouco (9,7% a 10,9%). Distância não é o driver principal — reforça que o problema é mais de processo do que de logística geográfica pura.

**✅ H6 — CONFIRMADA, mas em direção invertida à hipótese original.** MARKET_PLACE_DELIVERY (entregador do restaurante) tem taxa de atraso final menor (6,8%) que OWN_DELIVERY (frota do iFood, 10,2%). E olhando o atraso do restaurante especificamente, MARKET_PLACE_DELIVERY também é menor (24,4% vs. 40,0% do OWN_DELIVERY). Um ponto de atenção: ON_DEMAND mostra 100% de flag de atraso do entregador — isso é estatisticamente estranho e merece investigação separada (pode ser particularidade de como esse modelo é medido, não necessariamente um problema operacional real).

**✅ H7 — CONFIRMADA.** Rotas multi-drop têm taxa de atraso maior (12,3% vs. 9,8% em single-drop) e tempo de entrega real bem maior (2.467s vs. 1.583s). Importante: a plataforma já estima mais tempo para multi-drop (3.255s vs. 2.269s) — ou seja, o sistema sabe que multi-drop é mais lento, mas mesmo assim atrasa proporcionalmente mais.

**✅ H8 — CONFIRMADA.** Bike tem taxa de atraso quase 50% maior que moto (14,9% vs. 9,5%), e essa diferença se mantém mesmo controlando por faixa de distância — ou seja, não é só porque bike pega rotas mais longas, é uma diferença estrutural do modal.

**❌ H9 — REFUTADA, e na direção oposta ao esperado.** Na média, o tempo real é 692 segundos MENOR que o tempo estimado — a plataforma não está sendo otimista demais, está dando uma folga generosa na promessa ao cliente. Isso muda a leitura: o problema não é uma estimativa mal calibrada de forma otimista; é que, quando o processo falha, ele estoura até uma margem de segurança já bem folgada. Isso é um achado relevante — o "buffer" do sistema é grande, o que torna os 10% de atrasos ainda mais um sinal de falha real (não de meta irreal).

**✅ H10 — CONFIRMADA.** Em média, 30% dos pedidos têm atraso do restaurante "absorvido" pelo buffer da estimativa (não vira atraso final). Mas isso varia muito por loja — algumas lojas têm 100% dos pedidos dependendo desse buffer para não atrasar oficialmente. Essas lojas estão operando "no limite": qualquer aumento de volume ou redução do buffer as levaria a atraso visível. É um sinal de risco escondido nos KPIs oficiais.

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
