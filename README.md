# EducationGPT - A Versatile Educational Chatbot for Real-World Problem Solving

from flask import Flask, request, jsonify, render_template 
from googletrans import Translator
import speech_recognition as sr 
import pyttsx3 
import requests 
from PIL import Image 
import openai 
import os
import re

app = Flask(__name__)

# Initialize Translator and Text-to-Speech Engine

translator = Translator() 
tts_engine = pyttsx3.init() 
tts_engine.setProperty('rate', 150)

# Memory to store past conversations

memory = []

# Supported Languages

languages = ['en', 'ru', 'zh-cn', 'pa', 'hi', 'te', 'ta', 'fr', 'ja', 'ko', 'sw', 'fi', 'no', 'it', 'de', 'es', 'ar']

# Voice to Text Function

def voice_to_text(audio): 
    recognizer = sr.Recognizer() 
    try: 
        with sr.AudioFile(audio) as source: 
          audio_data = recognizer.record(source) 
        text = recognizer.recognize_google(audio_data) 
        return text 
    except Exception as e: 
            return f"Speech Recognition error: {str(e)}"

# Text to Speech Function

def text_to_speech(text): 
  try:
     tts_engine.say(text) 
     tts_engine.runAndWait()
  except Exception as e:
        print(f"TTS error: {str(e)}")

# Web Search Function

def web_search(query): 
    try:       
       response = requests.get(f'https://api.duckduckgo.com/?q={query}&format=json') 
       return response.json().get('AbstractText', 'No results found.')
    except Exception as e:
       return f"Web search error: {str(e)}"
    
# Image Generation Function

def create_image(text): 
    image = Image.new('RGB', (300, 100), color='white') 
    image.save(f'{text}.png') 
    return f'{text}.png'

# Save Memory

def save_memory(user_input, response): memory.append((user_input, response))

# Generate Response Function

def generate_response(input_text): 
    try: 
        response = openai.ChatCompletion.create( 
        model="gpt-3.5-turbo", messages=[{"role": "user", "content": input_text}] 
        ) 
        return response['choices'][0]['message']['content'] 
    except Exception as e: 
        return f"Error: {str(e)}"

# Home route for the webpage
@app.route('/') 
def home(): 
    return render_template('index.html')

# Chat route to handle user chat via POST request

@app.route('/chat', methods=['POST']) 

def chat(): 
    data = request.json 
    print(data)
    user_input = data.get('message') 
    lang = data.get('lang', 'en') 
    if lang not in languages: 
     lang = 'en' 
     # Translate user input to English for processing
    try:
      translated_input = translator.translate(user_input, dest='en').text 
    except Exception:
        translated_input = user_input
    if 'search' in user_input: 
        response_text = web_search(translated_input) 
    elif 'create image' in user_input: 
        image_path = create_image(translated_input) 
        response_text = f'Image created: {image_path}' 
    else: 
        response_text = generate_response(translated_input) 
        try:
         translated_response = translator.translate(response_text, dest=lang).text 
        except Exception:
           translated_response = response_text 
           save_memory(user_input, translated_response) 
        return jsonify({"response": translated_response})

# Voice Chat route to handle voice input

@app.route('/voice_chat', methods=['POST']) 
def voice_chat():
    if 'audio' not in request.files:
        return jsonify({"response": "No audio file provided."}), 400
    audio = request.files['audio'] 
    user_input = voice_to_text(audio) 
    response_text = generate_response(user_input) 
    text_to_speech(response_text) 
    save_memory(user_input, response_text) 
    return jsonify({"response": response_text})

# Upload route to receive files (like images, videos, and PDFs)

@app.route('/upload', methods=['POST']) 
def upload(): 
    if 'files' not in request.files:
        return jsonify({"response": "No file provided."}), 400
    file = request.files['file'] 
    if file.filename.endswith('.pdf'): 
         file.save(f'uploaded_{file.filename}') 
         return jsonify({"status": "PDF file received."}) 
    elif file.filename.endswith(('.png', '.jpg', '.jpeg', '.mp4', '.mkv')):
        file.save(f'uploaded_{file.filename}')
        return jsonify({"status": "File received"})
    else:
        return jsonify({"status": "Unsupported file type."})

# Memory route to retrieve past conversations

@app.route('/memory', methods=['GET']) 
def get_memory(): 
    return jsonify({"memory": memory})

# Run the Flask app

if __name__ == 'main': 
    app.run(debug=True, host='0.0.0.0', port=5000)  

    <!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EducationGPT - Chatbot</title>
</head>
<body>
    <h1>Welcome to EducationGPT Chatbot</h1>

    <!-- Chat Form -->
    <form id="chat-form">
        <label for="message">Enter your message:</label><br>
        <textarea id="message" name="message" placeholder="Ask me anything!" rows="4" cols="50"></textarea><br><br>
        
        <label for="language">Choose a language:</label>
        <select id="language" name="language">
            <option value="en">English</option>
            <option value="ru">Russian</option>
            <option value="zh-cn">Chinese</option>
            <option value="pa">Punjabi</option>
            <option value="hi">Hindi</option>
            <option value="te">Telugu</option>
            <option value="ta">Tamil</option>
            <option value="fr">French</option>
            <option value="ja">Japanese</option>
            <option value="ko">Korean</option>
            <option value="sw">Swahili</option>
            <option value="fi">Finnish</option>
            <option value="no">Norwegian</option>
            <option value="it">Italian</option>
            <option value="de">German</option>
            <option value="es">Spanish</option>
            <option value="ar">Arabic</option>
        </select><br><br>

        <button type="submit">Send</button>
    </form>

    <h2>Chatbot Response:</h2>
    <div id="response"></div>

    <!-- File Upload Form -->
    <form id="upload-form" enctype="multipart/form-data">
        <label for="file">Upload a file (PDF, image, video):</label><br>
        <input type="file" id="file" name="file"><br><br>
        <button type="submit">Upload</button>
    </form>

    <h2>Upload Status:</h2>
    <div id="upload-status"></div>

    <script>
        // Chat form handling
        const chatForm = document.getElementById('chat-form');
        chatForm.addEventListener('submit', function(event) {
            event.preventDefault();
            const message = document.getElementById('message').value;
            const language = document.getElementById('language').value;

            fetch('/chat', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ message: message, lang: language })
            })
            .then(response => response.json())
            .then(data => {
                document.getElementById('response').textContent = "Response: " + data.response;
            })
            .catch(error => console.error('Error:', error));
        });

        // File upload form handling
        const uploadForm = document.getElementById('upload-form');
        uploadForm.addEventListener('submit', function(event) {
            event.preventDefault();
            const formData = new FormData();
            const fileInput = document.getElementById('file');
            formData.append('file', fileInput.files[0]);

            fetch('/upload', {
                method: 'POST',
                body: formData
            })
            .then(response => response.json())
            .then(data => {
                document.getElementById('upload-status').textContent = "Upload Status: " + data.status;
            })
            .catch(error => console.error('Error:', error));
        });
    </script>
</body>
</html>
