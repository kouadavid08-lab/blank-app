# 🎈 Blank app template

A simple Streamlit app template for you to modify!

[![Open in Streamlit](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://blank-app-template.streamlit.app/)

### How to run it on your own machine

1. Install the requirements

   ```
   $ pip install -r requirements.txt
   ```

2. Run the app

   ```
   $ streamlit run streamlit_app.py
import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go

# ===================== CONFIG PAGE =====================
st.set_page_config(
    page_title="Analyseur Foot Pro v2.1",
    page_icon="⚽",
    layout="wide"
)

# ===================== STYLE =====================
st.markdown("""
<style>
.main { background-color: #f5f7f9; }
.stMetric {
    background-color: white;
    padding: 15px;
    border-radius: 10px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}
</style>
""", unsafe_allow_html=True)

# ===================== DATA =====================
def load_data():
    data = {
        'Equipe': ['PSG', 'Marseille', 'Lyon', 'Lille', 'Monaco', 'Lens', 'Rennes', 'Nice'],
        'Buts_Marqués': [45, 32, 28, 30, 35, 29, 27, 24],
        'Buts_Encaissés': [12, 25, 30, 22, 28, 20, 24, 18],
        'Possession_Moy': [62, 54, 51, 53, 55, 49, 52, 50],
        'Tirs_Par_Match': [15.5, 12.1, 11.8, 13.0, 14.2, 11.5, 12.3, 10.9],
        'Passes_Reussies': [89, 82, 80, 83, 81, 78, 80, 84]
    }
    df = pd.DataFrame(data)
    df['Diff_Buts'] = df['Buts_Marqués'] - df['Buts_Encaissés']
    return df

df = load_data()

# ===================== SIDEBAR =====================
st.sidebar.image("https://img.icons8.com/color/96/football.png", width=80)
st.sidebar.title("Dashboard Foot")
st.sidebar.markdown("---")

equipes = df['Equipe'].unique()
equipes_sel = st.sidebar.multiselect(
    "Comparer les équipes :",
    options=equipes,
    default=['PSG', 'Marseille']
)

df_filtre = df[df['Equipe'].isin(equipes_sel)]

# ===================== MAIN =====================
st.title("⚽ Analyseur de Performance Football")
st.write(f"Analyse comparative de **{len(equipes_sel)}** équipes de Ligue 1")

if not equipes_sel:
    st.warning("Veuillez sélectionner au moins une équipe.")
    st.stop()

# ===================== KPIs =====================
c1, c2, c3, c4 = st.columns(4)

best_attack = df_filtre.loc[df_filtre['Buts_Marqués'].idxmax()]
best_defense = df_filtre.loc[df_filtre['Buts_Encaissés'].idxmin()]

with c1:
    st.metric("Meilleure Attaque", best_attack['Equipe'])
    st.caption(f"{best_attack['Buts_Marqués']} buts marqués")

with c2:
    st.metric("Meilleure Défense", best_defense['Equipe'])
    st.caption(f"{best_defense['Buts_Encaissés']} buts encaissés")

with c3:
    st.metric("Possession Max", f"{df_filtre['Possession_Moy'].max()}%")

with c4:
    st.metric("Tirs / Match (moy)", round(df_filtre['Tirs_Par_Match'].mean(), 1))

st.markdown("---")

# ===================== TABS =====================
tab1, tab2, tab3 = st.tabs(["📊 Statistiques", "🕸️ Radar", "🎲 Simulateur"])

# ---------- TAB 1 : STATS ----------
with tab1:
    c1, c2 = st.columns(2)

    # Graphique buts (format long)
    df_long = df_filtre.melt(
        id_vars='Equipe',
        value_vars=['Buts_Marqués', 'Buts_Encaissés'],
        var_name='Type',
        value_name='Buts'
    )

    fig_buts = px.bar(
        df_long,
        x='Equipe',
        y='Buts',
        color='Type',
        barmode='group',
        title="Bilan Offensif & Défensif"
    )
    c1.plotly_chart(fig_buts, use_container_width=True)

    fig_scatter = px.scatter(
        df_filtre,
        x='Possession_Moy',
        y='Tirs_Par_Match',
        size='Buts_Marqués',
        color='Equipe',
        title="Style de jeu : Possession vs Tirs"
    )
    c2.plotly_chart(fig_scatter, use_container_width=True)

    st.subheader("📋 Classement (Différence de buts)")
    st.dataframe(
        df_filtre.sort_values('Diff_Buts', ascending=False),
        use_container_width=True
    )

# ---------- TAB 2 : RADAR ----------
with tab2:
    st.subheader("Profil tactique comparé")

    categories = ['Buts_Marqués', 'Possession_Moy', 'Tirs_Par_Match', 'Passes_Reussies']

    def normalize(series):
        return (series - series.min()) / (series.max() - series.min()) * 100

    df_norm = df.copy()
    for col in categories:
        df_norm[col] = normalize(df[col])

    fig_radar = go.Figure()

    for eq in equipes_sel:
        stats = df_norm[df_norm['Equipe'] == eq][categories].values.flatten()
        fig_radar.add_trace(go.Scatterpolar(
            r=stats,
            theta=categories,
            fill='toself',
            name=eq
        ))

    fig_radar.update_layout(
        polar=dict(radialaxis=dict(visible=True, range=[0, 100])),
        showlegend=True
    )

    st.plotly_chart(fig_radar, use_container_width=True)

# ---------- TAB 3 : SIMULATEUR ----------
with tab3:
    st.subheader("Simulation de match")

    if len(equipes_sel) < 2:
        st.info("Sélectionnez au moins deux équipes.")
        st.stop()

    eq1 = st.selectbox("🏠 Équipe Domicile", equipes_sel, index=0)
    eq2 = st.selectbox("✈️ Équipe Extérieure", equipes_sel, index=1)

    if eq1 == eq2:
        st.error("Veuillez choisir deux équipes différentes.")
        st.stop()

    force1 = df.loc[df['Equipe'] == eq1, 'Diff_Buts'].values[0]
    force2 = df.loc[df['Equipe'] == eq2, 'Diff_Buts'].values[0]

    diff = force1 - force2

    prob1 = max(5, min(80, 33 + diff * 2))
    prob2 = max(5, min(80, 33 - diff * 2))
    probN = max(5, 100 - prob1 - prob2)

    p1, p2, p3 = st.columns(3)
    p1.info(f"Victoire {eq1} : {round(prob1)}%")
    p2.info(f"Match nul : {round(probN)}%")
    p3.info(f"Victoire {eq2} : {round(prob2)}%")

    if force1 > force2:
        st.success(f"🔮 {eq1} est favori")
    elif force2 > force1:
        st.warning(f"🔮 {eq2} est favori")
    else:
        st.info("🔮 Match équilibré")

    st.caption("Simulation basée sur la différence de buts")

# ===================== FOOTER =====================
st.sidebar.markdown("---")
st.sidebar.caption("Analyseur Foot Pro • Streamlit + Plotly")
   ```
