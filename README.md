# 📊 DynamicScaler

**DynamicScaler** seleciona automaticamente o melhor scaler para cada variável numérica — e grava tudo de forma auditável.  
Ele combina testes estatísticos (normalidade, skew, curtose) com *optional* **validação cruzada preditiva** para garantir que **só transforma quando há ganho real**.

---

## ✨ Principais Características

| Recurso | Descrição |
|---------|-----------|
| **Estratégias** | `'auto'`, `'standard'`, `'robust'`, `'minmax'`, `'quantile'`, `None` (passthrough). |
| **Teste de normalidade** | `StandardScaler` só é considerado se o p‑valor do Shapiro‑Wilk ≥ `shapiro_p_val`. |
| **Fila inteligente** | `PowerTransformer → QuantileTransformer → RobustScaler → MinMaxScaler*` (*somente se `allow_minmax=True`). |
| **Validação estatística** | Checa pós‑transformação: desvio‑padrão, IQR e nº de valores únicos. |
| **Teste secundário** | Compara **kurtosis** à linha de base e a `kurtosis_thr`. |
| **Validação de importância** | Se `extra_validation=True` *ou* para `MinMaxScaler`, avalia ganho de importância via `importance_metric` e exige aumento ≥ `importance_gain_thr`. |
| **Auditável** | `report_as_df()` mostra métricas, candidatos testados, motivo de rejeição. |
| **Visual** | `plot_histograms()` compara distribuições antes/depois e exibe o scaler usado. |
| **Serialização segura** | Só salva scalers aprovados; usa hash de colunas para evitar mismatch em produção. |

---

## 🚀 Exemplo Rápido

```python
import pandas as pd
from dynamic_scaler import DynamicScaler   # nome do módulo/arquivo

df = pd.read_csv("meus_dados.csv")

scaler = DynamicScaler(
    strategy="auto",
    serialize=True,
    save_path="scalers.pkl",
    extra_validation=False    # desliga CV para rapidez
)

scaler.fit(df)
df_scaled = scaler.transform(df, return_df=True)

print(scaler.report_as_df().head())
scaler.plot_histograms(df, df_scaled, features=['idade', 'renda_mensal'])
```

### Exemplo avançado com validação de importância

```python
scaler_cv = DynamicScaler(
    strategy="auto",
    extra_validation=True,    # habilita validação para todos
    allow_minmax=True,        # deixa MinMax entrar
    importance_gain_thr=0.10, # exige aumento de 10% de importância
    importance_metric="shap",
    random_state=42
)

scaler_cv.fit(df_train[num_cols], y_train)
X_test_scaled = scaler_cv.transform(df_test[num_cols], return_df=True)
```

---

## 📊 Fluxo de Decisão (`strategy='auto'`)

```mermaid
flowchart TD
    Inicio --> VerificaIgnorados
    VerificaIgnorados -- ignorado --> Fim
    VerificaIgnorados -- ok --> TestaNormalidade
    TestaNormalidade -- normal --> EnfileiraStandard
    TestaNormalidade -- não_normal --> IgnoraStandard
    EnfileiraStandard --> Fila
    IgnoraStandard --> Fila
    Fila --> Loop
    Loop --> Candidato
    Candidato --> ValidaStats
    ValidaStats -- falha --> Loop
    ValidaStats -- passa --> ValidaSkew
    ValidaSkew -- não_melhora --> Loop
    ValidaSkew -- melhora --> ValidaKurt
    ValidaKurt -- falha --> Loop
    ValidaKurt -- passa --> CheckImp
    CheckImp -- necessidade_imp=true --> ValidaImp
    CheckImp -- necessidade_imp=false --> Escolhido
    ValidaImp -- ganho>=thr --> Escolhido
    ValidaImp -- ganho<thr --> Loop
    Loop -- fila_vazia --> SemScaler
    Escolhido --> Salva
    SemScaler --> Salva
    Salva --> Fim
```

### Segunda etapa de Validação

```mermaid
flowchart TD
    A[Novo Scaler] --> B{Skew reduzido?}
    B -- não --> Rejeita
    B -- sim --> C{Kurtosis adequada?}
    C -- não --> Rejeita
    C -- sim --> D{Importância habilitada?}
    D -- não --> Aceita
    D -- sim --> E{Ganho ≥ importance_gain_thr?}
    E -- sim --> Aceita
    E -- não --> Rejeita
```

---

## 📒 Referência de API

| Método | Descrição |
|--------|-----------|
| `fit(X, y=None)` | Treina e seleciona scalers; aceita `y` se precisar de CV. |
| `transform(X, return_df=False)` | Aplica scalers aprovados. |
| `inverse_transform(X)` | Reverte escalonamento. |
| `report_as_df()` | DataFrame detalhado com decisão e métricas. |
| `plot_histograms(orig, trans, features, show_qq=False)` | Visualiza distribuições antes/depois. |
| `save(path)` / `load(path)` | Serializa e restaura scalers + relatório + metadados. |

## 📝 Colunas do report

| Coluna | Descrição |
|--------|-----------|
| `chosen_scaler` | Nome do scaler aprovado ou `None`. |
| `validation_stats` | Métricas pós-transformação. |
| `ignored` | Lista de scalers ignorados. |
| `candidates_tried` | Candidatos testados. |
| `reason` | Pipe-separated flags explicando por que o scaler foi aceito (ex. stats|skew|kurt|imp). |

---

## ⚙️ Parâmetros Importantes

| Parâmetro | Default | Descrição |
|-----------|---------|-----------|
| `shapiro_p_val` | `0.01` | Valor‑p mínimo para considerar a variável normal. |
| `shapiro_n` | `5000` | Amostra máxima para o teste de Shapiro‑Wilk. |
| `validation_fraction` | `0.1` | Fração dos dados reservada para validação interna. |
| `kurtosis_thr` | `10.0` | Limite absoluto de curtose pós‑transformação. |
| `extra_validation` | `False` | Habilita CV preditiva para **todos** os candidatos. |
| `allow_minmax` | `True` | Permite que `MinMaxScaler` entre na fila. |
| `importance_metric` | `'shap'` | Métrica de importância: `'shap'`, `'gain'` ou função custom. |
| `importance_gain_thr` | `0.10` | Aumento relativo mínimo na importância da feature. |
| `cv_gain_thr` | `0.002` | (deprecated) mapeado para `importance_gain_thr`. |
| `ignore_scalers` | `[]` | Lista de scalers a serem ignorados de antemão. |

*(veja `help(DynamicScaler)` para todos os parâmetros)*

---

## 🔐 Serialização e Hash

Ao salvar, o DynamicScaler:
1. Mantém **apenas os scalers aprovados** (`selected_cols_`).
2. Cria um **hash MD5** das colunas salvas para garantir consistência.  
   No `load()`, se o hash divergir, é levantado erro — evita usar um scaler
   incompatível com o dataset atual.

---

## 🤝 Contribuições

Contribuições são bem‑vindas!  
Faça **fork**, crie um branch, abra seu *pull request* e vamos evoluir juntos.  
Issues com dúvidas, bugs ou sugestões são muito bem‑vindas.

---

> **Licença**: MIT
