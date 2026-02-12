# 📈 Perguntas de Negócio

Nesta seção, o objetivo é extrair inteligência do banco de dados para responder a questionamentos estratégicos.
Estarei utilizando Python (pandas) e SQL para responder a esses itens.

---

### 1. Performance e Crescimento da Carteira

> **Pergunta:** A carteira está crescendo?

```sql
query_crescimento = """
SELECT 
    FORMAT(C.Data_Contratacao, 'yyyy-MM') AS Mes_Referencia,
    COUNT(C.ID_Contrato) AS Qtd_Contratos,
    SUM(C.Valor_Solicitado) AS Volume_Originado,
    AVG(C.Valor_Solicitado) AS Ticket_Medio,
    SUM(R.Saldo_Devedor_Atual) AS Saldo_Devedor_Total
FROM DBO.Contratos C
INNER JOIN DBO.Risco R ON C.ID_Contrato = R.ID_Contrato
GROUP BY FORMAT(C.Data_Contratacao, 'yyyy-MM')
ORDER BY Mes_Referencia
"""

df_crescimento = pd.read_sql(query_crescimento, conexao)

#Calcula a variação percentual mês a mês 
df_crescimento['Variacao_Saldo_%'] = df_crescimento['Saldo_Devedor_Total'].pct_change() * 100
#Exibindo os resultados
print("Análise de Evolução de Carteira")
show(df_crescimento)

```

<img width="823" height="121" alt="image" src="https://github.com/user-attachments/assets/4e665b29-02e2-4836-8b52-2e16fcfdd895" />


A carteira apresentou uma queda em novembro, quando o saldo devedor total recuou 10,27% em relação a outubro. No entanto, em dezembro houve uma recuperação significativa, com crescimento de 25,01%, elevando o saldo para um nível superior ao observado no início do período. Dessa forma, apesar da retração pontual em novembro, a carteira encerra o trimestre maior do que começou, indicando crescimento no consolidado.

---

### 2. Qualidade de Crédito (Inadimplência)

> **Pergunta:** Qual é o percentual do saldo que já está em atraso superior a 90 dias?
Estatisticamente após 90 dias de atraso, a probabilidade de um cliente pagar a dívida cai drasticamente, e o banco geralmente começa a considerar esse valor como perda real.
Utilizando a seguinte formula:
<img width="420" height="60" alt="image" src="https://github.com/user-attachments/assets/7d9aaaf1-6062-49f4-bfc5-44d38775c19c" />

```sql
query_npl = """
SELECT 
    FORMAT(C.Data_Contratacao, 'yyyy-MM') AS Mes_Referencia,
    SUM(R.Saldo_Devedor_Atual) AS Saldo_Total,
    SUM(CASE WHEN R.Dias_Atraso > 90 THEN R.Saldo_Devedor_Atual ELSE 0 END) AS Saldo_NPL_90
FROM DBO.Contratos C
INNER JOIN DBO.Risco R ON C.ID_Contrato = R.ID_Contrato
GROUP BY FORMAT(C.Data_Contratacao, 'yyyy-MM')
ORDER BY Mes_Referencia
"""

# Carregando os dados
df_npl = pd.read_sql(query_npl, conexao)

# Calculando o indicador NPL 90%
df_npl['NPL_90_Percentual'] = (df_npl['Saldo_NPL_90'] / df_npl['Saldo_Total']) * 100

print(" Análise de Inadimplência (NPL 90)")
show(df_npl)

```
 <img width="519" height="127" alt="image" src="https://github.com/user-attachments/assets/5e754adb-0d10-4fc0-a8de-2e9488c67dcf" />

O crescimento de 25% em dezembro foi muito positivo porque o risco (NPL 90) não subiu, ele caiu para 2,48%. Isso prova que a expansão da carteira foi saudável e que o crédito está a ser bem selecionado.


---

### 3. Clientes

> **Pergunta:** Quais perfis de clientes são mais rentáveis?

```sql
query_perfil_rentavel = """
SELECT 
    CAD.Ocupacao, -- Use o nome exato que apareceu no seu print do Cadastro
    COUNT(CON.ID_Contrato) AS Total_Contratos,
    AVG(CON.Valor_Solicitado) AS Ticket_Medio,
    SUM(RIS.Saldo_Devedor_Atual) AS Saldo_Total,
    SUM(RIS.Provisao_PDD) AS Total_PDD,
    AVG(CON.Taxa_Juros_Mensal) AS Taxa_Media
FROM DBO.Cadastro CAD
INNER JOIN DBO.Contratos CON ON CAD.ID_Cliente = CON.ID_Cliente
INNER JOIN DBO.Risco RIS ON CON.ID_Contrato = RIS.ID_Contrato
GROUP BY CAD.Ocupacao
ORDER BY Total_Contratos DESC
"""
df_perfil = pd.read_sql(query_perfil_rentavel, conexao)
df_perfil['Risco_Relativo_%'] = (df_perfil['Total_PDD'] / df_perfil['Saldo_Total']) * 100

print("Análise de Rentabilidade por Perfil (Ocupação)")
show(df_perfil)
```
<img width="820" height="195" alt="image" src="https://github.com/user-attachments/assets/22b2e882-7047-4765-bc7b-e68849a837ff" />


Ao analisar a rentabilidade por ocupação, identifiquei que o perfil Pensionista é o mais eficiente para a estratégia de risco do banco, apresentando o menor índice de Provisão (6,84%).
Por outro lado, o perfil Aposentado, apesar de ter um ticket médio menor, possui um risco relativo de 10,08%. Isso indica que, para cada real emprestado nesse segmento, o banco precisa guardar quase o dobro de reserva comparado aos Pensionistas.

---

### 4. Eficiência Comercial

> **Pergunta:** Qual canal de venda (Agência, Digital, Parceiros), onde o banco deve investir mais budget?

```sql
query_canais = """
SELECT 
    C.Canal_Venda,
    COUNT(C.ID_Contrato) AS Total_Contratos,
    SUM(C.Valor_Solicitado) AS Volume_Financeiro,
    SUM(R.Saldo_Devedor_Atual) AS Saldo_Atual,
    SUM(R.Provisao_PDD) AS PDD_Total
FROM DBO.Contratos C
INNER JOIN DBO.Risco R ON C.ID_Contrato = R.ID_Contrato
GROUP BY C.Canal_Venda
ORDER BY Volume_Financeiro DESC
"""

df_canais = pd.read_sql(query_canais, conexao)
df_canais['Risco_Relativo_%'] = (df_canais['PDD_Total'] / df_canais['Saldo_Atual']) * 100

print("🎯 Análise de Eficiência por Canal de Venda")
show(df_canais)

```
<img width="737" height="142" alt="image" src="https://github.com/user-attachments/assets/4f42b308-b389-4ff4-9c92-0315c9597be0" />

Analisando os canais de venda, recomendo priorizar o investimento no canal Web. Ele não só trouxe o maior volume financeiro (R$ 7,35 milhões), como também possui o menor índice de risco (6,21%).
Já o canal App, apesar de estar crescendo em volume, apresenta o maior risco relativo (8,88%), o que exige uma reserva de capital maior. Portanto, o budget deve focar em escalar o canal Web enquanto revisamos os critérios de aprovação do App.

---

### 5. Fechamento

> **Pergunta:** Quais são os principais alertas do trimestre?

A explicação teorica que retirei da internet sobre essa analise...

***"Os dados de Rating Interno mostram a qualidade da carteira conforme a lógica da Resolução CMN 2.682: quanto pior o rating, maior a provisão (PDD) exigida pelo banco.
A maior parte dos clientes está concentrada nos ratings **A e AA**, indicando boa qualidade da carteira. O rating **D** já representa um ponto de atenção, pois responde por 13,17% da PDD. Já o rating **F** é o principal alerta: mesmo com poucos contratos (menos de 5% da base), consome 23,10% de toda a provisão do trimestre, pressionando o custo de capital.
"***

Resumidamente quanto mais proximo do indicador A e AA melhor, quanto mais longe pior. 

```sql
query_alertas = """
SELECT 
    R.Rating_Interno,
    COUNT(C.ID_Contrato) AS Qtd_Contratos,
    SUM(R.Saldo_Devedor_Atual) AS Saldo_Exposto,
    SUM(R.Provisao_PDD) AS PDD_Total
FROM DBO.Contratos C
INNER JOIN DBO.Risco R ON C.ID_Contrato = R.ID_Contrato
GROUP BY R.Rating_Interno
ORDER BY R.Rating_Interno
"""

df_ratings = pd.read_sql(query_alertas, conexao)
df_ratings['Representatividade_PDD_%'] = (df_ratings['PDD_Total'] / df_ratings['PDD_Total'].sum()) * 100

print("Distribuição de Risco por Rating")
show(df_ratings)

```
<img width="665" height="229" alt="image" src="https://github.com/user-attachments/assets/9d2226c0-175a-4737-874a-e24c5ceb9127" />

O principal alerta do trimestre é o Rating F. Identifiquei que este grupo é altamente ineficiente: ele possui poucos clientes, mas exige uma provisão de R$ 1.907,43, o que representa quase um quarto de toda a PDD do banco. Isso indica que uma pequena parcela de contratos de baixa qualidade está 'comendo' o lucro gerado pelos clientes Rating A.
