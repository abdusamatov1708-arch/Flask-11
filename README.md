# Flask-11
from flask import Flask, render_template_string, request, redirect, url_for, session, flash, abort
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
from functools import wraps

app = Flask(__name__)
app.secret_key = 'shaxsiy-notalar-ilovasi-maxfiy-kaliti-2026'

app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///personal_notes.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)


class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    notes = db.relationship('Note', backref='owner', lazy=True, cascade='all, delete-orphan')


class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    body = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


def current_user():
    user_id = session.get('user_id')
    return User.query.get(user_id) if user_id else None


def login_required(view):
    """Login bo'lmagan foydalanuvchilarni himoya qilish uchun dekorator"""

    @wraps(view)
    def wrapped(*args, **kwargs):
        if not current_user():
            flash("Ushbu sahifani ko'rish uchun avval tizimga kiring!", "error")
            return redirect(url_for('login'))
        return view(*args, **kwargs)

    return wrapped


@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user():
        return redirect(url_for('list_notes'))

    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        if len(username) < 2:
            flash("Ism kamida 2 ta belgidan iborat bo'lishi kerak!", "error")
            return redirect(url_for('login'))

        user = User.query.filter_by(username=username).first()
        if not user:
            user = User(username=username)
            db.session.add(user)
            db.session.commit()

        session['user_id'] = user.id
        flash(f"Xush kelibsiz, {username}!", "success")
        return redirect(url_for('list_notes'))

    return render_template_string(LOGIN_HTML)


@app.route('/logout')
def logout():
    session.clear()
    flash("Tizimdan muvaffaqiyatli chiqdingiz.", "success")
    return redirect(url_for('login'))


@app.route('/')
@app.route('/notes')
@login_required
def list_notes():
    user = current_user()
    notes = Note.query.filter_by(user_id=user.id).order_by(Note.created_at.desc()).all()
    return render_template_string(NOTES_HTML, notes=notes, user=user)


@app.route('/notes/new', methods=['GET', 'POST'])
@login_required
def new_note():
    if request.method == 'POST':
        title = request.form.get('title', '').strip()
        body = request.form.get('body', '').strip()

        if not title or not body:
            flash("Sarlavha va matn to'ldirilishi shart!", "error")
            return redirect(url_for('new_note'))

        note = Note(title=title, body=body, user_id=current_user().id)
        db.session.add(note)
        db.session.commit()

        flash("Yangi nota muvaffaqiyatli qo'shildi!", "success")
        return redirect(url_for('list_notes'))

    return render_template_string(NEW_NOTE_HTML)


@app.route('/notes/<int:note_id>/edit', methods=['GET', 'POST'])
@login_required
def edit_note(note_id):
    note = Note.query.get_or_404(note_id)

    if note.user_id != current_user().id:
        abort(403)

    if request.method == 'POST':
        note.title = request.form.get('title', '').strip()
        note.body = request.form.get('body', '').strip()

        if not note.title or not note.body:
            flash("Maydonlarni bo'sh qoldirmang!", "error")
            return redirect(url_for('edit_note', note_id=note.id))

        db.session.commit()
        flash("Nota muvaffaqiyatli yangilandi!", "success")
        return redirect(url_for('list_notes'))

    return render_template_string(EDIT_NOTE_HTML, note=note)


@app.route('/notes/<int:note_id>/delete', methods=['POST'])
@login_required
def delete_note(note_id):
    note = Note.query.get_or_404(note_id)

    if note.user_id != current_user().id:
        abort(403)

    db.session.delete(note)
    db.session.commit()
    flash("Nota muvaffaqiyatli o'chirildi!", "success")
    return redirect(url_for('list_notes'))



BASE_CSS = '''
body { font-family: 'Segoe UI', Arial, sans-serif; max-width: 700px; margin: 40px auto; padding: 0 20px; background-color: #f8f9fa; color: #333; }
.flash { padding: 12px; margin-bottom: 20px; border-radius: 5px; font-weight: 500; }
.success { background-color: #d4edda; color: #155724; border-left: 5px solid #28a745; }
.error { background-color: #f8d7da; color: #721c24; border-left: 5px solid #dc3545; }
.note-card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); margin-bottom: 15px; border-left: 4px solid #007bff; }
.btn { display: inline-block; padding: 8px 14px; border: none; border-radius: 4px; cursor: pointer; text-decoration: none; font-size: 0.9em; font-weight: bold; }
.btn-primary { background: #007bff; color: white; }
.btn-danger { background: #dc3545; color: white; }
.btn-secondary { background: #6c757d; color: white; }
input[type="text"], textarea { width: 100%; padding: 10px; margin: 8px 0 15px 0; box-sizing: border-box; border: 1px solid #ced4da; border-radius: 4px; }
textarea { height: 120px; }
.meta { font-size: 0.8em; color: #6c757d; margin-top: 10px; }
.header-box { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
'''

FLASH_BLOCK = '''
{% with messages = get_flashed_messages(with_categories=true) %}
  {% if messages %}
    {% for category, message in messages %}
      <div class="flash {{ category }}">{{ message }}</div>
    {% endfor %}
  {% endif %}
{% endwith %}
'''

LOGIN_HTML = f'''
<!DOCTYPE html>
<html>
<head><title>Kirish</title><style>{BASE_CSS}</style></head>
<body>
    <div style="max-width: 400px; margin: 100px auto; background: white; padding: 30px; border-radius: 8px; box-shadow: 0 4px 10px rgba(0,0,0,0.08);">
        <h2 style="margin-top:0; text-align:center;">🔐 Shaxsiy Notalar</h2>
        {FLASH_BLOCK}
        <form method="POST">
            <label>Foydalanuvchi ismi:</label>
            <input type="text" name="username" placeholder="Ismingizni kiriting" required minlength="2">
            <button type="submit" class="btn btn-primary" style="width: 100%; padding: 10px;">Tizimga kirish / Ro'yxatdan o'tish</button>
        </form>
    </div>
</body>
</html>
'''

NOTES_HTML = f'''
<!DOCTYPE html>
<html>
<head><title>Mening Notalarim</title><style>{BASE_CSS}</style></head>
<body>
    <div class="header-box">
        <h2>👤 {{{{ user.username }}}} ning notalari</h2>
        <div>
            <a href="{{{{ url_for('new_note') }}}}" class="btn btn-primary">+ Yangi nota</a>
            <a href="{{{{ url_for('logout') }}}}" class="btn btn-secondary">Chiqish</a>
        </div>
    </div>
    {FLASH_BLOCK}
    <hr>
    {{% for n in notes %}}
      <div class="note-card">
        <h3 style="margin-top: 0; color: #212529;">{{{{ n.title }}}}</h3>
        <p style="white-space: pre-line; color: #495057;">{{{{ n.body }}}}</p>
        <div class="meta">🕒 {{{{ n.created_at.strftime('%d-%b %H:%M') }}}}</div>
        <div style="margin-top: 15px; display: flex; gap: 8px;">
            <a href="{{{{ url_for('edit_note', note_id=n.id) }}}}" class="btn btn-secondary" style="padding: 5px 10px; font-size: 0.85em;">✎ Tahrirlash</a>
            <form method="POST" action="{{{{ url_for('delete_note', note_id=n.id) }}}}" style="display:inline;" onsubmit="return confirm('Haqiqatan ham bu notani ocha olasizmi?');">
                <button type="submit" class="btn btn-danger" style="padding: 5px 10px; font-size: 0.85em;">🗑 O'chirish</button>
            </form>
        </div>
      </div>
    {{% else %}}
      <p style="text-align: center; color: #6c757d; margin-top: 40px;">Hozircha notalaringiz mavjud emas. Birinchi notangizni qo'shing!</p>
    {{% endfor %}}
</body>
</html>
'''

NEW_NOTE_HTML = f'''
<!DOCTYPE html>
<html>
<head><title>Yangi Nota</title><style>{BASE_CSS}</style></head>
<body>
    <h2>📝 Yangi nota yaratish</h2>
    {FLASH_BLOCK}
    <div style="background: white; padding: 25px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.05);">
        <form method="POST">
          <label>Sarlavha:</label>
          <input type="text" name="title" placeholder="Nota sarlavhasi" required><br>
          <label>Matn:</label>
          <textarea name="body" placeholder="Fikrlaringizni shu yerga yozing..." required></textarea><br>
          <button class="btn btn-primary">Saqlash</button>
          <a href="{{{{ url_for('list_notes') }}}}" class="btn btn-secondary">Bekor qilish</a>
        </form>
    </div>
</body>
</html>
'''

EDIT_NOTE_HTML = f'''
<!DOCTYPE html>
<html>
<head><title>Notani tahrirlash</title><style>{BASE_CSS}</style></head>
<body>
    <h2>✎ Notani tahrirlash</h2>
    {FLASH_BLOCK}
    <div style="background: white; padding: 25px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.05);">
        <form method="POST">
          <label>Sarlavha:</label>
          <input type="text" name="title" value="{{{{ note.title }}}}" required><br>
          <label>Matn:</label>
          <textarea name="body" required>{{{{ note.body }}}}</textarea><br>
          <button class="btn btn-primary">Yangilash</button>
          <a href="{{{{ url_for('list_notes') }}}}" class="btn btn-secondary">Bekor qilish</a>
        </form>
    </div>
</body>
</html>
'''

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
