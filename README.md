# 📊 Desafio de Análise e Visualização de Dados – FADESP 2025

Este repositório contém a entrega do desafio técnico para a vaga de Analista de BI da FADESP, com foco em análise de dados, visualização e geração de insights.

## ✅ Estrutura do Repositório

- 📁 **etapa_1/**
  - Arquivo `.pbix` com indicadores e visualizações solicitadas na primeira etapa.
  - Exploração inicial de dados com segmentações por tempo, faturamento e desempenho.
  
  ### 📌 Power Query (M)
  ### 🗂️ Conexão com a Fonte

- Foi criada uma conexão com a **pasta de dados**, contendo os arquivos `.csv`, utilizando `Folder.Files`.
- A consulta foi nomeada como `Fonte_Local`, onde foram listados todos os arquivos disponíveis.

### 🔄 Otimização das Consultas

- A partir da `Fonte_Local`, **foi aplicada uma etapa de filtro** para extrair somente os arquivos `.csv` relevantes, descartando arquivos `.rar` e `.pbix`.
- Para cada arquivo `.csv`, foi criada uma nova **consulta referenciada** diretamente da `Fonte_Local`, de forma que **apenas uma consulta acesse a fonte**, reduzindo o número de leituras e melhorando a performance.

### 📥 Exemplo das consultas criadas:
- `avaliacoes_pedidos`
- `categorias_produtos`
- `clientes`
- `itens_pedidos`
- `pedidos`
- `pedidos_pagamentos`
- `produtos`
- `vendedores`
- `geolocalizacao`

🔕 A consulta `Fonte_Local` teve a **carga desabilitada** para evitar consumo desnecessário de memória.

Este script em M é um exemplo de limpeza de dados é realizado um mapeamento de categorias de produto de inglês para português, com tratamento de texto, substituições e padronização de capitalização.

let
    Produtos = Fonte_Local,

    Cabeçalhos_Promovidos =
    Table.PromoteHeaders(
        Csv.Document(
            File.Contents(Caminho & "\produtos.csv"),
            [Delimiter=",", Columns=9, Encoding=1252, QuoteStyle=QuoteStyle.Csv]
        ),
        [PromoteAllScalars=true]
    ),

    Tranformação_Dados = Table.TransformColumnTypes(
        Cabeçalhos_Promovidos,
        {
            {"product_id", type text},
            {"product_category_name", type text},
            {"product_name_lenght", Int64.Type},
            {"product_description_lenght", Int64.Type},
            {"product_photos_qty", Int64.Type},
            {"product_weight_g", Int64.Type},
            {"product_length_cm", Int64.Type},
            {"product_height_cm", Int64.Type},
            {"product_width_cm", Int64.Type}
        }
    ),

    // Substituições da categoria em português
    Substituicoes = List.Zip({
        Dim_categorias_produtos[product_category_name_english],
        Dim_categorias_produtos[product_category_name]
    }),

    // De-para de nomes
    De_Para = Table.TransformColumns(
        Tranformação_Dados,
        {   
            {
                "product_category_name",
                each
                    List.Accumulate(
                        Substituicoes,
                        _,
                        (state, pair) =>
                            Text.Replace(
                                " " & state & " ",
                                " " & pair{0} & " ",
                                " " & pair{1} & " "
                            )
                    )
            }
        }
    ),

    // Transformações adicionais: nulos, underscores, duplos underscores
    Tratamento_Texto = Table.TransformColumns(
        De_Para,
        {
            {
                "product_category_name",
                each
                    let
                        valorOriginal = if _ = null or Text.Trim(_) = "" then "outros" else _,
                        comBarra = Text.Replace(valorOriginal, "__", "/"),
                        comEspaco = Text.Replace(comBarra, "_", " ")
                    in
                        comEspaco,
                type text
            }
        }
    ),

    // Remoção de espaços extras e capitalização
    Ultimo_Tratamento = Table.TransformColumns(
        Tratamento_Texto,
        {
            {
                "product_category_name",
                each Text.Proper(Text.Trim(_)),
                type text
            }
        }
    )

in
    Ultimo_Tratamento

  ### 📌 Medidas (DAX)
  - Utilização de DAX para criação de medidas como YoY, ticket médio e classificação de clientes.
  - Criação de Análises temporais com o objetivo de Maximizar os insights do usuário.

Faturamento YoY = 
  -- Medida criada em 19/06/2025 às 18:06 por Vinícius Ribeiro
  -- O objetivo dessa medida é calcular o faturamento do mesmo período do ano anterior (Year over Year),
  -- considerando a data da última venda registrada no dataset

VAR dataUltimaVenda =
    CALCULATE(
        LASTDATE(Dim_Pedidos[order_purchase_timestamp]),
        REMOVEFILTERS(Dim_Calendario)
    )

VAR dataLimite =
    EDATE(dataUltimaVenda, -12)

VAR YoY =
    CALCULATE(
        [001. Sales Amount],
        DATEADD(Dim_Calendario[Data], -12, MONTH),
        FILTER(
            ALL(Dim_Calendario), Dim_Calendario[Data] <= dataLimite
        )
    )

RETURN
    YoY

Exemplo de medida criada com o objetivo de análisar de forma temporal o faturamento do ano anterior ao selecionado no dashboard.

- 📁 **etapa_2/**
  - Dashboard complementar em `.pbix`, com foco em KPIs adicionais e filtros dinâmicos.
  - Utilização de DAX para criação de medidas.

  ### 📌 Power Query (M)
  ### 🗂️ Conexão com a Fonte

- Foi criada uma conexão com a **pasta de dados**, contendo os arquivos `.csv`, utilizando `Folder.Files`.
- A consulta foi nomeada como `Fonte_Local`, onde foram listados todos os arquivos disponíveis.

### 🔄 Otimização das Consultas

- A partir da `Fonte_Local`, **foi aplicada uma etapa de filtro** para extrair somente os arquivos `.csv` relevantes, descartando arquivos `.rar` e `.pbix`.
- Para cada arquivo `.csv`, foi criada uma nova **consulta referenciada** diretamente da `Fonte_Local`, de forma que **apenas uma consulta acesse a fonte**, reduzindo o número de leituras e melhorando a performance.

### 📥 Exemplo das consultas criadas:

- `di_estado`
- `di_municipio`
- `arrecadacao_impostos_municipais`

🔕 A consulta `Fonte_Local` teve a **carga desabilitada** para evitar consumo desnecessário de memória.

Este script em M é um exemplo de limpeza de dados é realizado um mapeamento dos Estados do Brasil e é feito o tratamento para que ela se transforme em uma tabela dimensão já que existiam muitos dados duplicados

let
    Excel = Fonte_Local, //Faz a conexão com a fonte local somente uma vez, o que evita excessos de consultas a base de dados

    Arquivo_Excel = Excel{[#"Folder Path" = Caminho, Name = "di_estado.xlsx"]}[Content],

    Planilha = Excel.Workbook(Arquivo_Excel, true),

    DTB_Municipios_Sheet = Planilha{[Item = "DTB_Municípios", Kind = "Sheet"]}[Data],

    // Converte em lista de registros para varredura
    LinhasComoRegistros = Table.ToRecords(DTB_Municipios_Sheet),

    // Processo de limpeza seleciona colunas válidas com valores diferente de (≠ null, "", espaço ou "Novo!")
    ColunasValidas = List.Select(
        Table.ColumnNames(DTB_Municipios_Sheet),
        (colName) =>
            List.Count(
                List.Select(
                    List.Transform(LinhasComoRegistros, each Record.Field(_, colName)),
                    each _ <> null and Text.Trim(Text.From(_)) <> "" and Text.Trim(Text.From(_)) <> "Novo!"
                )
            ) > 0
    ),

    // Mantém apenas as colunas válidas
    Remover_Colunas_Lixo = Table.SelectColumns(DTB_Municipios_Sheet, ColunasValidas),

    // Tipagem das colunas principais
    Transformacao_Dados = Table.TransformColumnTypes(
        Remover_Colunas_Lixo,
        {
            {"UF", type text},
            {"Nome_UF", type text}
        }
    ),

    // Remove duplicatas com base na coluna UF
    Remover_Duplicatas_UF = Table.Distinct(Transformacao_Dados, {"UF"})
    
in
    Remover_Duplicatas_UF

 ### 📌 Medidas (DAX)
  - Utilização de DAX avançado para criação de medidas.
  - Criação de Análises temporais com o objetivo de Maximizar os insights do usuário.

arrecadacaoExibicao = 
-- Medida criada em 20/06/2025 às 18:06 por Vinícius Ribeiro

/* O objetivo dessa medida é exibir um texto fixo "Arrecadação R$" no nível total geral do relatório,
   e, nos demais níveis de granularidade (estado ou município), exibir o valor efetivo da arrecadação.
   Caso o valor seja igual ou inferior a zero, retorna BLANK() para ocultar no visual.
   Isso ajuda a evitar a poluição visual quando não há dados relevantes a serem apresentados.*/

VAR ValorArrecadado = SUM(arrecadacao_impostos_municipais[valor_arrecadado])
VAR EhTotalGeral = NOT ISINSCOPE(di_estado[Nome_UF]) && NOT ISINSCOPE(di_municipio[nome_municipio])

RETURN
    IF(
        EhTotalGeral,
        "Arrecadação R$",
        IF(ValorArrecadado > 0, ValorArrecadado, BLANK())
    )
--------------------------//-------------------------------

corCondicional = 
-- Medida criada em 20/06/2025 às 19:09 por Vinícius Ribeiro
 
/* O objetivo dessa medida é retornar uma cor condicional para destacar visualmente valores de arrecadação.
   Se o valor de arrecadação ultrapassar o limite definido (R$ 3.455.936.866), a cor retornada será laranja (#E66C37).
   Caso contrário, a cor será branca (#FFFFFF).
   Essa lógica é utilizada para aplicar formatação condicional em gráficos ou tabelas, chamando atenção para valores elevados. */

VAR Limite = 3455936866

VAR Valor = SUM(arrecadacao_impostos_municipais[valor_arrecadado])

RETURN
    IF(
        Valor > Limite,
        "#E66C37",   // Laranja
        "#FFFFFF"    // Branco
    )

- 📁 **etapa_3/**
  - Notebook Python (`.ipynb`) com análise exploratória dos dados.
  - Gráfico de dispersão interativo usando `plotly.express`, correlacionando **quantidade vendida** e **valor total por categoria**.
  - Aplicação das bibliotecas: pandas, matplotlib, seaborn, plotly.express e plotly.graph_objects.
  - Utilização de SQLite como banco local para execução de consultas SQL otimizadas.

📊 Análises realizadas:
  - Ranking de vendas por estado, com valor total formatado em R$.
  - Tempo médio de entrega, calculado com JULIANDAY() via SQL e representado em gauge plot (velocímetro).
  - Volume de vendas por mês, com visualização em gráfico de linha.
  - Dispersão de faturamento por categoria de produto, com destaques para as top 10 em vendas.

🧠 Etapas principais no código:
  - Leitura dos .csv e conversão para tabelas SQLite.
  - Normalização dos tipos com pandas.
  - Criação e execução de múltiplas consultas SQL para extração de insights.
  - Visualizações com matplotlib e plotly, incluindo:
  - Gráficos de barras horizontais com formatação brasileira.
  - Dispersões com rótulos condicionais.
  - Gráfico de indicador (tipo velocímetro) para tempo médio de entrega.

🧰 Exemplo de código:

  query_vendas_estado = """
    SELECT
        c.customer_state AS estado,
        ROUND(SUM(pag.payment_value), 2) AS total_vendas
    FROM 
        pedidos ped
    INNER JOIN pedidos_pagamentos pag ON 
      ped.order_id = pag.order_id
    INNER JOIN clientes c ON 
      ped.customer_id = c.customer_id
    GROUP BY 
      c.customer_state
    ORDER BY 
      total_vendas DESC
  """
  vendas_estado = pd.read_sql_query(query_vendas_estado, conn)

📈 Exemplos de Visualização

🔹 Tempo Médio de Entrega (Velocímetro com Plotly)
  fig = go.Figure(go.Indicator(
    mode = "gauge+number",
    value = float(tempo_entrega.iloc[0, 0]),
    title = {'text': "⏱ Tempo Médio de Entrega (dias)"},
    gauge = {
        'axis': {'range': [0, 30]},
        'bar': {'color': "black"},
        'steps': [
            {'range': [0, 10], 'color': "lightgreen"},
            {'range': [10, 20], 'color': "gold"},
            {'range': [20, 30], 'color': "tomato"}
        ]
    }
))
fig.show()


🔹 Total de Vendas por Estado (Barplot Horizontal)
plt.figure(figsize=(12, 8))
sns.barplot(data=vendas_estado, x="total_vendas", y="estado", palette="Blues_d")
plt.title("💰 Total de Vendas por Estado", fontsize=14)
plt.xlabel("Total em R$")
plt.ylabel("Estado")
plt.grid(axis='x')
plt.tight_layout()
plt.show()



## 📌 Observações

- Os arquivos `.pbix` foram compactados em `.zip` quando ultrapassaram 25MB, conforme exigência do GitHub.
- A análise foi realizada com base em dados simulados de um e-commerce.

## 📅 Entrega

Data limite: **22/06/2025**

---

Em caso de dúvidas ou necessidade de ajustes, fico à disposição para esclarecimentos.

