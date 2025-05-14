import pandas as pd
import pymssql
import streamlit as st
import seaborn as sns
import matplotlib.pyplot as plt
import numpy as np


st.set_page_config(
    page_title="Análise Raia",
    layout="wide",
    initial_sidebar_state="collapsed",
    menu_items={'About': "# Desenvolvido por F4"}
)
# =================== FUNÇÕES DE BUSCA ===================

@st.cache_data
def fetch_limab_data(trend_type):
    conn = pymssql.connect(
        server='TNABRWDW001',
        user='userF4Processo',
        password='F4Process',
        database='F4_Level2_CFTbWM3'
    )
    query = f"""
    SELECT PipeNumber, TrendTypeCode, InsDateTime, Position, Value
    FROM [Limab].[VwNdtTrendHistory]
    WHERE TrendTypeCode = '{trend_type}'
    """
    df = pd.read_sql(query, conn)
    conn.close()

    df['InsDateTime'] = pd.to_datetime(df['InsDateTime']).dt.tz_localize(None)
    return df

@st.cache_data
def fetchTFCPipes():
    conn = pymssql.connect(
        server=r'172.18.23.4',
        user=r'userMAIN',
        password='userMAIN',
        database='TFCWelded2_DW'
    )
    query = """
    SELECT * 
    FROM [dbo].[INPL_View_Pipes_and_Tubes] 
    WHERE 
        FABRICA = 'UOE' 
        AND (STATION = 'Grotnes' OR STATION = 'SMS') 
        AND TYPE = 'TUBE' 
        AND PRODUCTION_ORDER IN ('5006701', '5006703', '5006705', '5006707')
    ORDER BY INITIAL_DATE
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# =================== TRATAMENTO E GRÁFICOS ===================

def convert_perimeter_to_diameter(perimeter):
    try:
        return float(perimeter) / np.pi
    except:
        return np.nan

def get_combined_data(df_limab, df_tfc):
    if 'PipeNumber' not in df_limab or 'TUBE_NAME' not in df_tfc:
        return pd.DataFrame()
    return pd.merge(df_limab, df_tfc, left_on='PipeNumber', right_on='TUBE_NAME', how='inner')

def filter_extremidade(df, extremidade):
    if extremidade == "Ponta 1":
        return df.loc[df.groupby('PipeNumber')['Position'].idxmin()]
    elif extremidade == "Ponta 2":
        return df.loc[df.groupby('PipeNumber')['Position'].idxmax()]
    else:  # Meio
        return df[(df['Position'] >= 50) & (df['Position'] <= 11000)]

def plot_diametro_distribution(df, production_order, diameter_mm, extremidade):
    df_filtered = df[df['PRODUCTION_ORDER'] == production_order]
    df_filtered = filter_extremidade(df_filtered, extremidade)
    df_filtered['Diameter'] = df_filtered['Value'].apply(convert_perimeter_to_diameter)
    valid = pd.to_numeric(df_filtered['Diameter'], errors='coerce').dropna()

    if not valid.empty:
        sns.set(style="whitegrid")
        plt.figure(figsize=(10, 6))
        sns.histplot(valid, kde=True, stat="density", bins=30, color='blue')
        plt.axvline(diameter_mm, color='red', linestyle='--', label=f'Diâmetro Nominal: {diameter_mm:.1f} mm')
        plt.title(f'Distribuição de Diâmetro - PO {production_order} - {extremidade}')
        plt.xlabel('Diâmetro (mm)')
        plt.ylabel('Densidade')
        plt.legend()
        st.pyplot(plt)
    else:
        st.warning(f"Sem dados válidos para o diâmetro em {extremidade} - PO {production_order}")

def plot_ovalizacao_distribution(df, extremidade):
    df_filtered = filter_extremidade(df, extremidade)
    valid = pd.to_numeric(df_filtered['Value'], errors='coerce').dropna()

    if not valid.empty:
        sns.set(style="whitegrid")
        plt.figure(figsize=(10, 6))
        sns.histplot(valid, kde=True, stat="density", bins=30, color='green')
        plt.title(f'Distribuição de Ovalização - {extremidade}')
        plt.xlabel('Ovalização (mm)')
        plt.ylabel('Densidade')
        st.pyplot(plt)
    else:
        st.warning(f"Sem dados válidos de ovalização para {extremidade}")

# =================== STREAMLIT INTERFACE ===================

st.title("Análise de Diâmetro e Ovalização dos Tubos")

# Seleção da extremidade para os dois gráficos
extremidade = st.selectbox("Selecione a região do tubo para análise:", ("Ponta 1", "Ponta 2", "Meio"))

# Seleção de PO
production_order_diameters = {
    '5006701': 577.900,
    '5006703': 568.700,
    '5006705': 558.800,
    '5006707': 609.600
}

production_order = st.radio("Escolha um Production Order:", options=list(production_order_diameters.keys()))
diameter_mm = production_order_diameters[production_order]


# Carregar dados
df_limab_per = fetch_limab_data('PER')
df_limab_ova = fetch_limab_data('OVA')
df_tfc = fetchTFCPipes()

# Gerar combinados
df_combined_per = get_combined_data(df_limab_per, df_tfc)
df_combined_ova = get_combined_data(df_limab_ova, df_tfc)

# Plots
if not df_combined_per.empty and not df_combined_ova.empty:
    st.subheader("Distribuição do Diâmetro (convertido do Perímetro)")
    plot_diametro_distribution(df_combined_per, production_order, diameter_mm, extremidade)

    st.subheader("Distribuição da Ovalização")
    plot_ovalizacao_distribution(df_combined_ova, extremidade)
else:
    st.warning("Dados insuficientes para gerar os gráficos.")
