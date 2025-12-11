# ai_notes_app.py
# Full single-file AI-powered Note App
# Features:
# - Create, edit, delete notes
# - Store notes in SQLite
# - AI auto-summary (OpenAI API)
# - REST API + Web UI
# - Bootstrap responsive UI
# - English-only code
# -------------------------------------
# Requirements:
# pip install flask openai python-dotenv

import os
import sqlite3
from flask import (
    Flask, render_template_string, request, redirect,
    url_for, session, flash, jsonify, g
)
from datetime import datetime
from openai import OpenAI

# Load OpenAI key
client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))

app = Flask(__name__)
app.config["SECRET_KEY"] = "dev-secret"
DB_PATH = "notes.db"

# -----------------------------
# Database
# -----------------------------
def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(DB_PATH)
        g.db.row_factory = sqlite3.Row
    return g.db

@app.teardown_appcontext
def close_db(error):
    db = g.pop("db", None)
    if db:
        db.close()

def init_db():
    db = get_db()
    db.execute("""
    CREATE TABLE IF NOT EXISTS notes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        content TEXT NOT NULL,
        summary TEXT,
        created_at TEXT NOT NULL
    )
    """)
    db.commit()

with app.app_context():
    init_db()

# -----------------------------
# AI Summary Generator
# -----------------------------
def generate_summary(text):
    if not text.strip():
        return ""
    try:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": f"Summarize this note in one short paragraph:\n{text}"}],
            max_tokens=120
        )
        return response.choices[0].message.content
    except:
        return "AI summary unavailable."

# -----------------------------
# Routes
# -----------------------------
@app.route("/")
def home():
    db = get_db()
    notes = db.execute("SELECT * FROM notes ORDER BY created_at DESC").fetchall()
    return render_template_string(BASE_HTML, content=render_template_string(HOME_HTML, notes=notes))

@app.route("/add", methods=["POST"])
def add_note():
    title = request.form.get("title")
    content = request.form.get("content")

    summary = generate_summary(content)
    db = get_db()
    db.execute("""
        INSERT INTO notes (title, content, summary, created_at)
        VALUES (?, ?, ?, ?)
    """, (title, content, summary, datetime.utcnow().isoformat()))
    db.commit()

    flash("Note added successfully.", "success")
    return redirect(url_for("home"))

@app.route("/note/<int:nid>")
def view_note(nid):
    db = get_db()
    note = db.execute("SELECT * FROM notes WHERE id = ?", (nid,)).fetchone()
    if not note:
        flash("Note not found.", "danger")
        return redirect(url_for("home"))
    return render_template_string(BASE_HTML, content=render_template_string(VIEW_NOTE_HTML, note=note))

@app.route("/note/<int:nid>/delete", methods=["POST"])
def delete_note(nid):
    db = get_db()
    db.execute("DELETE FROM notes WHERE id = ?", (nid,))
    db.commit()
    flash("Note deleted.", "info")
    return redirect(url_for("home"))

@app.route("/api/notes")
def api_notes():
    db = get_db()
    rows = db.execute("SELECT * FROM notes ORDER BY created_at DESC").fetchall()
    return jsonify([dict(r) for r in rows])

# -----------------------------
# Templates
# -----------------------------
BASE_HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>AI Notes App</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">

<nav class="navbar navbar-dark bg-dark">
    <div class="container">
        <a class="navbar-brand" href="/">AI Notes App</a>
    </div>
</nav>

<div class="container mt-4">
    {% with messages = get_flashed_messages(with_categories=true) %}
        {% for cat, msg in messages %}
            <div class="alert alert-{{ cat }}">{{ msg }}</div>
        {% endfor %}
    {% endwith %}

    {{ content|safe }}
</div>

</body>
</html>
"""

HOME_HTML = """
<div class="card shadow-sm">
    <div class="card-body">
        <h4>Create Note</h4>
        <form method="POST" action="/add">
            <input name="title" class="form-control my-2" placeholder="Title" required>
            <textarea name="content" class="form-control my-2" placeholder="Content" rows="4" required></textarea>
            <button class="btn btn-primary">Add Note</button>
        </form>
    </div>
</div>

<h4 class="mt-4">All Notes</h4>
<div class="row">
    {% for n in notes %}
    <div class="col-md-4">
        <div class="card my-2 shadow-sm">
            <div class="card-body">
                <h5>{{ n.title }}</h5>
                <p>{{ n.summary }}</p>
                <a class="btn btn-sm btn-dark" href="/note/{{ n.id }}">Open</a>
            </div>
        </div>
    </div>
    {% endfor %}
</div>
"""

VIEW_NOTE_HTML = """
<div class="card shadow-sm">
    <div class="card-body">
        <h3>{{ note.title }}</h3>
        <p class="text-muted">Created: {{ note.created_at }}</p>
        <hr>
        <h5>Content</h5>
        <p>{{ note.content }}</p>

        <h5 class="mt-4">AI Summary</h5>
        <p>{{ note.summary }}</p>

        <form method="POST" action="/note/{{ note.id }}/delete">
            <button class="btn btn-danger mt-3">Delete</button>
            <a href="/" class="btn btn-secondary mt-3">Back</a>
        </form>
    </div>
</div>
"""

# -----------------------------
# Run App
# -----------------------------
if __name__ == "__main__":
    app.run(debug=True)
