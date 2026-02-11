# Credit Portfolio Risk Analytics
Projeto end-to-end de análise de risco de crédito, utilizando SQL, Python (ETL) e Power BI para avaliar crescimento da carteira, inadimplência, risco e rentabilidade.

## Objetivo do Projeto

Simular o ambiente analítico de uma instituição financeira, estruturando uma base de dados relacional e desenvolvendo um pipeline de dados para garantir a integridade de métricas estratégicas.

O foco central foi o saneamento de inconsistências e a implementação de **Regras de Negócio Bancárias** para análise de risco, provisão e rentabilidade.

---

## Arquitetura e Pipeline

Para garantir escalabilidade e integridade, o projeto foi estruturado em três camadas:

1. **Ingestão:** Python para tratamento inicial e integração via `pyodbc`
2. **Persistência:** SQL Server atuando como *Single Source of Truth*
3. **Business Logic:** ETL orientado a objeto e regras de negócio processadas diretamente no banco de dados para garantir performance

---

## Governança de Dados e Regras de Negócio

O maior diferencial deste projeto foi o tratamento de dados brutos inconsistentes, aplicando premissas reais do mercado financeiro brasileiro.

---

### 1️ Saneamento de Escala (Data Cleaning)

Foi identificada uma falha comum em ingestão de dados: perda do separador decimal em colunas monetárias e de taxas.

**Problema:**

* Valores de `Renda_Mensal` e `Taxa_Juros` apresentavam escala 100x maior que o real.

**Solução:**

* Normalização via SQL para refletir a realidade econômica do produto de Crédito Pessoal.

---

### 2️ Motor de Risco e Compliance (Resolução 2682 BACEN)

Implementação de lógica de provisionamento baseada em **Rating Interno** e **Dias de Atraso**, essencial para cálculo da **PDD (Provisão para Devedores Duvidosos)**.

**Correções Implementadas:**

* Ajuste automático de contratos com `Dias_Atraso > 0` que estavam incorretamente classificados como `"Em Dia"`
* Reclassificação de status para `"Inadimplente"` quando aplicável

**Cálculo de PDD aplicado sobre o Saldo Devedor:**

| Rating | Percentual de Provisão |
| ------ | ---------------------- |
| AA     | 0,5%                   |
| F      | 50%                    |
| H      | 100% (Perda estimada)  |

---

### 3️ Índice de Comprometimento de Renda (ICR)

Cálculo dinâmico do ICR cruzando dados de parcelas e renda mensal do cliente.

> **Premissa:** Prazos contratuais de 12, 24 e 48 meses impactam diretamente o risco de fluxo de caixa do cliente.

---

### 4️ Modelagem de Rentabilidade (CAC & LTV)

Criação de indicadores para identificar os perfis de clientes mais lucrativos.

* **CAC (Custo de Aquisição):** 7% sobre o valor contratado
* **Margem de Contribuição:** Receita de juros projetada - custo de aquisição
* **LTV (Lifetime Value):** Projeção com fator de renovação de 1.2x

---

## Tecnologias e Metodologia

* **SQL Server:** DDL, DML, `UPDATE` com `JOIN`, lógica de `CASE WHEN`
* **Python:** Automação do pipeline e extração de métricas
* **LaTeX:** Documentação formal das fórmulas de negócio




