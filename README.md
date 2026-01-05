# Simulador-de-investimento-de-fundo-imobili-rio
Simulador de investimento de fundo imobiliário

Simulador_FII_5_Anos_Painel (geração via Python)
-------------------------------------------------
Este script cria uma planilha Excel completa para simulação de investimentos em FIIs/REITs,
seguindo a estrutura observada no anexo e incorporando um painel de indicadores e gráficos.

Abas criadas:
- PARÂMETROS: controles gerais (aporte, corretagem %, horizonte, salário alvo, meta de renda etc.)
- CADASTROS: cadastro de ativos (ticker, categoria, preço atual)
- CARTEIRA: com pesos-alvo e DY anual por ativo
- SIMULAÇÃO: motor mensal com caixa acumulado por ativo, compras condicionadas e totais
- FLUXOS: série de fluxos (aportes, dividendos, liquidação final) com XIRR
- RELATÓRIOS: indicadores resumidos por ativo
- INVESTIMENTOS: estrutura para registrar compras manuais (opcional)
- DIVIDENDOS & SAQUES: estrutura para registrar proventos/saques manuais (opcional)
- PAINEL: KPIs e gráficos (patrimônio, renda vs meta, alocação, dividendos 12m)

Requisitos: openpyxl (já disponível no ambiente)
"""
from openpyxl import Workbook
from openpyxl.styles import Font, Alignment, PatternFill, Border, Side
from openpyxl.utils import get_column_letter
from openpyxl.worksheet.datavalidation import DataValidation
from openpyxl.chart import LineChart, BarChart, PieChart, Reference
from datetime import date

# ---------- Util ----------
thin = Side(style='thin', color='CCCCCC')
border = Border(left=thin, right=thin, top=thin, bottom=thin)

wb = Workbook()

# =========================
# Aba: PARÂMETROS
# =========================
ws_p = wb.active
ws_p.title = "PARÂMETROS"
headers = [("A", "Campo"), ("B", "Valor"), ("C", "Descrição")]
for col, title in headers:
    ws_p[f"{col}1"] = title
    ws_p[f"{col}1"].font = Font(bold=True)

params = [
    ("Moeda", "BRL", "Moeda base"),
    ("Aporte inicial", 0, "Valor inicial aplicado"),
    ("Aporte mensal", 800, "Aporte recorrente"),
    ("Crescimento anual dos aportes (%)", 0.03, "Taxa anual; a mensal é obtida por equivalência"),
    ("Horizonte (anos)", 5, "Número de anos simulados"),
    ("Reinvestir dividendos (Sim/Não)", "Sim", "Se 'Sim', dividendos acumulam caixa para compras"),
    ("Taxa de corretagem (% por ordem)", 0.005, "Percentual sobre o valor de cada ordem"),
    ("Taxa de custódia (% a.a.)", 0.0, "Percentual anual sobre patrimônio"),
    ("Inflação anual (%)", 0.04, "Para análise real"),
    ("Data de início", date(date.today().year, date.today().month, 1), "Primeiro mês da simulação"),
    ("Salário mensal alvo (BRL)", 8000, "Faixa referência: R$ 5k–10k"),
    ("Meta de renda mensal (10% do salário)", "=0.10*B11", "Renda alvo mensal (10%)"),
]
for i, (campo, valor, desc) in enumerate(params, start=2):
    ws_p[f"A{i}"] = campo
    ws_p[f"B{i}"] = valor
    ws_p[f"C{i}"] = desc

# Auxiliares
ws_p["A14"] = "Fator crescimento mensal dos aportes"; ws_p["C14"] = "Taxa mensal equivalente"
ws_p["B14"] = "=(1+B4)^(1/12)-1"
ws_p["A15"] = "Meses simulados"; ws_p["C15"] = "Total de meses (anos × 12)"
ws_p["B15"] = "=B5*12"

# Validação de dados
reinvdv = DataValidation(type="list", formula1='"Sim,Não"', allow_blank=False)
ws_p.add_data_validation(reinvdv)
reinvdv.add(ws_p["B6"])  # Reinvestir dividendos

# =========================
# Aba: CADASTROS
# =========================
ws_d = wb.create_sheet("CADASTROS")
cols_d = ["ATUALIZAÇÃO", "NOME", "CATEGORIA", "PREÇO ATUAL"]
for j, h in enumerate(cols_d, start=1):
    c = ws_d.cell(row=1, column=j, value=h)
    c.font = Font(bold=True)
    c.fill = PatternFill("solid", fgColor="D9E1F2")
    c.alignment = Alignment(horizontal="center")

cadastros = [
    (date.today(), "CORP11", "FII - Corporativo", 100.00),
    (date.today(), "OFFI11", "FII - Escritórios", 120.00),
    (date.today(), "CENT11", "FII - Centros Comerciais", 90.00),
    (date.today(), "LOGS11", "FII - Logística e Suprimentos", 110.00),
    (date.today(), "GALP11", "FII - Galpões", 105.00),
    (date.today(), "ROTA11", "FII - Renda Urbana", 95.00),
    (date.today(), "SHOP11", "FII - Shoppings", 98.00),
    (date.today(), "MALL11", "FII - Malls", 85.00),
    (date.today(), "VEND11", "FII - Varejo", 75.00),
    (date.today(), "CRED11", "FII - Crédito Imobiliário", 10.00),
    (date.today(), "DEBT11", "FII - Real Estate Debt", 12.00),
    (date.today(), "JURO11", "FII - Rendimento de Juros", 10.00),
    (date.today(), "HOME11", "FII - Residencial", 95.00),
    (date.today(), "CITY11", "FII - Desenvolvimento", 80.00),
    (date.today(), "HIBR11", "FII - Híbrido", 100.00),
]
for i, row in enumerate(cadastros, start=2):
    for j, val in enumerate(row, start=1):
        ws_d.cell(row=i, column=j, value=val)

# =========================
# Aba: CARTEIRA
# =========================
ws_c = wb.create_sheet("CARTEIRA")
cols_c = ["Ativo", "Tipo", "Peso Alvo (%)", "% Anual (DY)", "Preço ref.", "Imposto div (%)", "Qtd inicial"]
for j, h in enumerate(cols_c, start=1):
    c = ws_c.cell(row=1, column=j, value=h)
    c.font = Font(bold=True)
    c.fill = PatternFill("solid", fgColor="D9E1F2")
    c.alignment = Alignment(horizontal="center")

n = len(cadastros)
for i in range(n):
    row = i + 2
    nome = cadastros[i][1]
    tipo = cadastros[i][2]
    preco = cadastros[i][3]
    ws_c.cell(row=row, column=1, value=nome)
    ws_c.cell(row=row, column=2, value=tipo)
    ws_c.cell(row=row, column=3, value=round(100/n, 4))  # peso alvo uniformizado (%)
    ws_c.cell(row=row, column=4, value=0.60)  # DY anual equivalente a ~5%/m
    ws_c.cell(row=row, column=5, value=preco)
    ws_c.cell(row=row, column=6, value=0.00)  # imposto
    ws_c.cell(row=row, column=7, value=0)     # qtd inicial

ws_c["A{row}".format(row=n+3)] = "Soma dos pesos alvo (%)"
ws_c["B{row}".format(row=n+3)] = f"=SUM(C2:C{n+1})"
ws_c["C{row}".format(row=n+3)] = "Se ≠ 100%, os pesos serão normalizados automaticamente"
ws_c["A{row}".format(row=n+3)].font = Font(bold=True)
ws_c["B{row}".format(row=n+3)].font = Font(bold=True)

# =========================
# Aba: SIMULAÇÃO (motor mensal)
# =========================
ws_s = wb.create_sheet("SIMULAÇÃO")
main_headers = [
    "Mês", "Aporte mensal", "Aporte acumulado", "Dividendos líquidos (total)",
    "Dividendos acumulados", "Patrimônio total", "Renda mensal líquida", "Yield on cost"
]
for j, h in enumerate(main_headers, start=1):
    cell = ws_s.cell(row=1, column=j, value=h)
    cell.font = Font(bold=True)
    cell.fill = PatternFill("solid", fgColor="FFF2CC")

# Campos por ativo (10 colunas por ativo)
per_fields = [
    "Preço (proj)", "Cotas", "Divid. líquidos", "Aporte alocado",
    "Caixa acumulado", "Cotas compradas (mês)", "Custo da compra (c/ corret.)", "Saldo de caixa",
    "Valor est. (cotas m-1 × preço m)", "Peso atual (m-1 est.)"
]
start_col = len(main_headers) + 1
for i in range(n):
    base = start_col + i*10
    for k in range(10):
        title = per_fields[k] if k>0 else f"[{i+1}] {per_fields[k]}"
        c = ws_s.cell(row=1, column=base+k, value=title)
        c.font = Font(bold=True)
        c.fill = PatternFill("solid", fgColor="E2EFDA")

max_months_formula = "=PARÂMETROS!B15"  # Meses simulados
max_months = 60  # 5 anos
for r in range(2, 2 + max_months):
    # Datas e aportes
    ws_s[f"A{r}"] = "=PARÂMETROS!B10" if r == 2 else f"=EDATE(A{r-1},1)"
    ws_s[f"B{r}"] = "=PARÂMETROS!B3" if r == 2 else f"=B{r-1}*(1+PARÂMETROS!B14)"
    ws_s[f"C{r}"] = "=PARÂMETROS!B2 + B2" if r == 2 else f"=C{r-1} + B{r}"
    ws_s[f"D{r}"] = 0
    ws_s[f"E{r}"] = f"=IF({r}=2,D{r},E{r-1}+D{r})"
    ws_s[f"F{r}"] = 0
    ws_s[f"G{r}"] = f"=D{r}"
    ws_s[f"H{r}"] = f"=IF(C{r}>0, D{r}/C{r}, 0)"

    # Por ativo
    for i in range(n):
        base = start_col + i*10
        cr = 2 + i
        preco_col = get_column_letter(base)
        cotas_col = get_column_letter(base+1)
        div_col = get_column_letter(base+2)
        aporte_col = get_column_letter(base+3)
        caixa_col = get_column_letter(base+4)
        cotas_comp_col = get_column_letter(base+5)
        custo_comp_col = get_column_letter(base+6)
        saldo_caixa_col = get_column_letter(base+7)
        valor_est_col = get_column_letter(base+8)
        peso_atual_col = get_column_letter(base+9)

        # Preço projetado (crescimento de preço zerado; ajustável em CARTEIRA)
        ws_s.cell(row=r, column=base).value = (
            f"=CARTEIRA!E{cr} * (1+CARTEIRA!D{cr}) ^ ((ROW()-2)/12)"
        )
        # Aporte alocado (normalizado por peso alvo em %)
        ws_s.cell(row=r, column=base+3).value = (
            f"=B{r} * (CARTEIRA!C{cr}/SUM(CARTEIRA!C2:CARTEIRA!C{n+1}))"
        )
        # Yield mensal projetado
        yield_month_formula = f"=(CARTEIRA!D{cr}/12)"
        # Dividendos líquidos
        if r == 2:
            ws_s.cell(row=r, column=base+2).value = (
                f"=(CARTEIRA!G{cr} * {preco_col}{r} * {yield_month_formula}) * (1 - CARTEIRA!F{cr})"
            )
        else:
            ws_s.cell(row=r, column=base+2).value = (
                f"=({cotas_col}{r-1} * {preco_col}{r} * {yield_month_formula}) * (1 - CARTEIRA!F{cr})"
            )
        # Caixa acumulado
        if r == 2:
            ws_s.cell(row=r, column=base+4).value = (
                f"={aporte_col}{r} + IF(PARÂMETROS!B6=\"Sim\", {div_col}{r}, 0)"
            )
        else:
            ws_s.cell(row=r, column=base+4).value = (
                f"={saldo_caixa_col}{r-1} + {aporte_col}{r} + IF(PARÂMETROS!B6=\"Sim\", {div_col}{r}, 0)"
            )
        # Valor estimado (para pesos atuais)
        if r == 2:
            ws_s.cell(row=r, column=base+8).value = f"=CARTEIRA!G{cr} * {preco_col}{r}"
        else:
            ws_s.cell(row=r, column=base+8).value = f"={cotas_col}{r-1} * {preco_col}{r}"
        denom_terms = "+".join([f"{get_column_letter(start_col + j*10 + 8)}{r}" for j in range(n)])
        ws_s.cell(row=r, column=base+9).value = f"=IF(({denom_terms})>0, {valor_est_col}{r}/({denom_terms}), 0)"
        # Compra condicionada: mínimo de ordem e subalocação
        ws_p_min_ordem = "PARÂMETROS!B18"  # será criado abaixo
        ws_s.cell(row=r, column=base+5).value = (
            f"=IF(AND({caixa_col}{r}>=IFERROR({ws_p_min_ordem},100), {peso_atual_col}{r} < CARTEIRA!C{cr}/100), "
            f"INT({caixa_col}{r} / ({preco_col}{r} * (1+PARÂMETROS!B7))), 0)"
        )
        # Custo e saldo de caixa
        ws_s.cell(row=r, column=base+6).value = f"={cotas_comp_col}{r} * {preco_col}{r} * (1+PARÂMETROS!B7)"
        ws_s.cell(row=r, column=base+7).value = f"={caixa_col}{r} - {custo_comp_col}{r}"
        # Cotas acumuladas
        if r == 2:
            ws_s.cell(row=r, column=base+1).value = f"=CARTEIRA!G{cr} + {cotas_comp_col}{r}"
        else:
            ws_s.cell(row=r, column=base+1).value = f"={cotas_col}{r-1} + {cotas_comp_col}{r}"

    # Totais
    sum_div_ranges = ",".join([f"{get_column_letter(start_col + i*10 + 2)}{r}" for i in range(n)])
    ws_s[f"D{r}"] = f"=SUM({sum_div_ranges})"
    sum_pat_terms = "+".join([f"{get_column_letter(start_col + i*10 + 1)}{r}*{get_column_letter(start_col + i*10)}{r}" for i in range(n)])
    ws_s[f"F{r}"] = f"={sum_pat_terms}"

# Larguras
ws_s.column_dimensions['A'].width = 12
for col in ['B','C','D','E','F','G','H']:
    ws_s.column_dimensions[col].width = 18

# =========================
# Aba: FLUXOS
# =========================
ws_f = wb.create_sheet("FLUXOS")
for j, h in enumerate(["Data","Aporte (negativo)","Dividendos líquidos (positivo)","Valor de saída (último mês)","Fluxo total"], start=1):
    ws_f.cell(row=1, column=j, value=h).font = Font(bold=True)

for r in range(2, 2 + max_months):
    ws_f[f"A{r}"] = f"=SIMULAÇÃO!A{r}"
    ws_f[f"B{r}"] = f"=-(PARÂMETROS!B2 + SIMULAÇÃO!B{r})" if r == 2 else f"=-SIMULAÇÃO!B{r}"
    ws_f[f"C{r}"] = f"=SIMULAÇÃO!D{r}"
    ws_f[f"D{r}"] = 0
    ws_f[f"E{r}"] = f"=B{r}+C{r}+D{r}"
last_r = 1 + max_months + 1
ws_f[f"A{last_r}"] = f"=SIMULAÇÃO!A{1+max_months}"
ws_f[f"B{last_r}"] = 0
ws_f[f"C{last_r}"] = 0
ws_f[f"D{last_r}"] = f"=SIMULAÇÃO!F{1+max_months}"
ws_f[f"E{last_r}"] = f"=D{last_r}"
ws_f["G1"] = "Taxa Interna de Retorno (XIRR)"; ws_f["G1"].font = Font(bold=True)
ws_f["G2"] = f"=XIRR(E2:E{last_r},A2:A{last_r})"

# =========================
# Aba: RELATÓRIOS (resumo por ativo)
# =========================
ws_rpt = wb.create_sheet("RELATÓRIOS")
ws_rpt["A1"] = "Resumo por ativo"; ws_rpt["A1"].font = Font(bold=True, size=14)
ws_rpt["A3"] = "Ticker"; ws_rpt["B3"] = "Cotas"; ws_rpt["C3"] = "Patrimônio"; ws_rpt["D3"] = "Dividendos (12m)"
for c in ['A','B','C','D']:
    ws_rpt[f"{c}3"].font = Font(bold=True)
for i in range(n):
    ws_rpt[f"A{4+i}"] = f"=CARTEIRA!A{2+i}"
    cotas_col_letter = get_column_letter(start_col + i*10 + 1)
    preco_col_letter = get_column_letter(start_col + i*10)
    div_col_letter = get_column_letter(start_col + i*10 + 2)
    ws_rpt[f"B{4+i}"] = f"=SIMULAÇÃO!{cotas_col_letter}{1+max_months}"
    ws_rpt[f"C{4+i}"] = f"=SIMULAÇÃO!{cotas_col_letter}{1+max_months}*SIMULAÇÃO!{preco_col_letter}{1+max_months}"
    ws_rpt[f"D{4+i}"] = f"=SUM(SIMULAÇÃO!{div_col_letter}{1+max_months-11}:SIMULAÇÃO!{div_col_letter}{1+max_months})"

# =========================
# Abas auxiliares (INVESTIMENTOS, DIVIDENDOS & SAQUES)
# =========================
ws_inv = wb.create_sheet("INVESTIMENTOS")
for j, h in enumerate(["DATA","NOME","QTD","VALOR UNIT","VALOR TOTAL"], start=1):
    ws_inv.cell(row=1, column=j, value=h).font = Font(bold=True)
ws_div = wb.create_sheet("DIVIDENDOS & SAQUES")
for j, h in enumerate(["DATA","NOME","QTD","VALOR UNIT","TOTAL","TIPO"], start=1):
    ws_div.cell(row=1, column=j, value=h).font = Font(bold=True)

# =========================
# Aba: PAINEL (KPIs e gráficos)
# =========================
ws_painel = wb.create_sheet("PAINEL")
ws_painel["A1"] = "Indicadores"; ws_painel["A1"].font = Font(bold=True, size=14)
indics = [
    ("Patrimônio final", f"=SIMULAÇÃO!F{1+max_months}"),
    ("Total aportado", f"=SIMULAÇÃO!C{1+max_months}"),
    ("Dividendos do último mês", f"=SIMULAÇÃO!D{1+max_months}"),
    ("Dividendos médios (12 últimos)", f"=AVERAGE(SIMULAÇÃO!D{1+max_months-11}:SIMULAÇÃO!D{1+max_months})"),
    ("Dividendos acumulados", f"=SIMULAÇÃO!E{1+max_months}"),
    ("Yield on cost (último mês)", f"=SIMULAÇÃO!H{1+max_months}"),
    ("XIRR", "=FLUXOS!G2"),
    ("Meta de renda mensal (10% do salário)", "=PARÂMETROS!B12"),
    ("Percentual da meta (último mês)", f"=IF(PARÂMETROS!B12>0, SIMULAÇÃO!D{1+max_months}/PARÂMETROS!B12, 0)"),
]
for i, (lbl, formula) in enumerate(indics, start=2):
    ws_painel[f"A{i}"] = lbl
    ws_painel[f"B{i}"] = formula
    ws_painel[f"A{i}"].font = Font(bold=True)

# Série meta constante (para gráfico Renda vs Meta)
ws_painel["D2"] = "Meta (constante)"; ws_painel["D2"].font = Font(bold=True)
for r in range(3, 3 + max_months):
    ws_painel[f"D{r}"] = "=PARÂMETROS!B12"

# Gráfico: Patrimônio
line_chart = LineChart(); line_chart.title = "Patrimônio total ao longo do tempo"
line_chart.y_axis.title = "Patrimônio (moeda base)"; line_chart.x_axis.title = "Mês"
data_ref = Reference(ws_s, min_col=6, min_row=1, max_row=1+max_months)
cats = Reference(ws_s, min_col=1, min_row=2, max_row=1+max_months)
line_chart.add_data(data_ref, titles_from_data=True); line_chart.set_categories(cats)
line_chart.height = 12; line_chart.width = 26
ws_painel.add_chart(line_chart, "A10")

# Gráfico: Renda vs Meta
renda_chart = LineChart(); renda_chart.title = "Renda mensal líquida vs Meta"
renda_chart.y_axis.title = "Renda (BRL)"; renda_chart.x_axis.title = "Mês"
div_ref = Reference(ws_s, min_col=4, min_row=1, max_row=1+max_months)
meta_ref = Reference(ws_painel, min_col=4, min_row=2, max_row=2+max_months)
renda_chart.add_data(div_ref, titles_from_data=True)
renda_chart.add_data(meta_ref, titles_from_data=True)
renda_chart.set_categories(cats)
renda_chart.height = 12; renda_chart.width = 26
ws_painel.add_chart(renda_chart, "A26")

# Gráfico: Alocação alvo (pizza)
pie_chart = PieChart(); pie_chart.title = "Alocação por peso alvo"
labels = Reference(ws_c, min_col=1, min_row=2, max_row=n+1)
values = Reference(ws_c, min_col=3, min_row=2, max_row=n+1)
pie_chart.add_data(values, titles_from_data=False); pie_chart.set_categories(labels)
pie_chart.height = 12; pie_chart.width = 20
ws_painel.add_chart(pie_chart, "Q10")

# Gráfico: Dividendos por ativo (12m)
bar_div = BarChart(); bar_div.title = "Dividendos por ativo (12 meses)"; bar_div.y_axis.title = "Dividendos (BRL)"
labels2 = Reference(ws_rpt, min_col=1, min_row=4, max_row=3+n)
values2 = Reference(ws_rpt, min_col=4, min_row=4, max_row=3+n)
bar_div.add_data(values2, titles_from_data=False); bar_div.set_categories(la[Simulador_FII_5_Anos_Painel.xlsx](https://github.com/user-attachments/files/24439993/Simulador_FII_5_Anos_Painel.xlsx)
bels2)
bar_div.height = 12; bar_div.width = 20
ws_painel.add_chart(bar_div, "Q26")

# Bordas nos cabeçalhos
for ws in [ws_p, ws_d, ws_c, ws_s, ws_f, ws_rpt, ws_inv, ws_div, ws_painel]:
    for row in ws.iter_rows(min_row=1, max_row=ws.max_row, min_col=1, max_col=ws.max_column):
        for cell in row:
            if cell.row == 1:
                cell.border = border

# Largura das colunas em CARTEIRA
widths = [16, 22, 16, 18, 18, 20, 18]
for j, w in enumerate(widths, start=1):
    ws_c.column_dimensions[get_column_letter(j)].width = w

# Parâmetro extra (mínimo de ordem) em PARÂMETROS (B18)
ws_p["A18"] = "Mínimo de ordem por ativo (BRL)"; ws_p["B18"] = 100; ws_p["C18"] = "Compra somente se caixa ≥ este valor"


wb.save(filename)
print(filename)
filename = "Simulador_FII_5_Anos_Painel_GERADO.xlsx"[Simulador_FII_5_Anos_Painel.xlsx](https://github.com/user-attachments/files/24439999/Simulador_FII_5_Anos_Painel.xlsx)
