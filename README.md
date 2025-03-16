import requests
import re
import time
import speech_recognition as sr
import pyttsx3
import json
import os
import openai
import logging
import ast
import operator
from dotenv import load_dotenv

# Carregar variáveis de ambiente
load_dotenv()

# Configuração do mecanismo de fala
engine = pyttsx3.init()
voices = engine.getProperty('voices')

# Escolhendo uma voz feminina alternativa
for voice in voices:
    if "feminina" in voice.name.lower():
        engine.setProperty('voice', voice.id)
        break
engine.setProperty('rate', 170)

# Configuração de logs
logging.basicConfig(filename="debug.log", level=logging.DEBUG, 
                    format="%(asctime)s - %(levelname)s - %(message)s")

# Configuração da OpenAI API
openai.api_key = os.getenv("OPENAI_API_KEY", "SUA_CHAVE_AQUI")

def load_json(file_name):
    try:
        with open(file_name, "r", encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        return {}

def save_json(file_name, data):
    with open(file_name, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=4)

knowledge_base = load_json("knowledge.json")
interaction_history = load_json("history.json")
logs = load_json("logs.json")

def speak(text):
    try:
        engine.say(text)
        engine.runAndWait()
    except Exception as e:
        logging.error(f"Erro no mecanismo de fala: {e}")

def listen():
    recognizer = sr.Recognizer()
    with sr.Microphone() as source:
        print("🎤 Ouvindo... Fale algo!")
        recognizer.adjust_for_ambient_noise(source, duration=1)
        try:
            audio = recognizer.listen(source, timeout=10, phrase_time_limit=5)
            text = recognizer.recognize_google(audio, language="pt-BR")
            print(f"Você: {text}")
            return text.lower()
        except (sr.UnknownValueError, sr.WaitTimeoutError, sr.RequestError):
            return None

def search_wikipedia(query):
    url = f"https://pt.wikipedia.org/api/rest_v1/page/summary/{query.replace(' ', '_')}"
    try:
        response = requests.get(url)
        data = response.json()
        return data.get("extract", "Não encontrei informações sobre isso.")
    except Exception as e:
        logging.error(f"Erro ao buscar na Wikipedia: {e}")
        return "Não consegui encontrar nada."

def generate_response(prompt):
    try:
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "system", "content": "Você é um assistente útil."},
                      {"role": "user", "content": prompt}]
        )
        return response["choices"][0]["message"]["content"].strip()
    except Exception as e:
        logging.error(f"Erro ao conectar com OpenAI API: {e}")
        return "Não consegui gerar uma resposta."

def calculate_math(expression):
    try:
        tree = ast.parse(expression, mode='eval')
        for node in ast.walk(tree):
            if not isinstance(node, (ast.Expression, ast.BinOp, ast.Num, ast.UnaryOp, ast.Load)):
                return "Expressão inválida."
        return str(eval(expression, {"__builtins__": None}, {"+": operator.add, "-": operator.sub, "*": operator.mul, "/": operator.truediv}))
    except Exception:
        return "Não consegui calcular isso."

def learn_from_user(query):
    if re.match(r'^[0-9+\-*/(). ]+$', query):
        return calculate_math(query)
    
    if query in knowledge_base:
        return knowledge_base[query]
    
    ai_response = generate_response(query)
    print("🤖 Minha resposta: ", ai_response)
    speak("Minha resposta: " + ai_response)
    
    print("🤖 Essa resposta foi útil? Responda 'sim' ou 'não'")
    speak("Essa resposta foi útil? Responda 'sim' ou 'não'")
    feedback = listen()
    
    if feedback == "não":
        print("🤖 O que seria uma resposta melhor?")
        speak("O que seria uma resposta melhor?")
        better_response = listen()
        if better_response:
            knowledge_base[query] = better_response
            interaction_history[query] = interaction_history.get(query, 0) + 1
            logs[query] = {"correção": better_response, "tentativas": logs.get(query, {}).get("tentativas", 0) + 1}
            save_json("knowledge.json", knowledge_base)
            save_json("history.json", interaction_history)
            save_json("logs.json", logs)
            print("✅ Aprendi essa nova resposta!")
            speak("Aprendi essa nova resposta!")
            return better_response
    
    knowledge_base[query] = ai_response
    interaction_history[query] = interaction_history.get(query, 0) + 1
    logs[query] = {"resposta": ai_response, "tentativas": logs.get(query, {}).get("tentativas", 0) + 1}
    save_json("knowledge.json", knowledge_base)
    save_json("history.json", interaction_history)
    save_json("logs.json", logs)
    return ai_response

def chatbot_loop():
    while True:
        user_input = listen()
        if user_input is None:
            continue

        if user_input == "sair":
            print("👋 Encerrando chatbot...")
            speak("Até logo!")
            break

        response = learn_from_user(user_input)
        print("🤖 Chatbot:", response)
        speak(response)
        time.sleep(1)

print("🤖 Chatbot IA iniciado! Pergunte algo ou diga 'sair' para encerrar.")
speak("Olá! Estou pronta para responder suas perguntas. Pergunte algo!")
chatbot_loop()
