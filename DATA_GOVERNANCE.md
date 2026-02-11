## Objetivo

Este documento estabelece as diretrizes de governança, o dicionário de dados e o memorial descritivo das transformações aplicadas no banco de dados para todas as analises futuras.

---

#  1. Arquitetura do Banco de Dados

O banco foi desenhado em uma arquitetura de tabelas no SQL Server, separando dados cadastrais de métricas de performance e risco com dados ficticios criados por mim. Poderia ter pego dados da plataforma do kaggle porém como nossas regras no meio financeiro é bem especifica decidi criar manuamente e modelando as tabelas e colunas aplicando as regras necessarias como a de PDD, CAC, LTV. Isso me da mais opções para projetos futuros onde posso aplicar outros conceitos e regras.

### 🔑 Estrutura de Relacionamentos
<img width="1310" height="790" alt="DER" src="https://github.com/user-attachments/assets/ee73da8f-da1e-43d0-b1d0-0effb99f401f" />

* **`Cadastro (PK: ID_Cliente)`**: Tabela mestre de proponentes.
* **`Contratos (PK: ID_Contrato | FK: ID_Cliente)`**: Tabela de fatos de concessão.
* **`Rentabilidade (FK: ID_Contrato)`**: Extensão analítica de performance financeira.
* **`Risco (FK: ID_Contrato)`**: Extensão analítica de monitoramento de crédito e compliance.

---

# 2. Dicionário de Dados Detalhado

### 2.1 Tabela: `Cadastro` (Nativa)

| Coluna | Tipo | Descrição |
| --- | --- | --- |
| **ID_Cliente** | INT (PK) | Identificador único do cliente. |
| **Renda_Mensal** | DECIMAL | Renda bruta (normalizada após correção de escala). |
| **Score_Serasa_BoaVista** | INT | Pontuação de crédito externa. |
| **UF** | VARCHAR(2) | Unidade Federativa de residência. |

### 2.2 Tabela: `Contratos` (Nativa)

| Coluna | Tipo | Descrição |
| --- | --- | --- |
| **ID_Contrato** | INT (PK) | Identificador único da operação. |
| **Valor_Solicitado** | DECIMAL | Valor principal da dívida (Principal). |
| **Taxa_Juros_Mensal** | DECIMAL | Taxa nominal aplicada (Normalizada). |
| **Numero_Parcelas** | INT | Prazo contratual (ex: 12, 24, 48). |

### 2.3 Tabela: Rentabilidade

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| **ID_Contrato** | INT (FK) | Chave estrangeira vinculada à tabela de Contratos. |
| **Custo_Aquisicao_CAC** | DECIMAL | Custo comercial para aquisição do cliente (7% do principal). |
| **Margem_Contribuicao** | DECIMAL | Lucro bruto da operação após descontar o CAC. |
| **LTV** | DECIMAL | Projeção do valor total do cliente (Lifetime Value). |
| **Indice_Comprometimento_Renda** | DECIMAL | Percentual da renda mensal comprometido com a parcela. |

---

### 2.4 Tabela: Risco

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| **ID_Contrato** | INT (FK) | Chave estrangeira vinculada à tabela de Contratos. |
| **Rating_Interno** | VARCHAR | Classificação de risco (AA a H) conforme histórico e score. |
| **Status_Pagamento** | VARCHAR | Situação atual do contrato (Em Dia, Atrasado, Inadimplente). |
| **Dias_Atraso** | INT | Quantidade de dias de atraso no pagamento. |
| **Valor_Parcela** | DECIMAL | Valor da prestação mensal (calculado com juros). |
| **Saldo_Devedor_Atual** | DECIMAL | Montante que ainda resta ser pago pelo cliente. |
| **Provisao_PDD** | DECIMAL | Reserva financeira para perdas (conforme Resolução 2682). |
---

# 3. O "Motor de Risco" e a PDD (Provisão para Devedores Duvidosos) 
Seguindo as diretrizes da Resolução 2682 do BACEN, implementei uma lógica via SQL que ajusta a reserva financeira do banco com base no Rating (AA a H) e nos dias de atraso. Identifiquei contratos (como o "Caso 1004") que figuravam como "Em Dia", mas possuíam Rating F e 80 dias de atraso. Corrigir isso via código garante que a instituição não subestime suas perdas.

##  Motor de Risco e Cálculo de PDD

Este script realiza a conexão com o banco de dados SQL Server, processa a normalização de escalas financeiras e aplica regras de negócio para classificação de risco (PDD).

### Script Python
```python
import pyodbc
import pandas as pd
import numpy as np
from itables import show

# Aqui eu configura a conexao python com o banco de dados local utilizando o pyodbc
dados_conexao = (
    "Driver={SQL Server};"
    "Server=XXXXXX;"
    "Database=NERO_FORTE"
)
conexao = pyodbc.connect(dados_conexao)
cursor = conexao.cursor()

# Após conectado eu aplico as atualizações necessaria para abranger o calculo corredo da PDD
# Este bloco aplica as regras de escala, status por atraso e provisionamento (PDD)
sql_motor_risco = """
UPDATE R
SET 
    -- Normalização de Escala (Correção decimal)
    R.Valor_Parcela = R.Valor_Parcela / 100,
    R.Saldo_Devedor_Atual = R.Saldo_Devedor_Atual / 100,

    -- Motor de Status baseado em Dias de Atraso
    R.Status_Pagamento = CASE 
        WHEN R.Dias_Atraso = 0 THEN 'Em Dia'
        WHEN R.Dias_Atraso BETWEEN 1 AND 90 THEN 'Atrasado'
        ELSE 'Inadimplente (Default)' 
    END,

    -- Cálculo de Provisão (PDD) conforme Rating Interno (Lógica BACEN)
    R.Provisao_PDD = CASE 
        WHEN R.Rating_Interno = 'AA' THEN R.Saldo_Devedor_Atual * 0.005
        WHEN R.Rating_Interno = 'A'  THEN R.Saldo_Devedor_Atual * 0.01
        WHEN R.Rating_Interno = 'B'  THEN R.Saldo_Devedor_Atual * 0.02
        WHEN R.Rating_Interno = 'C'  THEN R.Saldo_Devedor_Atual * 0.03
        WHEN R.Rating_Interno = 'D'  THEN R.Saldo_Devedor_Atual * 0.10
        WHEN R.Rating_Interno = 'E'  THEN R.Saldo_Devedor_Atual * 0.30
        WHEN R.Rating_Interno = 'F'  THEN R.Saldo_Devedor_Atual * 0.50
        WHEN R.Rating_Interno = 'G'  THEN R.Saldo_Devedor_Atual * 0.70
        WHEN R.Rating_Interno = 'H'  THEN R.Saldo_Devedor_Atual * 1.00
        ELSE R.Saldo_Devedor_Atual * 0.05 
    END
FROM DBO.Risco R;
"""

cursor.execute(sql_motor_risco)
conexao.commit()

#Faço a leitura dos dados para validação
query_validacao = "SELECT * FROM DBO.Risco"
df_risco_final = pd.read_sql(query_validacao, conexao)

#E exibo usando o itables pois é uma visualização otima me dando um campo de search para dados dentro da tabela
show(df_risco_final)
```




#  4. Rentabilidade (CAC e LTV)

Quando criei a base de dados com dados aleatorios na tabelade de rentabilidade os valores de CAC estava irreal e estava dando um custo de aquisição maior que o emprestimo cedio ao cliente.
Então considerei as seguintes premissas para calculo do CAC, margem de contribução e LTV:

1. **CAC (7%)**  
   Provisão de custos de marketing e operacionais sobre o principal. Usando a formula:

   CAC = ValorSolicitado X 0,07 

3. **Margem de Contribuição**  
   Receita líquida de juros deduzida do custo de aquisição. Usando a formula:

   Margem = (ValorSolicitado X TaxaJuros/100 X Prazo) - CAC

4. **LTV (Fator 1.2x)**  
   Projeção conservadora considerando a probabilidade de novas operações ou upsell. Usando a formula:

   LTV = Margem de Contribuição X 1,2


### Script Python
```python
# Aqui calculamos o custo de aquisição, a margem líquida e a projeção de valor do cliente (LTV)
sql_rentabilidade = """
UPDATE R
SET 
    -- Cálculo do CAC: 7% do valor solicitado (Custo de marketing/operacional)
    R.Custo_Aquisicao_CAC = C.Valor_Solicitado * 0.07,

    -- Margem de Contribuição: (Juros Totais Recebidos) - CAC
    R.Margem_Contribuicao = (C.Valor_Solicitado * (C.Taxa_Juros/100) * C.Numero_Parcelas) - (C.Valor_Solicitado * 0.07),

    -- LTV: Estimativa de valor total considerando 20% de chance de renovação/upsell (1.2x)
    R.LTV = ((C.Valor_Solicitado * (C.Taxa_Juros/100) * C.Numero_Parcelas) - (C.Valor_Solicitado * 0.07)) * 1.2
FROM DBO.Rentabilidade R
INNER JOIN DBO.Contratos C ON R.ID_Contrato = C.ID_Contrato;
"""

#Aqui eu valido as atualizações
query_rentabilidade = """
SELECT 
    ID_Contrato, 
    Custo_Aquisicao_CAC, 
    Margem_Contribuicao, 
    LTV 
FROM DBO.Rentabilidade
"""
df_rentabilidade = pd.read_sql(query_rentabilidade, conexao)
show(df_rentabilidade)
```
    
   
   


# 5. Ajuste de escala e Cálculo de ICR (Índice de Comprometimento de Renda)

1. Analisando a base notei que os numeros estavam numa escala de 100x na coluna de valores de renda isso quer dizer que um cliente que ganha R$ 2.500,00 constava como R$ 250.000,00.
2. E outro problema foi que nao havia um calculo de quanto a parcela do empréstimo comprometia a renda do cliente, impedindo a segmentação de risco por superendividamento.

Para corrigir as ações realizadas foram:

1.Correção de Ponto Flutuante: Divisão sistemática por 100 com trava de segurança para evitar re-processamento.
2.Cálculo do ICR: Implementação da fórmula de comprometimento de renda para identificar o perfil de risco do tomador.

## Script Python
```python
# correção de escala e calculo de ICR
sql_saneamento_icr = """
-- Passo 1: Correção de escala
UPDATE DBO.Cadastro SET Renda_Mensal = Renda_Mensal / 100 WHERE Renda_Mensal > 100000;
UPDATE DBO.Contratos SET Valor_Solicitado = Valor_Solicitado / 100, Taxa_Juros_Mensal = Taxa_Juros_Mensal / 100;

-- Passo 2: Cálculo do ICR cruzando Tabelas de Cadastro, Contratos e Rentabilidade
UPDATE R
SET R.Indice_Comprometimento_Renda = (
    ((C.Valor_Solicitado * (1 + (C.Taxa_Juros_Mensal/100) * C.Numero_Parcelas)) / C.Numero_Parcelas) 
    / NULLIF(Cad.Renda_Mensal, 0)
) * 100
FROM DBO.Rentabilidade R
INNER JOIN DBO.Contratos C ON R.ID_Contrato = C.ID_Contrato
INNER JOIN DBO.Cadastro Cad ON C.ID_Cliente = Cad.ID_Cliente;
"""
```

# 6. Conclusão e Próximos Passos
Esse projeto vem para não apenas aplicar os meus conhecimentos, mas para mostrar um desafio real que se encontra no dia a dia onde nao apenas conhecimento de tecnologia é necessario para o sucesso das demandas.

Nessa normalização de dados eu entreguei:

   1. Criação do banco de dados 
   2. Dados limpos e confiaveis 
   3. Transparencia nos calculos para as regras de negocio aplicadas
   4. Base pronta para os proximos passos para um possivel modelo de IA

