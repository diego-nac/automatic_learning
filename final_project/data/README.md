# Datasets — Projeto Final CKP8277

**Disciplina:** CKP8277 – Aprendizagem Automática · UFC  
**Aluno:** Diego Melo — 603127

Sistema fotovoltaico monitorado entre dezembro de 2020 e outubro de 2021. Os dois arquivos capturam perspectivas complementares do mesmo sistema: as condições físicas externas (estação meteorológica) e o comportamento elétrico interno (inversor).

---

## 1. `estacao_meteorologica.csv`

### Visão geral

| Atributo | Valor |
|---|---|
| Formato | CSV |
| Registros | 35.336 |
| Colunas | 6 |
| Resolução temporal | ~1 minuto |
| Período | 2021-05-01 → 2021-10-13 |
| Valores nulos | Nenhum |

Dados coletados pela estação meteorológica instalada junto ao sistema fotovoltaico. Cada linha representa uma leitura instantânea das condições ambientais e da potência gerada pelos painéis.

**Este é o dataset indicado para treinamento do modelo de regressão de potência e detecção de anomalias**, pois relaciona causas físicas externas (irradiância, temperatura, vento) com o efeito (potência gerada), permitindo identificar desvios entre o comportamento esperado e o real.

### Colunas

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `DATETIME` | datetime | — | Timestamp da leitura (UTC-3, horário de Fortaleza) |
| `IRRADIATION` | int | W/m² | Irradiância solar global incidente sobre os painéis. Valor 0 indica período noturno ou ausência de sol. |
| `ENVTEMPSTATION` | float | °C | Temperatura do ar ambiente medida pela estação. Influencia a eficiência dos painéis por convecção. |
| `WINDSPEED` | float | m/s | Velocidade do vento. Contribui para o resfriamento dos painéis e redução das perdas térmicas. |
| `ENVTEMPPANELS` | int | °C | Temperatura superficial dos painéis fotovoltaicos. Temperaturas elevadas reduzem a eficiência (coeficiente de temperatura negativo). |
| `POWERFROMPANELS` | int | W | **Target.** Potência elétrica instantânea gerada pelos painéis. |

### Estatísticas descritivas

| Coluna | Mín | Média | Mediana | Máx | Desvio Padrão |
|---|---|---|---|---|---|
| IRRADIATION (W/m²) | 0 | 255,6 | 222 | 1.057 | 231,5 |
| ENVTEMPSTATION (°C) | 23,7 | 29,9 | 30,4 | 33,3 | 1,71 |
| WINDSPEED (m/s) | 0,0 | 2,15 | 2,0 | 9,4 | 1,14 |
| ENVTEMPPANELS (°C) | 26 | 48,5 | 48 | 57 | 3,58 |
| POWERFROMPANELS (W) | 0 | 20.378 | 15.812 | 55.028 | 16.517 |

### Observações importantes

- **1.919 registros** com `IRRADIATION = 0` correspondem a leituras noturnas ou de pré-amanhecer — potência gerada é zero nesses casos.
- **254 registros** com `POWERFROMPANELS = 0` e `IRRADIATION > 0` indicam possíveis anomalias ou partida do sistema.
- A temperatura dos painéis (`ENVTEMPPANELS`) é consistentemente **~18°C acima** da temperatura ambiente, reflexo do aquecimento por absorção de radiação.
- `IRRADIATION` é a feature com maior correlação com o target (Pearson r > 0,96).

---

## 2. `inversor.xlsx`

### Visão geral

| Atributo | Valor |
|---|---|
| Formato | XLSX (Excel) |
| Aba | Sheet1 |
| Registros | 171.146 |
| Colunas | 35 |
| Resolução temporal | ~1 minuto |
| Período | 2020-12-01 → 2021-10-13 |
| Valores nulos | Nenhum |

Dados exportados diretamente pelo inversor fotovoltaico via sistema de telemetria. Registra os parâmetros elétricos internos do equipamento: tensões e correntes no lado DC (painéis) e AC (rede elétrica), correntes individuais por string de painéis, temperatura interna e acumuladores de energia.

**Este dataset é indicado para monitoramento de saúde do sistema, diagnóstico de falhas por string e detecção de anomalias elétricas internas.** Não deve ser usado como features para prever potência pois as correntes CA e MPPT são matematicamente derivadas da própria potência (risco de data leakage).

### Modos de operação

| Modo | Registros | Descrição |
|---|---|---|
| Normal | 169.919 (99,3%) | Sistema gerando energia normalmente |
| Parado | 1.217 (0,7%) | Sistema em stand-by (noite, ausência de irradiância) |
| Fault | 10 (<0,01%) | Falha detectada pelo inversor |

### Colunas

#### Identificação temporal e estado

| Coluna | Tipo | Descrição |
|---|---|---|
| `hora` | datetime | Timestamp da leitura |
| `Modo de operação` | string | Estado do inversor: Normal, Parado ou Fault |

#### Lado DC — Entradas MPPT (Maximum Power Point Tracking)

O inversor possui 4 entradas MPPT independentes, cada uma conectada a um conjunto de strings de painéis.

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `V MPPT 1(V)` até `V MPPT 4(V)` | float | V | Tensão DC de entrada de cada tracker MPPT. Valores típicos: 530–740 V. |
| `I MPPT 1(A)` até `I MPPT 4(A)` | float | A | Corrente DC de entrada de cada tracker MPPT. Correlação com potência > 0,97. |

#### Lado AC — Saída para a Rede

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `Ua(V)`, `Ub(V)`, `Uc(V)` | float | V | Tensões de fase da rede elétrica trifásica (~234 V cada). |
| `I CA 1(A)`, `I CA 2(A)`, `I CA 3(A)` | float | A | Correntes de saída CA por fase. Correlação com potência > 0,98. |
| `F CA 1(Hz)`, `F CA 2(Hz)`, `F CA 3(Hz)` | float | Hz | Frequências da rede por fase (~60 Hz, norma brasileira). |

#### Correntes por String

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `Istr1(A)` até `Istr10(A)` | float / int | A | Corrente de cada string individual de painéis. Útil para detectar string sombreada, suja ou com módulo defeituoso. `Istr9` apresenta variância zero (possível sensor inativo). |

#### Métricas gerais do inversor

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `Potência(W)` | int | W | **Target.** Potência CA instantânea entregue à rede. |
| `Temperatura(℃)` | float | °C | Temperatura interna do inversor. Correlação moderada com potência (~0,62). |
| `Produção Hoje(kWh)` | float | kWh | Acumulador diário de energia gerada (reseta à meia-noite). |
| `Geração Total(kWh)` | float | kWh | Acumulador total desde a instalação do inversor. |
| `H Total(h)` | int | h | Horas totais de operação acumuladas. |
| `RSSI(%)` | int | % | Nível do sinal de comunicação sem fio do inversor (0–101%). Sem correlação com potência. |

### Estatísticas descritivas — colunas principais

| Coluna | Mín | Média | Mediana | Máx | Desvio Padrão |
|---|---|---|---|---|---|
| Potência (W) | 0 | 21.146 | 16.784 | 55.028 | 16.843 |
| Temperatura (°C) | 26,9 | 49,0 | 48,8 | 60,8 | 3,85 |
| V MPPT 1 (V) | 105,9 | 577,5 | 585,9 | 738,5 | 56,0 |
| I MPPT 1 (A) | 0,0 | 11,66 | 9,5 | 36,3 | 9,5 |
| Ua (V) | 0,0 | 234,8 | 235,0 | 242,2 | 2,98 |
| I CA 1 (A) | 0,0 | 31,3 | 25,2 | 82,9 | 24,7 |
| F CA 1 (Hz) | 0,0 | 60,0 | 60,0 | 60,15 | 0,41 |
| Istr1 (A) | 0,0 | 3,78 | 3,1 | 12,1 | 3,12 |
| RSSI (%) | 0 | 90,4 | 98 | 101 | 13,3 |

---

## Relação entre os Datasets

```
Condições externas           Sistema elétrico interno
(estacao_meteorologica.csv)  (inversor.xlsx)
─────────────────────────    ──────────────────────────────
IRRADIATION ──────────────►  V MPPT / I MPPT (entradas DC)
ENVTEMPPANELS                Istr1..10 (correntes por string)
ENVTEMPSTATION               Ua/Ub/Uc, I CA (saída AC)
WINDSPEED                    Temperatura interna
       │                           │
       └──────────► POWERFROMPANELS / Potência(W) ◄──────┘
                    (mesmo sinal físico, medido pelos dois)
```

| Uso recomendado | Dataset |
|---|---|
| Regressão de potência (previsão) | `estacao_meteorologica.csv` |
| Detecção de anomalias por resíduo | `estacao_meteorologica.csv` |
| Diagnóstico de string com falha | `inversor.xlsx` |
| Monitoramento de saúde do inversor | `inversor.xlsx` |
| Análise de modo Fault | `inversor.xlsx` |
