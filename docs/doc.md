# Documentação Técnica — Pipeline UCMF

Este documento detalha a lógica interna do script: cada etapa de limpeza,
as decisões de modelagem e os motivos por trás dos quatro cenários
comparados. Para uma visão rápida de uso, veja o [`README.md`](./README.md).

## 1. Contexto e objetivo

O script avalia o quanto um modelo consegue prever `NORMAL X ANORMAL`
(triagem cardiológica infantil) conforme se reduz progressivamente o tipo
de informação clínica disponível, simulando cenários de triagem com menos
recursos especializados:

- **Cenário A** — modelo de referência, com todos os achados de exame
  clínico (sopro, pulsos, B2, PPA).
- **Cenário B** — remove os achados de exame físico especializado, mas
  mantém o motivo de encaminhamento (impressão clínica de quem encaminhou
  o paciente).
- **Cenário C** — também remove o motivo de encaminhamento: tenta isolar o
  que dados vitais/demográficos/administrativos conseguem prever por si só.
- **Cenário D** — versão mais conservadora: também remove HDA 1/HDA2
  (sintomas já categorizados na anamnese pelo próprio serviço), isolando o
  caso de "nenhuma impressão clínica prévia", nem externa nem interna.

Essa progressão A→D ajuda a responder: "se eu não tiver acesso a um
cardiologista para auscultar o paciente, quanto da capacidade preditiva eu
perco — e quanto ainda resta de sinal em dados básicos?"

## 2. Pipeline de limpeza de dados

### 2.1 Carregamento e padronização de texto (`carregar_e_limpar_dados`)

- Faz `strip()` em todas as colunas de texto sem usar `.astype(str)`, para
  não transformar valores ausentes reais na string literal `"nan"`.
- Unifica variantes de grafia em `NORMAL X ANORMAL`, `SOPRO` e `PULSOS`
  (ex.: `"sistólico"` vs `"Sistólico"`).
- Trata `SEXO == "Indeterminado"` como dado ausente (`NaN`), não como uma
  terceira categoria clínica válida — decisão baseada na taxa de ausência
  muito mais alta de outras colunas clínicas dentro desse subgrupo, mais
  coerente com cadastro incompleto do que com prevalência real de sexo
  indeterminado/DSD.
- Em `PPA`, apenas o erro de planilha `"#VALUE!"` é convertido para `NaN`;
  `"Não Calculado"` é mantido como categoria válida própria, pois indica
  que a PA provavelmente não foi medida — informação diferente de
  simplesmente "não sei".
- Padroniza `Convenio` via `normalizar_texto_categoria` (preenche nulos
  antes de qualquer conversão de texto, depois aplica `strip()`/`upper()`).

### 2.2 Remoção de duplicatas (`remover_duplicatas`)

Remove linhas inteiramente idênticas em todas as colunas exceto `ID`,
chamada **logo após o carregamento**, antes de qualquer outra
transformação. Isso evita que cópias do mesmo registro acabem em treino e
holdout (ou em folds diferentes da validação cruzada) simultaneamente, o
que inflaria artificialmente ROC-AUC/F1/recall.

### 2.3 Datas e idade (`tratar_datas_e_idade`)

- `parse_data_com_correcao_seculo` interpreta datas no formato `dd/mm/yy` e
  corrige erro de século (ex.: `"27"` virando 2027 em vez de 1927) usando
  uma referência vinda do próprio dado:
  - `Atendimento` é comparado com `pd.Timestamp.now()` (correto e estável,
    pois uma data de atendimento já passada não volta a ser "futura").
  - `DN` (data de nascimento) é comparada com a data de atendimento **da
    própria linha**, não com "hoje" — ninguém nasce depois da própria
    consulta, e isso torna o resultado da limpeza independente de quando o
    script é executado.
  - Quando o parse textual falha, tenta interpretar o valor como número
    serial do Excel (origem `1899-12-30`), recuperando linhas que antes
    eram descartadas como "data inválida" só por estarem em outro formato.
- `verificar_discrepancia_idade` compara a IDADE informada na ficha com a
  idade calculada por `(Atendimento - DN)`; divergências acima de 5 anos
  são logadas como aviso, mas **não corrigidas automaticamente** — a
  decisão de qual fonte confiar é deixada para revisão manual.
- `IDADE` nula é preenchida com a idade calculada (`fillna`); linhas sem
  `IDADE` e sem datas válidas para calculá-la são descartadas.
- O ano de atendimento é guardado em uma coluna auxiliar interna
  (`_ANO_ATENDIMENTO`) antes de `Atendimento`/`DN`/`ID` serem descartadas —
  usada só na checagem exploratória de holdout temporal (seção 6), nunca
  como feature de modelo.

### 2.4 Limites clínicos e variáveis numéricas (`aplicar_limites_clinicos`)

- Zera valores fisiologicamente impossíveis: `IDADE` fora de [0, 19],
  `Peso`/`Altura` ≤ 0, `PA SISTOLICA` fora de (0, 300], `PA DIASTOLICA`
  fora de (0, 200], `FC` fora de [30, 250].
- `parse_valor_com_faixa` converte valores de FC registrados como faixa
  (ex.: `"92-100"`) para a média da faixa, em vez de deixá-los virar `NaN`
  num `pd.to_numeric` direto.
- Recalcula o IMC a partir de Peso/Altura já limpos (em vez de confiar no
  valor original do CSV) e descarta valores fora de [5, 45].
- Remove `Peso` e `Altura` do conjunto de variáveis preditoras
  (`COLS_REMOVER_SEMPRE`): ~35–45% dos registros trazem `0` como sentinela
  de "não medido", o que torna essas colunas brutas pouco confiáveis
  isoladamente. O IMC, já calculado, permanece como resumo derivado.

## 3. Definição dos cenários (variáveis removidas)

| Constante | Colunas | A partir de qual cenário é removida |
|---|---|---|
| `VARS_EXAME` | `SOPRO`, `PULSOS`, `B2`, `PPA` | B |
| `VARS_REFERENCIA` | `MOTIVO1`, `MOTIVO2` | C |
| `VARS_SINTOMAS_PREVIOS` | `HDA 1`, `HDA2` | D |

Os valores brutos de `PA SISTOLICA`/`PA DIASTOLICA`/`FC` permanecem em
todos os cenários (inclusive B), pois podem ser medidos em qualquer
triagem básica, sem depender de auscultação especializada. `PPA` é a
classificação percentilada derivada de PA + idade/sexo, por isso é tratada
como "rótulo de exame" e removida junto com os demais achados em B.

O Cenário C original (sem `VARS_EXAME` + `VARS_REFERENCIA`) foi mantido
sem a remoção de HDA 1/HDA2 para preservar comparabilidade com análises
anteriores; o Cenário D parte de C e remove também essas colunas, sendo a
versão mais estrita de "nenhuma impressão clínica prévia".

## 4. Pré-processamento e modelagem

### 4.1 Separação de holdout e categorias raras

- `train_test_split` estratificado (20%) é feito **antes** de qualquer CV
  ou seleção de modelo, e o holdout nunca participa da comparação de
  cenários/modelos — só da avaliação final.
- Categorias raras de `Convenio` são agrupadas em `"OUTRO"`
  (`ajustar_categorias_raras` / `aplicar_categorias_raras`), aprendendo o
  conjunto de categorias frequentes **somente no treino** e aplicando o
  mesmo conjunto ao holdout, evitando vazamento de dados.

### 4.2 Pipeline de transformação (`construir_pipeline`)

- **Numéricas**: `StandardScaler` aplicado **antes** do `KNNImputer`. Como
  o `StandardScaler` ignora `NaN` ao calcular média/desvio, o `KNNImputer`
  passa a calcular distância já no espaço padronizado — evitando que
  variáveis em escala maior (ex.: FC, 30–250) dominem a escolha de vizinhos
  sobre variáveis em escala menor (ex.: IDADE, IMC).
- **Categóricas**: imputação por `strategy='constant'` com rótulo explícito
  `NAO_INFORMADO` (em vez de `most_frequent`, que apagava o sinal de
  "não informado/não medido" em colunas com alta proporção de ausência,
  como HDA2 ~97%), seguida de `OneHotEncoder(handle_unknown='ignore',
  drop='first')`.
- `inferir_tipos_coluna` inclui um `assert` que garante que nenhuma coluna
  fica de fora de `num_cols`/`cat_cols` e seria descartada silenciosamente
  pelo `ColumnTransformer`.

### 4.3 Modelos comparados (`avaliar_modelos_cv`)

Regressão Logística, SVC, Random Forest e XGBoost usam `class_weight`
(ou `scale_pos_weight`, no caso do XGBoost) para lidar com desbalanceamento
de classes. `GradientBoostingClassifier` não tem `class_weight` nativo, por
isso recebe `sample_weight` balanceado manualmente, roteado ao classificador
final do pipeline via `_cross_validate_compat` (que tenta a API `params=`
do `cross_validate` mais recente e cai para `fit_params=` em versões mais
antigas do scikit-learn).

A validação cruzada usa `RepeatedStratifiedKFold` (`N_SPLITS` × `N_REPEATS`),
com métricas ROC-AUC, F1 e Recall (classe Anormal). `XGBClassifier` usa
`n_jobs=1` para não competir por threads com o paralelismo do
`cross_validate` (`N_JOBS_CV`).

Importante: o `pipeline` retornado por `avaliar_modelos_cv` **não está
ajustado** — `cross_validate` clona o estimador internamente por fold e
descarta os clones. O fit real do modelo final acontece em
`avaliar_holdout`.

## 5. Avaliação em holdout e importância de variáveis

- `avaliar_holdout` clona o pipeline, treina no conjunto de treino completo
  e avalia no holdout (nunca visto durante a CV), reportando ROC-AUC, F1 e
  Recall.
- `extrair_importancias` calcula importância por **permutação** (não por
  impureza/Gini) no holdout, usando o modelo definido em
  `MODELO_PARA_IMPORTANCIA` (Random Forest, por padrão). A permutação
  embaralha as colunas originais de entrada antes do `ColumnTransformer`,
  então o resultado já vem com os nomes reais das variáveis, sem precisar
  reconstruir nomes de dummies do one-hot — e evita o viés da importância
  de Gini a favor de variáveis contínuas/de alta cardinalidade.

## 6. Comparação estatística entre cenários (`comparar_cenarios`)

Aplica um teste t pareado no ROC-AUC fold a fold entre cada par de
cenários. Isso é válido porque todos os cenários compartilham o mesmo `y`
(mesmas linhas, mesmo alvo), logo o `RepeatedStratifiedKFold` com o mesmo
`RANDOM_STATE` produz exatamente os mesmos splits em todos eles.

**Atenção**: com poucos folds (`N_SPLITS` × `N_REPEATS`), esse teste tem
baixo poder estatístico — um p-valor não significativo pode refletir
apenas poucas observações, não ausência real de diferença entre cenários.
Aumentar `N_REPEATS` torna o teste mais robusto.

## 7. Checagem exploratória de holdout temporal (`checagem_holdout_temporal`)

Etapa **adicional**, fora da comparação A/B/C/D principal: treina nos anos
de atendimento mais antigos e avalia no ano mais recente (em vez do split
aleatório estratificado usado no resto do script), usando o conjunto de
variáveis do Cenário C e Random Forest.

Motivação: o split aleatório é correto para estimar desempenho i.i.d., mas
o dataset cobre vários anos e parece ter mudado de protocolo ao longo do
tempo (ex.: a própria diferenciação de PPA "Não Calculado"). Uma queda
grande de desempenho nesse split temporal, em comparação ao holdout
aleatório do Cenário C, sugere drift de protocolo/composição que o split
i.i.d. não captura. Essa checagem **não substitui nem altera** a análise
principal, e não participa da seleção/comparação de modelos.

## 8. Saídas geradas

- Console: tabelas de CV por cenário/modelo, resultados de holdout,
  comparações estatísticas par a par entre cenários, e o resultado da
  checagem de holdout temporal.
- `comparacao_importancias.png`: 4 painéis lado a lado (um por cenário) com
  as top 15 variáveis por importância de permutação no holdout. Todos os
  painéis compartilham a mesma escala de eixo X para permitir comparar a
  magnitude da importância entre cenários; o eixo acomoda valores
  levemente negativos (ruído de amostragem em variáveis sem sinal real).

## 9. Pontos de atenção para quem for estender o script

- Qualquer nova coluna categórica/numérica adicionada ao CSV precisa ser
  capturada por `inferir_tipos_coluna`; o `assert` interno avisa se algo
  ficaria de fora.
- Novas variáveis "de impressão clínica prévia" (análogas a MOTIVO1/2 ou
  HDA 1/2) devem ser adicionadas à lista de remoção do cenário apropriado,
  e não apenas descartadas globalmente — para preservar a lógica
  progressiva A→D.
- Se o CSV de origem mudar, vale reconferir os `value_counts()` das colunas
  categóricas tratadas defensivamente em `carregar_e_limpar_dados` (ex.:
  variantes de grafia em SOPRO/PULSOS), já que o comentário no código
  indica que algumas dessas variantes específicas podem não existir mais
  na versão mais recente da base.