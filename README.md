import streamlit as st
import pandas as pd
import folium
from streamlit_folium import st_folium
import plotly.express as px

# PROFIL UTILISATEUR

def user_login():
    """Système simple de connexion utilisateur avec vérification e-mail"""
    if "user_email" not in st.session_state:
        st.session_state["user_email"] = None

    st.sidebar.subheader("📧 Connexion par e-mail")

    if not st.session_state["user_email"]:
        email_input = st.sidebar.text_input("Votre adresse e-mail :")

        if st.sidebar.button("Se connecter"):
            if "@" in email_input and "." in email_input:
                st.session_state["user_email"] = email_input
                st.sidebar.success(f"Bienvenue {email_input} 👋")
            else:
                st.sidebar.error("❌ Veuillez entrer une adresse e-mail valide (ex: nom@domaine.com)")
    else:
        st.sidebar.info(f"Connecté en tant que {st.session_state['user_email']}")
        if st.sidebar.button("Se déconnecter"):
            st.session_state["user_email"] = None
            st.sidebar.success("Déconnexion réussie.")



# FONCTIONS


@st.cache_data
def load_data(csv_path):
    """Charge le fichier CSV des ONG"""
    df = pd.read_csv(csv_path)
    return df

def filter_data(df, country, domains, search_name):
    """Filtre les données selon pays, domaine et recherche"""
    filtered = df.copy()
    if country != "Tous":
        filtered = filtered[filtered["pays"] == country]
    if domains:
        filtered = filtered[filtered["domaine"].isin(domains)]
    if search_name:
        filtered = filtered[filtered["nom"].str.contains(search_name, case=False, na=False)]
    return filtered

def create_map(df):
    """Crée la carte interactive Folium"""
    m = folium.Map(location=[10, 20], zoom_start=2.3, tiles="cartodb positron")

    for _, row in df.iterrows():
        popup_html = f"""
        <b>{row['nom']}</b><br>
        <i>{row['domaine']}</i> — {row['pays']}<br>
        <a href="{row['site_web']}" target="_blank">🌐 Site officiel</a>
        """
        folium.Marker(
            [row["latitude"], row["longitude"]],
            popup=popup_html,
            tooltip=row["nom"],
            icon=folium.Icon(color="blue", icon="info-sign")
        ).add_to(m)
    return m

def show_theme_analysis(df):
    st.subheader("🎯 Associations par thématique")

    themes = sorted(df["domaine"].dropna().unique())
    selected_theme = st.selectbox("Choisissez un thème :", themes)

    theme_df = df[df["domaine"] == selected_theme]

    st.markdown(f"### {len(theme_df)} associations dans le domaine **{selected_theme}**")

    for _, row in theme_df.iterrows():
        st.markdown(f"""
        **{row['nom']}**  
        🌍 *{row['pays']}*  
        🧩 **Domaine :** {row['domaine']}  
        🔗 [Site officiel]({row['site_web']})  
        """)
        st.markdown("---")



#  PAGE DE DONS

def donation_page(df):
    """Page de dons avec enregistrement dans l'historique"""
    st.subheader("💰 Soutenir une association")

    # Vérification du profil utilisateur
    user = st.session_state.get("user_email", None)
    if not user:
        st.warning("👤 Veuillez vous connecter dans la barre latérale avant de faire un don.")
        return

    # Sélection d'une ONG
    ong_names = sorted(df["nom"].unique().tolist())
    selected_ong = st.selectbox("Choisissez une association :", ong_names)

    # Informations sur l'ONG sélectionnée
    ong_info = df[df["nom"] == selected_ong].iloc[0]
    st.write(f"**Pays :** {ong_info['pays']}")
    st.write(f"**Domaine :** {ong_info['domaine']}")
    st.write(f"**Site officiel :** [{ong_info['site_web']}]({ong_info['site_web']})")

    # Champ de saisie du montant
    montant = st.number_input("Montant du don (€)", min_value=1, step=5)

    # Initialisation de l'historique des dons si nécessaire
    if "historique_dons" not in st.session_state:
        st.session_state["historique_dons"] = []

    # Bouton d’action
    if st.button("🌍 Faire un don maintenant"):
        don = {
            "utilisateur": user,
            "association": selected_ong,
            "montant": montant,
            "domaine": ong_info["domaine"],
            "date": pd.Timestamp.now().strftime("%d/%m/%Y")
        }
        st.session_state["historique_dons"].append(don)

        st.success(f"Merci {user} pour votre don de {montant} € à **{selected_ong}** ❤️")
        st.balloons()
        st.markdown(f"[👉 Cliquez ici pour finaliser votre don sur le site officiel de {selected_ong}]({ong_info['site_web']})")

    # Ligne séparatrice et affichage du total
    st.markdown("---")
    total_dons = sum(d["montant"] for d in st.session_state["historique_dons"])
    st.metric("Total des dons simulés sur cette session", f"{total_dons} €")
    
     # 💾 Sauvegarde de l'historique dans un fichier CSV
    dons_df = pd.DataFrame(st.session_state["historique_dons"])
    dons_df.to_csv("historique_dons.csv", index=False)


# HISTORIQUE DES DONS

def show_don_history(df):
    st.subheader("🧾 Historique de mes dons")

    if "historique_dons" not in st.session_state or len(st.session_state["historique_dons"]) == 0:
        st.info("Aucun don enregistré pour le moment.")
        return

    dons_df = pd.DataFrame(st.session_state["historique_dons"])
    total = dons_df["montant"].sum()
    st.metric("Total des dons", f"{total} €")
    st.dataframe(dons_df[["date", "association", "montant", "domaine"]])

    #  Nouveau graphique Plotly 
    st.subheader("📈 Répartition des dons par domaine")
    fig = px.bar(
        dons_df,
        x="domaine",
        y="montant",
        color="domaine",
        title="Montant total des dons par domaine",
        text_auto=True
    )
    st.plotly_chart(fig, use_container_width=True)

# APPLICATION PRINCIPALE

def main():
    st.set_page_config(page_title="ONG Explorer 2.0", layout="wide")

    st.title("ONG Explorer 2.0 — Une vue d'ensemble sur les actions humanitaires mondiales")

    df = load_data("bdd_ong.csv")
    user_login()

    # Onglets de navigation
    tabs = st.tabs([
        "📍 Carte interactive",
        "🎯 Par thématique",
        "💰 Faire un don",
        "🧾 Mes dons",
    ])

    #  Onglet Carte 
    with tabs[0]:
        st.sidebar.header("🧭 Filtres")

        countries = ["Tous"] + sorted(df["pays"].unique().tolist())
        selected_country = st.sidebar.selectbox("Choisir un pays", countries)

        domains = sorted(df["domaine"].unique().tolist())
        selected_domains = st.sidebar.multiselect("Choisir un ou plusieurs domaines", domains)

        search_name = st.sidebar.text_input("🔍 Rechercher une ONG")

        filtered_df = filter_data(df, selected_country, selected_domains, search_name)
        st.sidebar.write(f"**{len(filtered_df)} ONG trouvées**")

        # ✅ Conversion des coordonnées en float (pour éviter l'erreur)
        filtered_df["latitude"] = pd.to_numeric(filtered_df["latitude"], errors="coerce")
        filtered_df["longitude"] = pd.to_numeric(filtered_df["longitude"], errors="coerce")

        # ✅ Suppression des lignes sans coordonnées valides
        filtered_df = filtered_df.dropna(subset=["latitude", "longitude"])

        st.subheader("Carte des ONG dans le monde")
        m = create_map(filtered_df)
        st_folium(m, width=900, height=500)

        with st.expander("Voir les données filtrées"):
            st.dataframe(filtered_df)

    # ---- Onglet Analyse par pays ----
    with tabs[1]:
        show_theme_analysis(df)

    # ---- Onglet Faire un don ----
    with tabs[2]:
        donation_page(df)

    # ---- Onglet Mes dons ----
    with tabs[3]:
        show_don_history(df)

# LANCEMENT

if __name__ == "__main__":
    main()

    
