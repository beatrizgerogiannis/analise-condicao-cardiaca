# Triagem de Cardiopatia Infantil (UCMF) — Pipeline de Limpeza e Modelagem 

Este é um projeto de **Mineração de Dados (Data Mining)** baseado na metodologia **KDD (Knowledge Discovery in Databases)**. O objetivo é extrair padrões ocultos e informações úteis de uma base de dados reais de fichas de triagem cardiológica infantil da UCMF, simulando diferentes níveis de infraestrutura médica através de 4 cenários de modelagem.

---

## 🔄 O que foi feito (Metodologia KDD)

O desenvolvimento do pipeline seguiu rigorosamente as etapas do processo KDD para transformar dados brutos em conhecimento acionável:

### 1. Exploracão e Filtragem Inicial 🔎
Antes de qualquer alteração, foi feita uma análise exploratória completa do dataset original **sem remover nenhum dado**. O objetivo foi mapear o estado real da base, identificando:
* Tipos de sujeira e preenchimentos incorretos.
* Linhas com ausência crítica de informações.
* Recursos (*features*) irrelevantes para o problema clínico ou puramente administrativas.

### 2. Limpeza e Padronização de Dados 🧹
Após o diagnóstico da base, aplicamos técnicas de engenharia de dados para garantir a consistência dos modelos:
* **Remoção de duplicatas:** Eliminadas logo no início para evitar vazamento de dados (*data leakage*) entre treino e validação.
* **Tratamento de valores impossíveis:** Filtragem de limites clínicos (ex: idades negativas ou >19 anos, batimentos cardíacos incompatíveis com a vida).
* **Padronização de textos:** Unificação de grafias variadas para o mesmo diagnóstico (ex: "sistólico" vs "Sistólico").
* **Correção de datas:** Recuperação de anos corrompidos (erros de século no Excel) e cálculo automático da idade real.

### 3. Cenários de Modelagem e a "Feature de Ouro" 🟡
Para avaliar o impacto real dos dados clínicos e forçar os modelos utilizados (Regressão Logística, SVC, Random Forest, Gradient Boosting, XGBoost) a encontrarem padrões mesmo na ausência de exames complexos, dividimos a análise em **4 cenários progressivos**. Isso nos permitiu testar a robustez dos algoritmos retirando gradativamente as informações, incluindo a **"feature de ouro"** (os achados do exame físico especializado do cardiologista, como Sopro e Pulsos):

| Cenário | Variáveis Removidas | O que simula na prática? |
| :--- | :--- | :--- |
| **Cenário A** | Nenhuma | **Modelo Completo:** Infraestrutura ideal com exame clínico especializado disponível. |
| **Cenário B** | SOPRO, PULSOS, B2, PPA | **Sem Exame Físico (Feature de Ouro):** Triagem em locais sem cardiologista ou estetoscópio avançado. |
| **Cenário C** | + MOTIVO1, MOTIVO2 | **Sem Encaminhamento:** Triagem puramente baseada em dados vitais e demográficos básicos. |
| **Cenário D** | + HDA 1, HDA2 | **Zero Impressão Clínica:** O cenário mais estrito; avalia se há sinal preditivo apenas em dados brutos de triagem. |

---

## 🩺 Como o resultado auxilia na prática?

O resultado dessa mineração de dados traz respostas de alto valor para a gestão de saúde pública e triagem clínica:

* **Medição do impacto da ausência do especialista:** Mostra matematicamente o quanto de acurácia (ROC-AUC/Recall) se perde quando o paciente não passa por uma ausculta especializada (Cenário B).
* **Identificação de padrões alternativos:** Força o modelo a descobrir relações ocultas entre sinais vitais básicos (Peso, Altura, IMC, Pressão Arterial e Frequência Cardíaca) para prever anormalidades cardíacas.
* **Otimização de recursos:** Ajuda a definir se um protocolo de triagem simplificado (Cenário C ou D) é viável e seguro para rodar em postos de saúde periféricos ou telemedicina, direcionando casos graves de forma eficiente mesmo sem um especialista na ponta.

---

## 🛠️ Requisitos e Instalação

* Python 3.9+
* Pandas, NumPy, SciPy, Matplotlib, Scikit-Learn, XGBoost
* Instalação:

```bash
pip install pandas numpy scipy matplotlib scikit-learn xgboost
```

Para a lógica de cada etapa, decisões de limpeza e justificativas, veja
[`DOCUMENTACAO.md`](./DOCUMENTACAO.md).

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

## Limitações conhecidas ❗

- O dataset cobre vários anos e parece ter mudado de protocolo ao longo do
  tempo (ex.: diferenciação de PPA "Não Calculado"); o split aleatório usado
  na análise principal é correto para estimar desempenho i.i.d., mas não
  captura esse possível drift — daí a checagem de holdout temporal ao final.
- Discrepâncias entre IDADE informada e idade calculada por data são apenas
  logadas, não corrigidas automaticamente; revisão manual é recomendada.
- Com poucos folds, o teste t pareado entre cenários tem baixo poder
  estatístico.
