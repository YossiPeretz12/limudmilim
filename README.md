from flask import Flask, render_template, request, jsonify
import json
import time
from datetime import datetime, timedelta

app = Flask(__name__)

# מאגר מילים לדוגמה
word_bank = [
    {"word": "superstition", "meaning": "אמונה תפלה", "association": "סופר שטויות"},
    {"word": "intrigued", "meaning": "מסוקרן", "association": "מישהו נכנס לחדר עם טריקים"},
    {"word": "keen", "meaning": "נלהב, חד", "association": "קין נשמע כמו 'כֵּן' – כן בהתלהבות"},
]

PROGRESS_FILE = "learned_words.json"

# קובץ דרישות ל-Replit
requirements_content = """
flask
"""

with open("requirements.txt", "w", encoding="utf-8") as file:
    file.write(requirements_content)

def load_progress():
    try:
        with open(PROGRESS_FILE, "r", encoding="utf-8") as file:
            return json.load(file)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}

def save_progress(progress):
    with open(PROGRESS_FILE, "w", encoding="utf-8") as file:
        json.dump(progress, file, ensure_ascii=False, indent=4)

@app.route('/')
def home():
    return render_template("index.html")

@app.route('/get_word', methods=['GET'])
def get_word():
    progress = load_progress()
    now = datetime.now()
    
    for entry in word_bank:
        word = entry["word"]
        if word not in progress or (not progress[word]["known"] and (now - datetime.fromtimestamp(progress[word]["last_seen"])).seconds > 3600):
            return jsonify(entry)
    return jsonify({"word": "", "meaning": "", "association": ""})

@app.route('/submit_answer', methods=['POST'])
def submit_answer():
    data = request.json
    word = data["word"]
    known = data["known"]
    progress = load_progress()
    progress[word] = {"known": known, "last_seen": time.time()}
    save_progress(progress)
    return jsonify({"status": "success"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=81, debug=True)

# יצירת קובץ HTML לממשק המשתמש
html_content = """
<!DOCTYPE html>
<html lang='he'>
<head>
    <meta charset='UTF-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1.0'>
    <title>לימוד מילים באנגלית</title>
    <script>
        async function fetchWord() {
            let response = await fetch('/get_word');
            let data = await response.json();
            document.getElementById('word').innerText = data.word || "כל המילים נלמדו!";
            document.getElementById('meaning').innerText = data.meaning;
            document.getElementById('association').innerText = data.association;
        }
        
        async function submitAnswer(known) {
            let word = document.getElementById('word').innerText;
            if (word) {
                await fetch('/submit_answer', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ word: word, known: known })
                });
                fetchWord();
            }
        }
        window.onload = fetchWord;
    </script>
</head>
<body>
    <h1>לימוד מילים באנגלית</h1>
    <h2 id='word'></h2>
    <p id='meaning'></p>
    <p id='association'></p>
    <button onclick='submitAnswer(true)'>מכיר</button>
    <button onclick='submitAnswer(false)'>לא מכיר</button>
</body>
</html>
"""

with open("templates/index.html", "w", encoding="utf-8") as file:
    file.write(html_content)
