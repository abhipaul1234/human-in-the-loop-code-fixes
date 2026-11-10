# Lumina Academic: fixes

Flask study tracker: users log study sessions, test scores and wellness data, and get AI-written insights and charts.

## Logic & data

### A database function that only exists in SQLite breaks the charts on PostgreSQL (High)

**Why it's a problem.** The "last 7 days" filter for the dashboard chart was written as raw SQL using `datetime('now', '-7 days')`, which is a SQLite function. It worked on the development database, but on the PostgreSQL production database the query fails, so the charts break for every user.

**AI-written code**

```python
from sqlalchemy import text
daily_query = db.session.query(
    db.func.date(StudySession.created_at).label('date'),
    db.func.sum(StudySession.duration).label('total_minutes')
).filter(
    StudySession.created_at >= text("datetime('now', '-7 days')")   # SQLite only
).group_by(db.func.date(StudySession.created_at))
```

**Fixed code**

```diff
-    from sqlalchemy import text
+    # cutoff computed in Python: works on SQLite and PostgreSQL alike
+    week_ago = datetime.utcnow() - timedelta(days=7)
     daily_query = db.session.query(
         db.func.date(StudySession.created_at).label('date'),
         db.func.sum(StudySession.duration).label('total_minutes')
-    ).filter(
-        StudySession.created_at >= text("datetime('now', '-7 days')")
+    ).filter(
+        StudySession.created_at >= week_ago
     ).group_by(db.func.date(StudySession.created_at))
```

**Status.** Fixed (untested)

**Lesson.** Test against the database you deploy on, and keep dates and SQL functions portable.

### Form numbers are not checked, so bad input crashes or divides by zero (Moderate)

**Why it's a problem.** The logging forms converted fields with `int(...)` and `float(...)` straight from the request, so an empty field raised an error (a 500 page). Ratings were never limited to 1 to 5, and a maximum score of 0 was accepted and later caused a division by zero when the percentage was shown.

**AI-written code**

```python
duration = int(request.form['duration'])
focus_rating = int(request.form['focus_rating'])
score = float(request.form['score'])
max_score = float(request.form['max_score'])   # 0 is accepted

@property
def percentage(self):
    return round((self.score / self.max_score) * 100, 1)
```

**Fixed code**

```python
def form_number(name, cast, low=None, high=None, default=None):
    """A number from the submitted form, checked against its allowed range."""
    raw = (request.form.get(name) or '').strip()
    if raw == '' and default is not None:
        return default
    try:
        value = cast(raw)
    except (TypeError, ValueError):
        raise ValueError(name)
    if value != value or (low is not None and value < low) or (high is not None and value > high):
        raise ValueError(name)
    return value
```

```diff
-        duration = int(request.form['duration'])
-        difficulty_level = int(request.form['difficulty_level'])
-        focus_rating = int(request.form['focus_rating'])
+        try:
+            duration = form_number('duration', int, 1, 24 * 60)
+            difficulty_level = form_number('difficulty_level', int, 1, 5)
+            focus_rating = form_number('focus_rating', int, 1, 5)
+        except ValueError as e:
+            return invalid_field(str(e), 'log_study.html', 'Log Study Session')
```

```diff
-        score = float(request.form['score'])
-        max_score = float(request.form['max_score'])
+        try:
+            max_score = form_number('max_score', float, 0)
+            if max_score <= 0:
+                raise ValueError('max_score')
+            score = form_number('score', float, 0, max_score)
+        except ValueError as e:
+            return invalid_field(str(e), 'log_performance.html', 'Log Performance')
```

```diff
     def percentage(self):
-        return round((self.score / self.max_score) * 100, 1)
+        if not self.max_score:  # rows saved before max_score was validated
+            return 0.0
+        return round((self.score / self.max_score) * 100, 1)
```

**Status.** Fixed (untested)

**Lesson.** Check type, range and "cannot be zero" on every number a form sends, and answer bad input with a message, not an error page.

## Reliability & quality

### The app quietly starts on a throw-away database (High)

**Why it's a problem.** On a read-only host, creating the local `instance` folder fails, and the code answered by switching to a SQLite file in the temp folder and printing a warning. Serverless instances do not share or keep that folder, so accounts and logged data disappear between requests or deploys while the app looks like it works.

**AI-written code**

```python
try:
    os.makedirs('instance', exist_ok=True)
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///YOUR_DB_FILE.db'
except OSError:
    # read-only host: use a throwaway database instead of crashing
    temp_db = os.path.join(tempfile.gettempdir(), 'YOUR_DB_FILE.db')
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + temp_db
    print("Warning: using a temporary database.")
```

**Fixed code**

```diff
     except OSError:
-        # Read-only hosts such as Vercel cannot create folders next to the code. Start with a throwaway
-        # database in the temp folder instead of crashing; set DATABASE_URL to keep data permanently.
+        # A database in the temp folder is not shared between instances and is wiped on restart, so every
+        # account and log would silently disappear. Refuse to start unless that is explicitly wanted (demos).
+        if os.getenv('ALLOW_TEMP_DATABASE') != '1':
+            raise RuntimeError('DATABASE_URL must point to a PostgreSQL database on this host '
+                               '(set ALLOW_TEMP_DATABASE=1 only for a throwaway demo).')
         temp_db = os.path.join(tempfile.gettempdir(), 'YOUR_DB_FILE.db')
```

**Status.** Fixed (untested)

**Lesson.** Fail loudly when a required resource is missing; a silent fallback that loses data is worse than a crash.

### The AI integration uses a retired SDK and model name (Moderate)

**Why it's a problem.** The AI insights page used the old `google-generativeai` package and a `gemini-1.5-flash` model name. That model has been retired, so the page always fell back to its error text. The call also had no timeout, so a slow reply blocked the request.

**AI-written code**

```python
import google.generativeai as genai
...
model = genai.GenerativeModel('gemini-1.5-flash-latest')
...
response = model.generate_content(prompt)
```

**Fixed code**

```diff
-import google.generativeai as genai
+from google import genai
```

```diff
+GEMINI_MODEL = os.getenv('GEMINI_MODEL', 'gemini-2.5-flash')   # a retirement becomes a config change
-    model = genai.GenerativeModel('gemini-1.5-flash-latest')
+    # client created with the current SDK and a 30 second timeout
```

```diff
-        response = model.generate_content(prompt)
+        response = model.models.generate_content(model=GEMINI_MODEL, contents=prompt)
```

```diff
-google-generativeai==0.8.1
+google-genai>=1.0,<2  # replaces the retired google-generativeai SDK
```

**Status.** Fixed (untested)

**Lesson.** Model names and SDKs age quickly: make the model name configurable and add a timeout to every outside call.
