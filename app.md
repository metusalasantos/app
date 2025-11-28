# Install streamlit if not already installed
!pip install streamlit

import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import seaborn as sns
import matplotlib.pyplot as plt

# Configuração da Página
st.set_page_config(page_title="Olist Storytelling", layout="wide", page_icon="📦")

# Estilização Customizada
st.markdown("""
<style>
    .main-header {
        font-size: 3rem;
        color: #FF9900;
        text-align: center;
        font-weight: bold;
    }
    .sub-header {
        font-size: 1.5rem;
        color: #4B4B4B;
    }
    .insight-box {
        background-color: #f0f2f6;
        border-left: 5px solid #FF9900;
        padding: 20px;
        border-radius: 5px;
        margin: 10px 0;
    }
    .big-number {
        font-size: 2rem;
        font-weight: bold;
        color: #2E86C1;
    }
</style>
""", unsafe_allow_html=True)

# --- 1. CARREGAMENTO E PREPARAÇÃO DOS DADOS ---
@st.cache_data
def load_data():
    # Carregando datasets essenciais
    orders = pd.read_csv('olist_orders_dataset.csv')
    items = pd.read_csv('olist_order_items_dataset.csv')
    products = pd.read_csv('olist_products_dataset.csv')
    payments = pd.read_csv('olist_order_payments_dataset.csv')
    reviews = pd.read_csv('olist_order_reviews_dataset.csv')
    customers = pd.read_csv('olist_customers_dataset.csv')

    # Conversão de datas
    date_cols = ['order_purchase_timestamp', 'order_approved_at',
                 'order_delivered_carrier_date', 'order_delivered_customer_date',
                 'order_estimated_delivery_date']
    for col in date_cols:
        orders[col] = pd.to_datetime(orders[col])

    # Merge Mestre (Joining tables para análise centralizada)
    # Orders + Items
    df = orders.merge(items, on='order_id', how='left')
    # + Products
    df = df.merge(products, on='product_id', how='left')
    # + Customers
    df = df.merge(customers, on='customer_id', how='left')
    # + Reviews
    df = df.merge(reviews, on='order_id', how='left')

    return df, orders, items, payments, reviews, products, customers

try:
    with st.spinner('Carregando e processando os dados da Olist...'):
        df, orders, items, payments, reviews, products, customers = load_data()
except Exception as e:
    st.error(f"Erro ao carregar arquivos. Certifique-se de que todos os CSVs estão no mesmo diretório. Erro: {e}")
    st.stop()

# --- HEADER ---
st.markdown('<div class="main-header">O E-commerce Brasileiro em Números</div>', unsafe_allow_html=True)
st.markdown('<div style="text-align: center; margin-bottom: 30px;">Uma Análise Exploratória e Storytelling com dados da Olist</div>', unsafe_allow_html=True)

# Métricas Rápidas
col1, col2, col3, col4 = st.columns(4)
col1.metric("Total de Pedidos", f"{orders.shape[0]:,}".replace(",", "."))
col2.metric("Receita Total (Produtos)", f"R$ {items['price'].sum():,.2f}".replace(",", "X").replace(".", ",").replace("X", "."))
col3.metric("Ticket Médio", f"R$ {items['price'].mean():.2f}".replace(".", ","))
col4.metric("Cidades Atendidas", f"{customers['customer_city'].nunique()}")

st.markdown("---")

# --- CAPÍTULO 1: A EVOLUÇÃO TEMPORAL ---
st.header("1. O Pulso das Vendas: Sazonalidade e Crescimento")

# Preparando dados temporais
orders['month_year'] = orders['order_purchase_timestamp'].dt.to_period('M')
sales_per_month = df.groupby(df['order_purchase_timestamp'].dt.to_period('M'))['price'].sum().reset_index()
sales_per_month['order_purchase_timestamp'] = sales_per_month['order_purchase_timestamp'].dt.to_timestamp()

col_chart, col_txt = st.columns([2, 1])

with col_chart:
    fig_time = px.line(sales_per_month, x='order_purchase_timestamp', y='price',
                       title='Evolução da Receita Mensal',
                       labels={'order_purchase_timestamp': 'Data', 'price': 'Receita (R$)'},
                       markers=True)
    fig_time.add_vrect(x0="2017-11-01", x1="2017-11-30", annotation_text="Black Friday 2017", annotation_position="top left", fillcolor="green", opacity=0.1, line_width=0)
    st.plotly_chart(fig_time, use_container_width=True)

with col_txt:
    st.markdown("""
    ### O Efeito Black Friday

    Observe o pico massivo em **Novembro de 2017**. Este é o efeito clássico da **Black Friday** no varejo brasileiro.

    * **Crescimento Constante:** Existe uma tendência clara de alta ao longo de 2017 e 2018.
    * **Queda Abrupta:** O gráfico cai no final não por falta de vendas, mas porque o dataset se encerra em Setembro/Outubro de 2018.
    """)
    st.markdown('<div class="insight-box">💡 <strong>Insight:</strong> A preparação de estoque e logística deve começar pelo menos 3 meses antes de novembro para suportar o pico de demanda.</div>', unsafe_allow_html=True)

st.markdown("---")

# --- CAPÍTULO 2: GEOGRAFIA ---
st.header("2. A Geografia Desigual do E-commerce")

col_geo1, col_geo2 = st.columns(2)

# Mapa de calor por estado
orders_by_state = customers['customer_state'].value_counts().reset_index()
orders_by_state.columns = ['Estado', 'Pedidos']

with col_geo1:
    fig_map = px.choropleth(orders_by_state,
                            locations="Estado",
                            locationmode="country names", # Olist usa siglas BR, Plotly entende bem com geojson customizado, mas usaremos barplot para simplicidade robusta aqui ou scattergeo
                            scope="south america",
                            color="Pedidos",
                            color_continuous_scale="Oranges")
    # Alternativa mais robusta sem GeoJSON externo: Gráfico de Barras
    fig_bar_state = px.bar(orders_by_state, x='Estado', y='Pedidos',
                           title="Pedidos por Estado (Concentração)",
                           color='Pedidos', color_continuous_scale='OrRd')
    st.plotly_chart(fig_bar_state, use_container_width=True)

with col_geo2:
    st.markdown("""
    ### A Hegemonia do Sudeste

    O gráfico não deixa dúvidas: **São Paulo (SP)** é o motor do e-commerce brasileiro neste dataset, seguido por Rio de Janeiro (RJ) e Minas Gerais (MG).

    Isso cria um **desafio logístico**:
    1.  Vender para o Sudeste é barato e rápido.
    2.  Vender para o Norte/Nordeste é caro e demorado.
    """)

    # Frete Médio por Estado
    freight_state = df.groupby('customer_state')['freight_value'].mean().sort_values(ascending=False).head(5).reset_index()
    st.write("**Estados com Frete Médio Mais Caro:**")
    st.dataframe(freight_state.style.format({"freight_value": "R$ {:.2f}"}), hide_index=True)

st.markdown('<div class="insight-box">💡 <strong>Insight Logístico:</strong> Clientes do Norte (PB, RR, RO) pagam fretes significativamente mais altos, o que pode ser uma barreira de entrada. Estratégias de subsídio de frete para essas regiões podem aumentar a conversão.</div>', unsafe_allow_html=True)

st.markdown("---")

# --- CAPÍTULO 3: PRODUTOS ---
st.header("3. O Que Estamos Comprando?")

top_categories = df['product_category_name'].value_counts().head(10).reset_index()
top_categories.columns = ['Categoria', 'Qtd Vendas']

fig_cat = px.bar(top_categories, x='Qtd Vendas', y='Categoria', orientation='h',
                 title='Top 10 Categorias Mais Vendidas',
                 color='Qtd Vendas', color_continuous_scale='Blues')
fig_cat.update_layout(yaxis={'categoryorder':'total ascending'})

st.plotly_chart(fig_cat, use_container_width=True)

st.markdown("""
A categoria **"cama_mesa_banho"** lidera, seguida por **"beleza_saude"** e **"esporte_lazer"**.
Curiosamente, categorias de tickets altos como "informatica_acessorios" também têm alto volume, indicando confiança do consumidor em comprar eletrônicos online.
""")

st.markdown("---")

# --- CAPÍTULO 4: A DOR DO CLIENTE (REVIEW vs ENTREGA) ---
st.header("4. O Vilão da Reputação: Atrasos na Entrega")

st.markdown("Aqui está a análise mais crítica. Cruzamos o tempo de entrega com a nota dada pelo cliente (1 a 5).")

# Calculando tempo de entrega real
df_delivered = df[df['order_status'] == 'delivered'].copy()
df_delivered['delivery_days'] = (df_delivered['order_delivered_customer_date'] - df_delivered['order_purchase_timestamp']).dt.days

# Agrupando tempo médio por Score
avg_time_score = df_delivered.groupby('review_score')['delivery_days'].mean().reset_index()

col_rev1, col_rev2 = st.columns([2, 1])

with col_rev1:
    fig_rev = px.bar(avg_time_score, x='review_score', y='delivery_days',
                     title='Tempo Médio de Entrega (dias) vs. Nota da Avaliação',
                     color='delivery_days', color_continuous_scale='RdYlGn_r',
                     labels={'review_score': 'Nota (Estrelas)', 'delivery_days': 'Dias para Entregar'})
    st.plotly_chart(fig_rev, use_container_width=True)

with col_rev2:
    st.markdown("### A Correlação Fatal")
    st.markdown("""
    Olhe para o gráfico:
    * **5 Estrelas:** Média de ~10 a 11 dias de entrega.
    * **1 Estrela:** Média salta para **mais de 20 dias**.

    A paciência do consumidor tem limite. Passou de 2 semanas, a chance de uma avaliação negativa dispara.
    """)

# Comentários negativos mais comuns (Análise simples de texto)
st.markdown("#### O que dizem os clientes insatisfeitos (Score 1)?")
bad_reviews = df[df['review_score'] == 1]['review_comment_message'].dropna()
st.text_area("Amostra de reclamações reais:", "\n".join(bad_reviews.head(3).values), height=150)

st.markdown('<div class="insight-box">💡 <strong>Insight Crítico:</strong> Logística não é apenas "entregar o pacote", é "proteger a marca". Reduzir o tempo de entrega em 20% poderia aumentar o NPS (Net Promoter Score) drasticamente.</div>', unsafe_allow_html=True)

st.markdown("---")

# --- CAPÍTULO 5: PAGAMENTOS ---
st.header("5. Como Pagamos?")

payment_types = payments['payment_type'].value_counts().reset_index()
payment_types.columns = ['Tipo', 'Total']

col_pay1, col_pay2 = st.columns(2)

with col_pay1:
    fig_pie = px.pie(payment_types, values='Total', names='Tipo', hole=0.4,
                     title='Distribuição dos Meios de Pagamento',
                     color_discrete_sequence=px.colors.sequential.RdBu)
    st.plotly_chart(fig_pie, use_container_width=True)

with col_pay2:
    installments = payments[payments['payment_type'] == 'credit_card']['payment_installments'].value_counts().head(5)
    st.write("**Parcelamento no Cartão de Crédito (Top 5):**")
    st.bar_chart(installments)
    st.caption("A maioria paga em 1x, mas o parcelamento em até 10x é muito comum.")

st.markdown("---")

# --- CONCLUSÃO ---
st.markdown("""
<div class="insight-box" style="background-color: #e8f4f8; border-left: 5px solid #2E86C1;">
    <h3>🥡 Conclusões Estratégicas</h3>
    <ol>
        <li><strong>Logística é o Core Business:</strong> A satisfação do cliente é inversamente proporcional ao tempo de entrega. Atrasos matam a retenção.</li>
        <li><strong>O Brasil é Grande:</strong> A estratégia de frete precisa ser regionalizada. O modelo "frete único" prejudica a conversão no Norte/Nordeste.</li>
        <li><strong>Calendário Comercial:</strong> A Black Friday é o evento do ano. O estoque de "Cama, Mesa e Banho" precisa estar preparado para novembro.</li>
        <li><strong>Crédito é Rei:</strong> Quase 75% das vendas dependem de cartão de crédito. Oferecer parcelamento sem juros é uma alavanca de vendas poderosa.</li>
    </ol>
</div>
""", unsafe_allow_html=True)
