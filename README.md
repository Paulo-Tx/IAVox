# IAVox
ferramenta baseada em inteligência artificial que gera relatórios de aprendizagem a partir da voz do usuário.
import streamlit as st
import speech_recognition as sr
import requests
import json
import time

API_KEY = "AIzaSyBbg6lqbEC22SzffzT8x1I4Pt8i5fc8UXQ"
GEMINI_URL = f"https://generativelanguage.googleapis.com/v1/models/gemini-1.5-pro:generateContent?key={API_KEY}"

# Inicializar sessão
if 'modelo_relatorio' not in st.session_state:
    st.session_state.modelo_relatorio = ""
if 'relatorio_gerado' not in st.session_state:
    st.session_state.relatorio_gerado = ""

# Etapa 1 - Definir modelo de relatório
st.title("Relatório Técnico de Feedback por Voz")

if st.session_state.modelo_relatorio == "":
    st.subheader("📌 Primeira Etapa: Personalize seu modelo de relatório")
    modelo = st.text_area("Descreva como deve ser o relatório (ex: linguagem, estrutura, título, assinatura...)")
    if st.button("Salvar modelo de relatório"):
        if modelo.strip() != "":
            st.session_state.modelo_relatorio = modelo
            st.success("Modelo salvo com sucesso!")
        else:
            st.warning("Digite algo antes de salvar.")
    st.stop()

# Função para gravar áudio
def gravar_audio(timeout=10):
    reconhecedor = sr.Recognizer()
    with sr.Microphone() as source:
        st.info("🎤 Gravando... Você pode falar e, se quiser finalizar antes dos 10 segundos, diga 'finalizar relatório'.")
        reconhecedor.adjust_for_ambient_noise(source)
        audio = reconhecedor.listen(source, timeout=timeout, phrase_time_limit=timeout)
        st.success("Gravação encerrada.")
    try:
        texto = reconhecedor.recognize_google(audio, language="pt-BR")
        return texto
    except sr.UnknownValueError:
        return None
    except sr.RequestError as e:
        return f"Erro na API de reconhecimento: {e}"

# Função para gerar relatório com a Gemini API
def gerar_relatorio(texto_usuario, modelo_usuario):
    prompt = f"""
Você é um assistente de escrita técnica. A partir do texto abaixo, siga o modelo fornecido para gerar um relatório técnico de feedback.

MODELO PERSONALIZADO:
{modelo_usuario}

TEXTO BASE DO USUÁRIO:
"{texto_usuario}"

Finalize com:
Atenciosamente,

Paulo Teixeira  
Assessor Pedagógico da Aprendizagem Profissional  
Fundação Fé e Alegria do Brasil | Filial Pernambuco
"""

    payload = {
        "contents": [
            {
                "parts": [{"text": prompt}]
            }
        ]
    }

    headers = {"Content-Type": "application/json"}
    resposta = requests.post(GEMINI_URL, headers=headers, data=json.dumps(payload))

    if resposta.status_code == 200:
        dados = resposta.json()
        try:
            return dados["candidates"][0]["content"]["parts"][0]["text"]
        except:
            return "Erro ao interpretar a resposta da IA."
    else:
        return f"Erro na API Gemini: {resposta.status_code} - {resposta.text}"

# Etapa 2 - Captar áudio e gerar relatório
st.subheader("🎙️ Grave seu feedback oral")

if st.button("Iniciar gravação"):
    texto_audio = gravar_audio()

    if texto_audio:
        st.write(f"📝 Texto reconhecido: {texto_audio}")
        if "finalizar relatório" in texto_audio.lower():
            texto_audio = texto_audio.lower().replace("finalizar relatório", "").strip()
        with st.spinner("Gerando relatório com IA..."):
            relatorio = gerar_relatorio(texto_audio, st.session_state.modelo_relatorio)
            st.session_state.relatorio_gerado = relatorio
    else:
        st.warning("Não foi possível reconhecer sua fala.")

# Etapa 3 - Mostrar relatório com opções
if st.session_state.relatorio_gerado:
    st.subheader("📄 Relatório gerado")
    texto_editado = st.text_area("Você pode revisar e editar:", st.session_state.relatorio_gerado, height=400)

    col1, col2, col3 = st.columns(3)
    with col1:
        if st.button("💾 Salvar relatório"):
            st.success("Relatório salvo na sessão. (Futuramente poderá ser enviado à nuvem ou salvo como PDF)")
    with col2:
        if st.button("📝 Editar novamente"):
            st.session_state.relatorio_gerado = texto_editado
            st.info("Você pode continuar editando acima.")
    with col3:
        if st.button("🗑️ Excluir tudo"):
            st.session_state.relatorio_gerado = ""
            st.success("Relatório apagado.")
