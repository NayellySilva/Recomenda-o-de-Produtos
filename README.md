# Santander Product Recommendation — EDA, Séries Temporais & Baseline (TDE 01)

Este repositório contém a entrega técnica da **Fase 1 (TDE 01)** do projeto de recomendação de produtos financeiros baseado no desafio oficial do Kaggle: [*Santander Product Recommendation*](https://www.kaggle.com/c/santander-product-recommendation).

O foco deste trabalho compreende a **ingestão e amostragem reprodutível de grandes volumes de dados**, **limpeza e tratamento cadastral**, **análise exploratória multivariada (EDA)**, **modelagem de sazonalidade em séries temporais**, **definição de split cronológico sem data leakage** e a **implementação da métrica oficial (MAP@7) com benchmark de baseline**.

---

## 👥 Equipe Técnica e Divisão de Responsabilidades

O desenvolvimento deste projeto foi realizado colaborativamente na branch de desenvolvimento `TDE_01`, com cada membro assumindo uma etapa do pipeline:

| Integrante | Módulo Desenvolvido | Escopo Técnico / Commit |
| :--- | :--- | :--- |
| **Paulo Wendel** | *Setup & Ingestão* | Pipeline de leitura em blocos (*chunks*), amostragem de 30k clientes com semente fixa e exportação comprimida. |
| **Rafaele Ferreira** | *Limpeza & Tipagem* | Conversão forçada de tipos (`errors='coerce'`), consistência binária dos 24 produtos e auditoria estatística descritiva. |
| **Isac Oliveira** | *Demografia & Portfólio* | Mapeamento visual de idade, segmentação bancária, status de atividade e ranking de penetração total de contratos. |
| **Matheus Rosado** | *Sazonalidade & Split* | Engenharia de novas adesões via `diff() == 1`, dinâmica temporal mês a mês e divisão cronológica (Treino vs. Validação). |
| **Nayelly Roberta** | *Correlação & Sinergia* | Matriz multivariada de correlação de Pearson, heatmap de coocorrência de produtos e documentação técnica do projeto. |
| **Nilza Kelly** | *Métrica & Baseline* | Implementação formal do MAP@7, construção do modelo benchmark por popularidade e consolidação via Pull Request / Merge na `main`. |

---

## 🎯 Definição do Problema de Negócio

O objetivo consiste em antecipar as necessidades financeiras dos clientes do Banco Santander, prevendo **quais novos produtos bancários** serão contratados no período futuro.

* **Janela Histórica Total:** Janeiro de 2015 a Maio de 2016 (17 meses de observação).
* **Particularidade do Target:** O modelo não deve prever contratos já mantidos pelo cliente. O alvo é estritamente a **aquisição de novos produtos** (transição de estado de $0$ no mês anterior para $1$ no mês corrente).
* **Métrica Oficial de Avaliação:** **MAP@7** (*Mean Average Precision at 7*), priorizando a ordem de relevância das 7 principais recomendações.

---

## 🏗️ Arquitetura de Dados & Reprodutibilidade

O conjunto de dados bruto original disponibilizado no Kaggle (`train_ver2.csv`) possui **2.2 GB** e dezenas de milhões de registros, inviabilizando o versionamento direto no Git e gerando limitações de memória RAM.

### Estratégia de Amostragem Adotada:
1. **Mapeamento de IDs:** Leitura exclusiva da coluna `ncodpers` para mapear todos os clientes únicos.
2. **Amostragem Representativa:** Seleção pseudo-aleatória de **30.000 clientes** com semente determinística (`np.random.seed(42)`).
3. **Extração Iterativa em Chunks:** Varredura completa do dataset bruto em lotes de $1.000.000$ de linhas, capturando todo o histórico temporal desses 30k clientes.
4. **Otimização para Versionamento:** Compactação da base filtrada em formato Gzip (`santander_30k_sample.csv.gz`), reduzindo o volume de mais de 70 MB brutos para **~7.5 MB**, permitindo execução direta e imediata por qualquer avaliador.

* **Total de registros na amostra:** `427.751` linhas
* **Clientes únicos monitorados:** `30.000`
* **Colunas mapeadas:** `48` (sendo 24 variáveis de produtos `ind_*_ult1`)

---

## 🔍 Principais Descobertas da Análise Exploratória (EDA)

### 1. Auditoria Cadastral & Outliers
* **Idade (`age`):** Identificou-se uma distribuição bimodal concentrada em jovens universitários (20–25 anos) e profissionais adultos (40–50 anos). Foram detectados registros extremos (mínimo de 2 anos e máximo de 112 anos), exigindo tratamentos específicos nas etapas subsequentes.
* **Renda Familiar (`renta`):** Alta assimetria à direita e presença de valores ausentes (~20%), indicando a necessidade futura de imputação por medianas segmentadas por região/idade.

### 2. Penetração de Estoque vs. Novas Contratações
* **Líder em Estoque:** O produto `ind_cco_fin_ult1` (Conta Corrente) domina a base absoluta com mais de 270 mil ocorrências ativas.
* **Líderes de Novas Aquisições:** Ao isolar apenas as transições reais de contratação (`diff() == 1`), os produtos com maior dinamismo de adesão são:
  1. `ind_recibo_ult1` (Débito em Conta / Domiciliação de Pagamentos)
  2. `ind_nom_pens_ult1` (Pensão)
  3. `ind_nomina_ult1` (Conta Salário)

### 3. Sinergias e Cross-Selling (Correlação de Pearson)
* **Associação Quase Perfeita:** Identificou-se correlação extrema ($r \approx 0.96$) entre `ind_nomina_ult1` e `ind_nom_pens_ult1`.
* **Produtos Satélites:** Contratos de fluxo de renda (salário/pensão) apresentam forte correlação com Conta Nómina ($r \approx 0.78$) e Débito Automático ($r > 0.48$). A contratação de um serviço salarial funciona como gatilho imediato para múltiplos produtos adicionais.

### 4. Sazonalidade Temporal
* **Pico Anual de Contratações:** Fevereiro de 2016 atingiu o recorde histórico da série temporal (~1.570 novas adesões na amostra).
* **Ciclos de Entrada:** Observou-se um salto expressivo no volume total da base a partir de Julho/2015, correlacionado ao ciclo de captação de clientes universitários.

---

## ⏱️ Estratégia de Validação Cronológica (Anti-Leakage)

Modelos de recomendação com séries temporais exigem validação cronológica estrita para impedir vazamento de dados do futuro (*data leakage*):

```text
[------------- JANELA DE TREINO -------------] [--- JANELA DE VALIDAÇÃO ---]
Jan/2015 -------------------------> Fev/2016   Mar/2016 ------------> Mai/2016
           (340.614 linhas)                               (87.137 linhas)