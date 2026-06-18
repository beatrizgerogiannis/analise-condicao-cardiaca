# Triagem de Cardiopatia Infantil (UCMF) — Pipeline de Limpeza e Modelagem

Pipeline em Python para limpar os dados brutos da UCMF (fichas de triagem
cardiológica infantil) e comparar, via validação cruzada + holdout, quatro
cenários de modelagem que simulam diferentes níveis de informação clínica
disponível no momento da triagem.

## O que o script faz

1. Carrega e limpa o CSV bruto (`carregar_e_limpar_dados`, `tratar_datas_e_idade`,
   `aplicar_limites_clinicos`): padroniza texto, corrige datas com erro de século,
   recupera datas em formato serial do Excel, converte faixas de FC em média,
   trata sentinelas de "não medido", remove duplicatas e calcula o IMC.
2. Separa um holdout estratificado (20%) **antes** de qualquer validação cruzada,
   para que a estimativa final de desempenho não seja otimista.
3. Treina e compara 5 modelos (Regressão Logística, SVC, Random Forest,
   Gradient Boosting, XGBoost) em 4 cenários de variáveis disponíveis:

   | Cenário | Variáveis removidas | O que simula |
   |---|---|---|
   | A | nenhuma | modelo completo, com exame clínico |
   | B | SOPRO, PULSOS, B2, PPA | triagem sem exame físico especializado |
   | C | + MOTIVO1, MOTIVO2 | triagem sem motivo de encaminhamento |
   | D | + HDA 1, HDA2 | triagem sem nenhuma impressão clínica prévia |

4. Compara estatisticamente os cenários (teste t pareado no ROC-AUC fold a fold).
5. Calcula importância de variáveis por permutação no holdout (Random Forest)
   e gera um gráfico comparativo (`comparacao_importancias.png`).
6. Roda uma checagem exploratória adicional de holdout temporal (treino nos
   anos mais antigos, teste no ano mais recente), só para sinalizar possível
   drift de protocolo ao longo do tempo.

Para a lógica de cada etapa, decisões de limpeza e justificativas, veja
[`DOCUMENTACAO.md`](./DOCUMENTACAO.md).

## Requisitos

- Python 3.9+
- pandas, numpy, scipy, matplotlib
- scikit-learn
- xgboost

Instalação:

```bash
pip install pandas numpy scipy matplotlib scikit-learn xgboost
```

## Dados esperados

Um CSV em `../data/UCMF_raw.csv` (caminho configurável via `CSV_PATH`), contendo
pelo menos as colunas: `ID`, `Atendimento`, `DN`, `IDADE`, `SEXO`, `Peso`,
`Altura`, `PA SISTOLICA`, `PA DIASTOLICA`, `FC`, `SOPRO`, `PULSOS`, `B2`, `PPA`,
`MOTIVO1`, `MOTIVO2`, `HDA 1`, `HDA2`, `Convenio` e o alvo `NORMAL X ANORMAL`
(valores `Normal`/`Anormal`).

## Como executar

```bash
python pipeline_ucmf.py
```

O script imprime no console as tabelas de validação cruzada, os resultados de
holdout por cenário, as comparações estatísticas entre cenários e a checagem
de holdout temporal. Ao final, salva `comparacao_importancias.png` com as
top 15 variáveis por importância de permutação em cada cenário.

## Configuração principal

Os parâmetros mais relevantes estão no topo do arquivo:

| Constante | Para que serve |
|---|---|
| `CSV_PATH` | caminho do CSV de entrada |
| `TARGET` | nome da coluna-alvo |
| `RANDOM_STATE` | seed para reprodutibilidade |
| `N_SPLITS` / `N_REPEATS` | configuração do `RepeatedStratifiedKFold` |
| `HOLDOUT_SIZE` | fração separada para holdout (padrão 20%) |
| `TOP_N_CONVENIO` | nº de categorias de Convenio mantidas (resto vira "OUTRO") |
| `MODELO_PARA_IMPORTANCIA` | modelo usado para extrair importância por permutação |

Aumentar `N_REPEATS` torna o teste estatístico entre cenários mais robusto,
ao custo de mais tempo de execução.

## Limitações conhecidas

- O dataset cobre vários anos e parece ter mudado de protocolo ao longo do
  tempo (ex.: diferenciação de PPA "Não Calculado"); o split aleatório usado
  na análise principal é correto para estimar desempenho i.i.d., mas não
  captura esse possível drift — daí a checagem de holdout temporal ao final.
- Discrepâncias entre IDADE informada e idade calculada por data são apenas
  logadas, não corrigidas automaticamente; revisão manual é recomendada.
- Com poucos folds, o teste t pareado entre cenários tem baixo poder
  estatístico.