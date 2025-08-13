import customtkinter as ctk
import threading
import requests, json, os, tempfile
from gtts import gTTS
import speech_recognition as sr
from groq import Groq
from playsound import playsound
import traceback
import time

# === Conversation Memory ===
chat_history = []

# === Voice Output Toggle & Language ===
voice_enabled = True
selected_language = "en"  # Default English
selected_lang_name = "English"

# === API Keys ===
GROQ_API_KEY = "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"
SERPER_API_KEY = "a6d7c6cd31116d364cd753b07cf2a5588b62581f"

# === Initialize Groq ===
groq_client = Groq(api_key=GROQ_API_KEY)

# === Helper: safe UI add message ===
def add_message(text, sender="ai"):
    try:
        color = "#00e5ff" if sender == "ai" else "#d1ffd6"
        anchor_side = "w" if sender == "ai" else "e"
        bubble = ctk.CTkLabel(chat_frame, text=text, fg_color=color,
                              text_color="black",
                              corner_radius=8, wraplength=600, justify="left", anchor="w")
        bubble.pack(fill="x", padx=10, pady=6, anchor=anchor_side)
        chat_frame.update_idletasks()
        chat_frame._canvas.yview_moveto(1.0)   # Scroll to bottom fix
    except Exception:
        print("Error adding message to UI:", traceback.format_exc())

# === Groq ask with language enforcement ===
def ask_groq(prompt):
    global selected_language, selected_lang_name
    system_instruction = f"You are Sarvix AI. Always respond in {selected_lang_name}."
    chat_history.append({"role": "system", "content": system_instruction})
    chat_history.append({"role": "user", "content": prompt})
    recent_history = chat_history[-10:]
    try:
        resp = groq_client.chat.completions.create(
            model="llama3-70b-8192",
            messages=recent_history,
            temperature=0.7,
            max_tokens=400
        )
        reply = resp.choices[0].message.content.strip()
        chat_history.append({"role": "assistant", "content": reply})
        return reply
    except Exception as e:
        return f"Error (Groq): {e}"

# === Serper Search + translate result ===
def search_google(query):
    global selected_lang_name
    headers = {'X-API-KEY': SERPER_API_KEY, 'Content-Type': 'application/json'}
    data = {'q': query}
    try:
        res = requests.post("https://google.serper.dev/search", headers=headers, data=json.dumps(data))
        j = res.json()
        if 'answerBox' in j and 'answer' in j['answerBox']:
            result = j['answerBox']['answer']
        elif 'organic' in j and len(j['organic']) > 0:
            result = "\n".join([f"- {item.get('snippet','')}" for item in j['organic'][:3]])
        else:
            result = "I couldn't find anything relevant."

        # Translate result to selected language via Groq
        translation_prompt = f"Translate the following text to {selected_lang_name}:\n{result}"
        translated = ask_groq(translation_prompt)
        return translated
    except Exception as e:
        return f"Error (Search): {e}"

# === Decide Groq vs Google ===
def get_response(query):
    q = query.lower().strip()

    # Keywords that require live data search
    realtime_triggers = [
        "latest", "today", "current", "news", "update", "price", "rate",
        "weather", "temperature", "score", "match", "event", "happening",
        "live", "trending", "stock", "gold", "crypto"
    ]

    # Only question starters related to facts/figures, but exclude "who" and "what"
    question_starters_for_search = [
        "when", "where", "how much", "how many", "is", "are"
    ]

    has_numbers = any(char.isdigit() for char in q)

    # If query likely needs live search
    if (any(word in q for word in realtime_triggers) or
        any(q.startswith(s) for s in question_starters_for_search) or
        has_numbers):
        return search_google(query)

    # Else, respond with AI chat model (Groq)
    return ask_groq(query)

# === gTTS Speak ===
def speak(text):
    global selected_language, voice_enabled
    if not voice_enabled or not text:
        return
    try:
        with tempfile.NamedTemporaryFile(delete=False, suffix=".mp3") as fp:
            tmp_path = fp.name
        tts = gTTS(text=text, lang=selected_language)
        tts.save(tmp_path)
        playsound(tmp_path)
        time.sleep(0.1)
        try:
            os.remove(tmp_path)
        except:
            pass
    except Exception as e:
        add_message(f"Error (TTS): {e}", "ai")

# === Voice Capture ===
def capture_voice(display_callback):
    r = sr.Recognizer()
    with sr.Microphone() as source:
        display_callback("🎤 Listening...")
        audio = r.listen(source, phrase_time_limit=8)
    try:
        text = r.recognize_google(audio)
        return text if text.strip() else None
    except Exception as e:
        print("Speech recognition error:", e)
        return None

# === Build UI ===
ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")

app = ctk.CTk()
app.geometry("900x600")
app.title("Sarvix AI")

# Sidebar
sidebar = ctk.CTkFrame(app, width=160, corner_radius=0)
sidebar.pack(side="left", fill="y")

logo_label = ctk.CTkLabel(sidebar, text="SARVIX AI", font=("Arial", 20, "bold"))
logo_label.pack(pady=18)

def clear_chat():
    global chat_history
    chat_history = []
    for widget in chat_frame.winfo_children():
        widget.destroy()

new_chat_btn = ctk.CTkButton(sidebar, text="🆕 New Chat", command=clear_chat)
new_chat_btn.pack(pady=8)

def toggle_voice():
    global voice_enabled
    voice_enabled = not voice_enabled
    voice_button.configure(text=f"🔊 Voice {'ON' if voice_enabled else 'OFF'}")

voice_button = ctk.CTkButton(sidebar, text="🔊 Voice ON", command=toggle_voice)
voice_button.pack(pady=8)

# === Language Selector ===
def set_language(choice):
    global selected_language, selected_lang_name
    lang_map = {
        "English": "en",
        "Hindi": "hi",
        "Kannada": "kn",
        "Tamil": "ta",
        "Telugu": "te"
    }
    selected_language = lang_map.get(choice, "en")
    selected_lang_name = choice
    lang_label.configure(text=f"Lang: {choice}")

lang_menu = ctk.CTkOptionMenu(sidebar, values=["English", "Hindi", "Kannada", "Tamil", "Telugu"], command=set_language)
lang_menu.set("English")
lang_menu.pack(pady=(6,0))

lang_label = ctk.CTkLabel(sidebar, text="Lang: English")
lang_label.pack(pady=(4, 12))

# Chat Area
chat_frame = ctk.CTkScrollableFrame(app, width=720, corner_radius=10)
chat_frame.pack(side="top", fill="both", expand=True, padx=10, pady=10)

# Input Bar
input_bar = ctk.CTkFrame(app, height=60, corner_radius=0)
input_bar.pack(side="bottom", fill="x", padx=8, pady=8)

entry = ctk.CTkEntry(input_bar, placeholder_text="Type your message...", width=560)
entry.pack(side="left", padx=10, pady=10)

def process_input(user_text):
    if not user_text or not user_text.strip():
        return
    add_message(user_text, sender="user")
    entry.delete(0, "end")

    def worker():
        try:
            response = get_response(user_text)
            add_message(response, sender="ai")
            speak(response)
        except Exception as e:
            add_message(f"Error processing input: {e}", "ai")
            print(traceback.format_exc())
    threading.Thread(target=worker, daemon=True).start()

def on_send():
    process_input(entry.get())

def on_voice():
    def worker():
        add_message("🎤 Listening...", sender="ai")
        user_text = capture_voice(lambda s: add_message(s, "ai"))
        if user_text:
            process_input(user_text)
        else:
            add_message("❌ Could not recognize speech", "ai")
    threading.Thread(target=worker, daemon=True).start()

send_btn = ctk.CTkButton(input_bar, text="Send", width=80, command=on_send)
send_btn.pack(side="left", padx=(0,6))

mic_btn = ctk.CTkButton(input_bar, text="🎤", width=60, command=on_voice)
mic_btn.pack(side="left", padx=(0,6))

# Allow Enter key to send
def enter_pressed(event):
    on_send()
entry.bind("<Return>", enter_pressed)

if __name__ == "__main__":
    add_message("Hello! I'm Sarvix AI. Type or speak to start.", "ai")
    app.mainloop()

