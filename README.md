# aplikasi-las-gratis
import streamlit as st
import google.generativeai as genai
import matplotlib.pyplot as plt
import matplotlib.patches as patches

# --- KONFIGURASI HALAMAN ---
st.set_page_config(page_title="Welding Inspector AI", page_icon="🔥", layout="wide")

# --- CSS BIAR KEREN ---
st.markdown("""
    <style>
    .stButton>button { width: 100%; background-color: #FF4B4B; color: white; font-weight: bold; }
    .metric-card { background-color: #f0f2f6; padding: 15px; border-radius: 10px; text-align: center; border-left: 5px solid #FF4B4B; }
    </style>
    """, unsafe_allow_html=True)

# --- FUNGSI GAMBAR KAPUH (SVG STYLE) ---
def draw_groove(groove_type, root_gap=2, angle=60):
    fig, ax = plt.subplots(figsize=(4, 2))
    ax.set_axis_off()
    if groove_type == "Single V-Groove":
        poly_L = [[0,0], [2,0], [2.5,2], [0,2]]
        poly_R = [[4,2], [4.5,0], [6.5,0], [6.5,2]]
        ax.text(3.25, -0.5, "V-Groove", ha='center')
    elif groove_type == "Fillet (T-Joint)":
        poly_L = [[0,0], [6,0], [6,1], [0,1]]
        poly_R = [[2.5,1], [3.5,1], [3.5,4], [2.5,4]]
        ax.text(3, -0.5, "T-Joint", ha='center')
    else: # Square
        poly_L = [[0,0], [2,0], [2,2], [0,2]]
        poly_R = [[2.5,0], [4.5,0], [4.5,2], [2.5,2]]
        ax.text(2.25, -0.5, "Square Butt", ha='center')
        
    ax.add_patch(patches.Polygon(poly_L, closed=True, color='grey'))
    ax.add_patch(patches.Polygon(poly_R, closed=True, color='grey'))
    ax.set_xlim(-1, 7)
    ax.set_ylim(-1, 5)
    return fig

# --- UI UTAMA ---
st.title("🔥 AI Welding Inspector (Gratis)")

# --- AMBIL API KEY DARI RAHASIA SERVER (SECRETS) ---
# Ini kuncinya agar user tidak perlu input key
try:
    api_key = st.secrets["GOOGLE_API_KEY"]
except:
    st.error("⚠️ API Key belum disetting di Server Streamlit!")
    st.stop()

col1, col2 = st.columns(2)

with col1:
    st.subheader("1. Input Data")
    material = st.selectbox("Material", ["ASTM A36 Carbon Steel", "SS 304", "SS 316L", "Aluminium 6061"])
    proses = st.selectbox("Proses", ["SMAW", "GMAW", "GTAW", "FCAW"])
    posisi = st.selectbox("Posisi", ["1G", "2G", "3G", "4G", "1F", "2F"])
    ampere = st.slider("Ampere (A)", 50, 300, 120)
    voltage = st.slider("Voltage (V)", 10, 40, 24)

with col2:
    st.subheader("2. Visual Kapuh")
    kapuh = st.selectbox("Tipe Kapuh", ["Single V-Groove", "Square Butt", "Fillet (T-Joint)"])
    st.pyplot(draw_groove(kapuh), use_container_width=False)
    
    # Hitung Heat Input
    hi = (voltage * ampere * 0.06) / 150 # Asumsi speed 150
    st.markdown(f"<div class='metric-card'><h3>Heat Input: {hi:.2f} kJ/mm</h3></div>", unsafe_allow_html=True)

if st.button("🚀 ANALISIS HASIL LAS"):
    genai.configure(api_key=api_key)
    model = genai.GenerativeModel('gemini-1.5-flash')
    
    prompt = f"""
    Bertindaklah sebagai Welding Inspector. Analisis parameter ini:
    Material: {material}, Proses: {proses}, Posisi: {posisi}, Kapuh: {kapuh}
    Ampere: {ampere} A, Voltage: {voltage} V, Heat Input: {hi} kJ/mm.
    
    Berikan output singkat dalam Bahasa Indonesia:
    1. Apakah parameter ini AMAN atau BERISIKO cacat?
    2. Prediksi cacat yang mungkin terjadi (misal: Undercut/Porosity).
    3. Rekomendasi perbaikan singkat.
    """
    
    with st.spinner("Sedang Menganalisis..."):
        try:
            response = model.generate_content(prompt)
            st.success("Hasil Analisis:")
            st.write(response.text)
        except Exception as e:
            st.error(f"Gagal koneksi: {e}")
