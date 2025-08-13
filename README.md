import threading
import traceback
import json
import os
import datetime
import fitz
import speech_recognition as sr
from gtts import gTTS
from playsound import playsound
import requests
import customtkinter as ctk

# ===== CONFIG =====
SERPER_API_KEY = "YOUR_SERPER_API_KEY"
SyllabusFile = "syllabus.json"
ReminderFile = "reminders.json"

voice_enabled = True
selected_language = "en"
selected_lang_name = "English"
chat_history = []

# ===== Helper Functions =====
def speak(text):
    if voice_enabled:
        tts = gTTS(text=text, lang=selected_language)
        filename = "temp.mp3"
        tts.save(filename)
        playsound(filename)
        os.remove(filename)

def capture_voice():
    recognizer = sr.Recognizer()
    with sr.Microphone() as source:
        audio = recognizer.listen(source)
        try:
            return recognizer.recognize_google(audio, language=selected_language)
        except:
            return None

def load_syllabus_db():
    if os.path.exists(SyllabusFile):
        with open(SyllabusFile, "r") as f:
            return json.load(f)
    return {}

def get_serper_response(query, memory_context=None):
    combined_query = f"{memory_context}\n{query}" if memory_context else query
    url = "https://google.serper.dev/search"
    headers = {"X-API-KEY": SERPER_API_KEY, "Content-Type": "application/json"}
    payload = {"q": combined_query, "num": 5}
    try:
        response = requests.post(url, headers=headers, json=payload)
        data = response.json()
        snippets = [item["snippet"] for item in data.get("organic", [])]
        return "\n".join(snippets) if snippets else "Sorry, I could not find an answer online."
    except Exception as e:
        return f"Error fetching answer: {e}"

def get_response(user_text):
    syllabus_db = load_syllabus_db()
    context = "\n".join(chat_history[-10:])
    # offline syllabus
    for subject, topics in syllabus_db.items():
        for topic, content in topics.items():
            if topic.lower() in user_text.lower():
                return content
    # fallback to Serper
    return get_serper_response(user_text, memory_context=context)

# ===== UI =====
ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")
app = ctk.CTk()
app.geometry("900x600")
app.title("Sarvix EduBot")

# Sidebar
sidebar = ctk.CTkFrame(app, width=160, corner_radius=0)
sidebar.pack(side="left", fill="y")
ctk.CTkLabel(sidebar, text="SARVIX AI", font=("Arial", 20, "bold")).pack(pady=18)

def clear_chat():
    global chat_history
    chat_history = []
    for widget in chat_frame.winfo_children():
        widget.destroy()
ctk.CTkButton(sidebar, text="🆕 New Chat", command=clear_chat).pack(pady=8)

def toggle_voice():
    global voice_enabled
    voice_enabled = not voice_enabled
    voice_button.configure(text=f"🔊 Voice {'ON' if voice_enabled else 'OFF'}")
voice_button = ctk.CTkButton(sidebar, text="🔊 Voice ON", command=toggle_voice)
voice_button.pack(pady=8)

# Language Selector
def set_language(choice):
    global selected_language, selected_lang_name
    lang_map = {"English":"en", "Hindi":"hi", "Kannada":"kn", "Tamil":"ta", "Telugu":"te"}
    selected_language = lang_map.get(choice, "en")
    selected_lang_name = choice
    lang_label.configure(text=f"Lang: {choice}")
ctk.CTkOptionMenu(sidebar, values=["English","Hindi","Kannada","Tamil","Telugu"], command=set_language).pack(pady=(6,0))
lang_label = ctk.CTkLabel(sidebar, text="Lang: English")
lang_label.pack(pady=(4,12))

# Chat Frame
chat_frame = ctk.CTkScrollableFrame(app, width=720, corner_radius=10)
chat_frame.pack(side="top", fill="both", expand=True, padx=10, pady=10)

def add_message(text, sender="ai"):
    global chat_history
    chat_history.append(f"{sender}: {text}")
    label = ctk.CTkLabel(chat_frame, text=text, anchor="w", justify="left", wraplength=600,
                         fg_color="#2563EB" if sender=="ai" else "#1E40AF",
                         text_color="white", corner_radius=8, padx=10, pady=6)
    label.pack(anchor="w" if sender=="ai" else "e", pady=4, padx=6)

# Input bar
input_bar = ctk.CTkFrame(app, height=60, corner_radius=0)
input_bar.pack(side="bottom", fill="x", padx=8, pady=8)
entry = ctk.CTkEntry(input_bar, placeholder_text="Type your message...", width=560)
entry.pack(side="left", padx=10, pady=10)

def process_input(user_text):
    if not user_text or not user_text.strip():
        return
    add_message(user_text, "user")
    entry.delete(0, "end")
    def worker():
        try:
            response = get_response(user_text)
            add_message(response, "ai")
            speak(response)
        except Exception as e:
            add_message(f"Error: {e}", "ai")
            print(traceback.format_exc())
    threading.Thread(target=worker, daemon=True).start()

def on_send():
    process_input(entry.get())

def on_voice():
    def worker():
        add_message("🎤 Listening...", "ai")
        user_text = capture_voice()
        if user_text:
            add_message(user_text, "user")
            process_input(user_text)
        else:
            add_message("❌ Could not recognize speech", "ai")
    threading.Thread(target=worker, daemon=True).start()

send_btn = ctk.CTkButton(input_bar, text="Send", width=80, command=on_send)
send_btn.pack(side="left", padx=(0,6))
mic_btn = ctk.CTkButton(input_bar, text="🎤", width=60, command=on_voice)
mic_btn.pack(side="left", padx=(0,6))
entry.bind("<Return>", lambda e: on_send())

if __name__ == "__main__":
    add_message("Hello! I'm Sarvix EduBot. Type or speak to start.", "ai")
    app.mainloop()
