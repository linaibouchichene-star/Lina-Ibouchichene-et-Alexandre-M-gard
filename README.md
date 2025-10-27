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


BASE DE DONNEES :
nom,pays,domaine,latitude,longitude,site_web
Médecins Sans Frontières,Mali,Santé,12.6392,-8.0029,https://www.msf.org
Save the Children,Éthiopie,Éducation,9.145,40.4897,https://www.savethechildren.net
CARE International,Niger,Urgence,17.6078,8.0817,https://www.care.org
Action Contre la Faim,Sénégal,Nutrition,14.4974,-14.4524,https://www.actioncontrelafaim.org
Plan International,Cameroun,Éducation,7.3697,12.3547,https://plan-international.org
Oxfam,Burkina Faso,Développement,12.2383,-1.5616,https://www.oxfam.org
World Vision,Rwanda,Éducation,-1.9403,29.8739,https://www.wvi.org
Handicap International,Maroc,Santé,31.7917,-7.0926,https://www.handicap-international.org
SOS Villages d’Enfants,Côte d’Ivoire,Enfance,7.5399,-5.5471,https://www.sos-childrensvillages.org
WaterAid,Nigeria,Eau,9.082,8.6753,https://www.wateraid.org
Red Cross,Kenya,Urgence,-0.0236,37.9062,https://www.ifrc.org
UNICEF,Afrique du Sud,Enfance,-30.5595,22.9375,https://www.unicef.org
World Food Programme,Soudan du Sud,Nutrition,6.877,31.307,https://www.wfp.org
Mercy Corps,Ouganda,Développement,1.3733,32.2903,https://www.mercycorps.org
Islamic Relief,Ethiopie,Urgence,9.145,40.4897,https://www.islamic-relief.org
Doctors of the World,Guinée,Santé,9.9456,-9.6966,https://www.medecinsdumonde.org
AVSI Foundation,Kenya,Éducation,-0.0236,37.9062,https://www.avsi.org
Human Appeal,Soudan,Santé,12.8628,30.2176,https://humanappeal.org.uk
The Hunger Project,Bénin,Développement,9.3077,2.3158,https://thp.org
World Relief,Tanzanie,Développement,-6.3690,34.8888,https://worldrelief.org
African Medical and Research Foundation,Kenya,Santé,-1.2921,36.8219,https://amref.org
Education for All,Madagascar,Éducation,-18.7669,46.8691,https://www.educationforall.org
Heifer International,Éthiopie,Agriculture,9.145,40.4897,https://www.heifer.org
HABITAT for Humanity,Ghana,Logement,7.9465,-1.0232,https://www.habitat.org
Refugees International,Libye,Urgence,26.3351,17.2283,https://www.refugeesinternational.org
Global Health Corps,Malawi,Santé,-13.2543,34.3015,https://ghcorps.org
FairTrade Africa,Kenya,Développement,-0.0236,37.9062,https://www.fairtradeafrica.net
Solidarités International,Centrafrique,Urgence,6.6111,20.9394,https://www.solidarites.org
Friends of the Earth,Gabon,Environnement,-0.8037,11.6094,https://www.foei.org
Médecins Sans Frontières,Ukraine,Santé,48.3794,31.1656,https://www.msf.org
Oxfam,Kenya,Développement,-0.0236,37.9062,https://www.oxfam.org
CARE International,Niger,Urgence,17.6078,8.0817,https://www.care.org
Save the Children,Éthiopie,Éducation,9.145,40.4897,https://www.savethechildren.net
Action Contre la Faim,Sénégal,Nutrition,14.4974,-14.4524,https://www.actioncontrelafaim.org
Plan International,Cameroun,Éducation,7.3697,12.3547,https://plan-international.org
World Vision,Rwanda,Développement,-1.9403,29.8739,https://www.wvi.org
Handicap International,Maroc,Santé,31.7917,-7.0926,https://www.handicap-international.org
UNICEF,Inde,Enfance,20.5937,78.9629,https://www.unicef.org
World Food Programme,Haïti,Nutrition,18.9712,-72.2852,https://www.wfp.org
Mercy Corps,Ouganda,Développement,1.3733,32.2903,https://www.mercycorps.org
Islamic Relief,Soudan,Urgence,12.8628,30.2176,https://www.islamic-relief.org
Doctors of the World,Guinée,Santé,9.9456,-9.6966,https://www.medecinsdumonde.org
AVSI Foundation,Kenya,Éducation,-0.0236,37.9062,https://www.avsi.org
The Hunger Project,Bénin,Développement,9.3077,2.3158,https://thp.org
World Relief,Tanzanie,Développement,-6.3690,34.8888,https://worldrelief.org
Heifer International,Éthiopie,Agriculture,9.145,40.4897,https://www.heifer.org
HABITAT for Humanity,Ghana,Logement,7.9465,-1.0232,https://www.habitat.org
Refugees International,Liban,Urgence,33.8547,35.8623,https://www.refugeesinternational.org
Global Health Corps,Malawi,Santé,-13.2543,34.3015,https://ghcorps.org
FairTrade Africa,Kenya,Développement,-0.0236,37.9062,https://www.fairtradeafrica.net
Solidarités International,Centrafrique,Urgence,6.6111,20.9394,https://www.solidarites.org
Friends of the Earth,Brésil,Environnement,-14.235,-51.9253,https://www.foei.org
Greenpeace,France,Environnement,46.2276,2.2137,https://www.greenpeace.org
Red Cross,États-Unis,Urgence,37.0902,-95.7129,https://www.redcross.org
Amnesty International,Espagne,Droits humains,40.4637,-3.7492,https://www.amnesty.org
WWF,Afrique du Sud,Environnement,-30.5595,22.9375,https://www.wwf.org
Doctors Without Borders,Philippines,Santé,12.8797,121.774,https://www.msf.org
World Wildlife Fund,Chine,Environnement,35.8617,104.1954,https://www.worldwildlife.org
Caritas Internationalis,Italie,Développement,41.8719,12.5674,https://www.caritas.org
GlobalGiving,États-Unis,Développement,37.0902,-95.7129,https://www.globalgiving.org
Operation Smile,Égypte,Santé,26.8206,30.8025,https://www.operationsmile.org
Room to Read,Népal,Éducation,28.3949,84.124,https://www.roomtoread.org
WaterAid,Bangladesh,Eau,23.685,90.3563,https://www.wateraid.org
Human Rights Watch,Canada,Droits humains,56.1304,-106.3468,https://www.hrw.org
Save the Rhino,Tanzanie,Environnement,-6.3690,34.8888,https://www.savetherhino.org
Rotary International,Suisse,Développement,46.8182,8.2275,https://www.rotary.org
Food for the Poor,Haïti,Nutrition,18.9712,-72.2852,https://www.foodforthepoor.org
Crisis Action,France,Plaidoyer,46.2276,2.2137,https://crisisaction.org
Transparency International,Allemagne,Droits humains,51.1657,10.4515,https://www.transparency.org
Human Appeal,Palestine,Santé,31.9522,35.2332,https://humanappeal.org.uk
World Literacy Foundation,Australie,Éducation,-25.2744,133.7751,https://worldliteracyfoundation.org
One Acre Fund,Rwanda,Agriculture,-1.9403,29.8739,https://oneacrefund.org
BRAC,Bangladesh,Développement,23.685,90.3563,https://www.brac.net
Concern Worldwide,Irlande,Urgence,53.4129,-8.2439,https://www.concern.net
Wildlife Conservation Society,États-Unis,Environnement,37.0902,-95.7129,https://www.wcs.org
nom,pays,domaine,latitude,longitude,site_web
Relief International,Liban,Urgence,33.8547,35.8623,https://www.ri.org
CARE Japan,Japon,Urgence,36.2048,138.2529,https://www.care.org
Doctors Without Borders,Colombie,Santé,4.5709,-74.2973,https://www.msf.org
War Child,Afghanistan,Enfance,33.9391,67.7100,https://www.warchild.org
Good Neighbors,Corée du Sud,Développement,35.9078,127.7669,https://www.goodneighbors.org
Save the Children,Indonésie,Éducation,-0.7893,113.9213,https://www.savethechildren.net
Norwegian Refugee Council,Norvège,Urgence,60.4720,8.4689,https://www.nrc.no
Finn Church Aid,Finlande,Éducation,61.9241,25.7482,https://www.kirkonulkomaanapu.fi
People in Need,République tchèque,Développement,49.8175,15.4730,https://www.peopleinneed.net
ReliefWeb,Suisse,Plaidoyer,46.8182,8.2275,https://reliefweb.int
Aide et Action,France,Éducation,46.2276,2.2137,https://www.aide-et-action.org
SolidarityNow,Grèce,Droits humains,39.0742,21.8243,https://www.solidaritynow.org
Terre des Hommes,Suisse,Enfance,46.8182,8.2275,https://www.tdh.ch
ACTED,France,Urgence,46.2276,2.2137,https://www.acted.org
Christian Aid,Royaume-Uni,Développement,55.3781,-3.4360,https://www.christianaid.org.uk
CAFOD,Royaume-Uni,Développement,55.3781,-3.4360,https://www.cafod.org.uk
Zakat Foundation of America,États-Unis,Humanitaire,37.0902,-95.7129,https://www.zakat.org
Relief Society of Tigray,Éthiopie,Urgence,9.145,40.4897,https://www.rest-tigray.org
Human Rights Without Frontiers,Belgique,Droits humains,50.5039,4.4699,https://hrwf.eu
Solidarité Laïque,France,Éducation,46.2276,2.2137,https://www.solidarite-laique.org
Médecins du Monde,Espagne,Santé,40.4637,-3.7492,https://www.medicosdelmundo.org
CARE Canada,Canada,Urgence,56.1304,-106.3468,https://care.ca
World Hope International,Sierra Leone,Développement,8.4606,-11.7799,https://www.worldhope.org
GAVI Alliance,Suisse,Santé,46.8182,8.2275,https://www.gavi.org
FHI 360,États-Unis,Développement,37.0902,-95.7129,https://www.fhi360.org
International Rescue Committee,Irak,Urgence,33.2232,43.6793,https://www.rescue.org
Hope for Haiti,Haïti,Développement,18.9712,-72.2852,https://hopeforhaiti.com
Relief International,Soudan,Urgence,12.8628,30.2176,https://www.ri.org
Food for the Hungry,Kenya,Nutrition,-0.0236,37.9062,https://www.fh.org
World Animal Protection,Pays-Bas,Environnement,52.1326,5.2913,https://www.worldanimalprotection.org
Concern Universal,Mozambique,Développement,-18.6657,35.5296,https://www.concernuniversal.org
Global Water Partnership,Suède,Eau,60.1282,18.6435,https://www.gwp.org
CARE India,Inde,Urgence,20.5937,78.9629,https://www.careindia.org
Relief International,Yémen,Urgence,15.5527,48.5164,https://www.ri.org
ChildFund International,Vietnam,Éducation,14.0583,108.2772,https://www.childfund.org
BuildOn,États-Unis,Éducation,37.0902,-95.7129,https://www.buildon.org
Habitat for Humanity,Philippines,Logement,12.8797,121.774,https://www.habitat.org
SOS Villages d’Enfants,Brésil,Enfance,-14.235,-51.9253,https://www.sos-childrensvillages.org
Médecins Sans Frontières,Grèce,Santé,39.0742,21.8243,https://www.msf.org
Oxfam,Guatemala,Développement,15.7835,-90.2308,https://www.oxfam.org
Handicap International,Indonésie,Santé,-0.7893,113.9213,https://www.handicap-international.org
Save the Children,Turquie,Éducation,38.9637,35.2433,https://www.savethechildren.net
Islamic Relief,Somalie,Urgence,5.1521,46.1996,https://www.islamic-relief.org
World Vision,Bolivie,Développement,-16.2902,-63.5887,https://www.wvi.org
Human Rights Watch,Suède,Droits humains,60.1282,18.6435,https://www.hrw.org
CARE International,Thaïlande,Urgence,15.8700,100.9925,https://www.care.org
ActionAid,Nigeria,Développement,9.082,8.6753,https://actionaid.org
Mercy Malaysia,Malaisie,Santé,4.2105,101.9758,https://www.mercy.org.my
Plan International,Paraguay,Éducation,-23.4425,-58.4438,https://plan-international.org
World Food Programme,Pérou,Nutrition,-9.19,-75.0152,https://www.wfp.org
The Asia Foundation,Asie,Développement,0,0,https://asiafoundation.org :contentReference[oaicite:0]{index=0}
China Foundation for Poverty Alleviation,Chine,Développement,39.9042,116.4074,https://www.cfpa.org.cn/ :contentReference[oaicite:1]{index=1}
Hagar International,Vietnam,Santé & droits humains,21.0285,105.8542,https://hagarinternational.org :contentReference[oaicite:2]{index=2}
Asian Forum for Human Rights and Development (FORUM-ASIA),Thaïlande,Droits humains,13.7563,100.5018,https://www.forum-asia.org/ :contentReference[oaicite:3]{index=3}
Malteser International,Inde,Santé & secours,20.5937,78.9629,https://www.malteser-international.org/en/our-work/asia.html :contentReference[oaicite:4]{index=4}
CARE International,Thaïlande,Urgence & développement,15.8700,100.9925,https://www.care.org/our-work/where-we-work/thailand/ :contentReference[oaicite:5]{index=5}
ECPAT International,Thaïlande,Protection enfants,15.8700,100.9925,https://www.ecpat.org/ :contentReference[oaicite:6]{index=6}
HelpAge Asia Pacific,Thaïlande,Droits des personnes âgées,15.8700,100.9925,https://www.helpage.org/asia-pacific/ :contentReference[oaicite:7]{index=7}
Slum Soccer,Inde,Insertion & jeunesse,20.5937,78.9629,https://www.slumsoccer.org/ :contentReference[oaicite:8]{index=8}
Global Sikhs,Inde,Secours & aide d’urgence,20.5937,78.9629,https://theglobalsikhs.org/ :contentReference[oaicite:9]{index=9}
War Child,Afghanistan,Enfance & éducation,33.9391,67.7100,https://www.warchild.org/ :contentReference[oaicite:10]{index=10}
Educate Girls,Inde,Éducation des filles,20.5937,78.9629,https://www.educategirls.ngo/ :contentReference[oaicite:11]{index=11}
BRAC,Bangladesh,Développement & réduction de la pauvreté,23.6850,90.3563,https://www.brac.net/ :contentReference[oaicite:12]{index=12}
Animals Asia,Chine/Vietnam,Protection animale,35.8617,104.1954,https://www.animalsasia.org/ :contentReference[oaicite:13]{index=13}
Dubai Cares,Émirats arabes unis,Éducation,25.2048,55.2708,https://www.dubaicares.ae/ :contentReference[oaicite:14]{index=14}
People in Need,République tchèque (Asie du Sud/Asie),49.8175,15.4730,https://www.peopleinneed.net/ :contentReference[oaicite:15]{index=15}
Good Neighbors,Corée du Sud,Développement,35.9078,127.7669,https://www.goodneighbors.org/ :contentReference[oaicite:16]{index=16}
Child in Need Institute (CINI),Inde,Éducation & santé,20.5937,78.9629,https://www.cini-india.org/ :contentReference[oaicite:17]{index=17}
GAVI Alliance,Suisse (Asie programme),46.8182,8.2275,https://www.gavi.org/ :contentReference[oaicite:18]{index=18}
World Vision,Inde,Développement & secours,20.5937,78.9629,https://www.wvi.org/ :contentReference[oaicite:19]{index=19}
The Asia Foundation (bis),Asie,Renforcement institutionnel,0,0,https://asiafoundation.org/ :contentReference[oaicite:20]{index=20}

    
