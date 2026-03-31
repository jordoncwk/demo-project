# Scriptwriter Tool Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local Flask web app that researches trending content, then generates a short-form video script, caption, and hashtags using the Claude API.

**Architecture:** Python + Flask backend with plain HTML/CSS/JS frontend. A multi-step wizard (topic → research → format → results) uses standard form submissions and server-side rendering. The Claude API handles all generation; DuckDuckGo search handles research. Data persists in local JSON files.

**Tech Stack:** Python 3.10+, Flask, anthropic SDK, duckduckgo-search, python-dotenv, pytest, pytest-flask

---

## Task 1: Project Setup

**Files:**
- Create: `requirements.txt`
- Create: `.env.example`
- Create: `app.py`
- Create: `src/__init__.py`
- Create: `tests/__init__.py`
- Create: `tests/conftest.py`
- Create: `.gitignore`

- [ ] **Step 1: Create requirements.txt**

```
flask==3.0.3
anthropic==0.28.0
duckduckgo-search==6.1.9
python-dotenv==1.0.1
pytest==8.2.2
pytest-flask==1.3.0
```

- [ ] **Step 2: Create .env.example**

```
ANTHROPIC_API_KEY=your-api-key-here
SECRET_KEY=change-this-to-a-random-string
```

- [ ] **Step 3: Create .gitignore**

```
.env
data/
__pycache__/
*.pyc
.pytest_cache/
.superpowers/
```

- [ ] **Step 4: Create src/__init__.py and tests/__init__.py**

Both files are empty. Just create them:

```python
# src/__init__.py  (empty)
```

```python
# tests/__init__.py  (empty)
```

- [ ] **Step 5: Create app.py**

```python
import os
from flask import Flask, render_template, request, redirect, url_for
from dotenv import load_dotenv
from src.storage import load_settings, save_settings, load_scripts, save_script
from src.research import research_topic
from src.generator import generate_script, generate_caption, generate_hashtags

load_dotenv()


def create_app(config=None):
    app = Flask(__name__)
    app.config['DATA_DIR'] = os.getenv('DATA_DIR', 'data')
    app.secret_key = os.getenv('SECRET_KEY', 'dev-secret')

    if config:
        app.config.update(config)

    @app.route('/')
    def dashboard():
        scripts = load_scripts(app.config['DATA_DIR'])
        return render_template('dashboard.html', scripts=scripts)

    @app.route('/settings', methods=['GET', 'POST'])
    def settings():
        if request.method == 'POST':
            data = {
                'niche': request.form['niche'],
                'tone': request.form['tone'],
                'brand_details': request.form['brand_details'],
                'platforms': request.form.getlist('platforms'),
            }
            save_settings(data, app.config['DATA_DIR'])
            return redirect(url_for('dashboard'))
        current = load_settings(app.config['DATA_DIR'])
        return render_template('settings.html', settings=current)

    @app.route('/create', methods=['GET', 'POST'])
    def create():
        settings = load_settings(app.config['DATA_DIR'])
        if request.method == 'POST':
            topic = request.form.get('topic', '').strip()
            research = research_topic(settings['niche'], topic or None)
            return render_template('create.html', settings=settings, research=research, topic=topic)
        return render_template('create.html', settings=settings, research=None, topic='')

    @app.route('/generate', methods=['POST'])
    def generate():
        settings = load_settings(app.config['DATA_DIR'])
        topic = request.form['topic']
        angle = request.form['angle']
        format_type = request.form['format_type']
        research_summary = request.form['research_summary']

        script = generate_script(settings['niche'], settings['tone'], angle, format_type, research_summary)
        caption = generate_caption(script, settings['niche'], settings['tone'])
        keywords = research_summary[:200]
        hashtags = generate_hashtags(topic, settings['niche'], keywords)

        entry = save_script({
            'topic': topic,
            'angle': angle,
            'format': format_type,
            'niche': settings['niche'],
            'tone': settings['tone'],
            'script': script,
            'caption': caption,
            'hashtags': hashtags,
            'research_summary': research_summary,
        }, app.config['DATA_DIR'])

        return redirect(url_for('results', script_id=entry['id']))

    @app.route('/results/<script_id>')
    def results(script_id):
        scripts = load_scripts(app.config['DATA_DIR'])
        entry = next((s for s in scripts if s['id'] == script_id), None)
        if entry is None:
            return redirect(url_for('dashboard'))
        return render_template('results.html', entry=entry)

    return app


app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
```

- [ ] **Step 6: Create tests/conftest.py**

```python
import pytest
from app import create_app


@pytest.fixture
def app(tmp_path):
    app = create_app({
        'TESTING': True,
        'DATA_DIR': str(tmp_path),
        'SECRET_KEY': 'test-secret',
    })
    yield app


@pytest.fixture
def client(app):
    return app.test_client()
```

- [ ] **Step 7: Install dependencies**

```bash
pip install -r requirements.txt
```

Expected: packages install without errors.

- [ ] **Step 8: Copy .env.example to .env and add your API key**

```bash
cp .env.example .env
```

Edit `.env` and replace `your-api-key-here` with your actual Anthropic API key from https://console.anthropic.com.

- [ ] **Step 9: Commit**

```bash
git add requirements.txt .env.example app.py src/__init__.py tests/__init__.py tests/conftest.py .gitignore
git commit -m "feat: project setup — Flask app factory, deps, test fixtures"
```

---

## Task 2: Storage Module

**Files:**
- Create: `src/storage.py`
- Create: `tests/test_storage.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_storage.py
import json
import os
import pytest
from src.storage import load_settings, save_settings, load_scripts, save_script


def test_load_settings_returns_defaults_when_file_missing(tmp_path):
    result = load_settings(str(tmp_path))
    assert result['niche'] == ''
    assert result['tone'] == 'casual'
    assert result['brand_details'] == ''
    assert result['platforms'] == ['tiktok']


def test_save_and_load_settings_roundtrip(tmp_path):
    data = {'niche': 'fitness', 'tone': 'hype', 'brand_details': 'gym owner', 'platforms': ['tiktok', 'reels']}
    save_settings(data, str(tmp_path))
    result = load_settings(str(tmp_path))
    assert result == data


def test_load_scripts_returns_empty_list_when_file_missing(tmp_path):
    result = load_scripts(str(tmp_path))
    assert result == []


def test_save_script_adds_id_and_created_at(tmp_path):
    entry = save_script({'topic': 'growth tips', 'script': 'Hello world'}, str(tmp_path))
    assert 'id' in entry
    assert 'created_at' in entry
    assert entry['topic'] == 'growth tips'


def test_save_script_persists_and_load_scripts_returns_it(tmp_path):
    save_script({'topic': 'test topic', 'script': 'test script'}, str(tmp_path))
    scripts = load_scripts(str(tmp_path))
    assert len(scripts) == 1
    assert scripts[0]['topic'] == 'test topic'


def test_save_script_prepends_newest_first(tmp_path):
    save_script({'topic': 'first', 'script': 'a'}, str(tmp_path))
    save_script({'topic': 'second', 'script': 'b'}, str(tmp_path))
    scripts = load_scripts(str(tmp_path))
    assert scripts[0]['topic'] == 'second'
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
pytest tests/test_storage.py -v
```

Expected: `ModuleNotFoundError: No module named 'src.storage'`

- [ ] **Step 3: Implement src/storage.py**

```python
import json
import os
import uuid
from datetime import datetime

_DEFAULTS = {
    'niche': '',
    'tone': 'casual',
    'brand_details': '',
    'platforms': ['tiktok'],
}


def _path(data_dir, filename):
    return os.path.join(data_dir, filename)


def load_settings(data_dir='data'):
    p = _path(data_dir, 'settings.json')
    if not os.path.exists(p):
        return dict(_DEFAULTS)
    with open(p) as f:
        return json.load(f)


def save_settings(settings, data_dir='data'):
    os.makedirs(data_dir, exist_ok=True)
    with open(_path(data_dir, 'settings.json'), 'w') as f:
        json.dump(settings, f, indent=2)


def load_scripts(data_dir='data'):
    p = _path(data_dir, 'scripts.json')
    if not os.path.exists(p):
        return []
    with open(p) as f:
        return json.load(f)


def save_script(script_data, data_dir='data'):
    scripts = load_scripts(data_dir)
    script_data['id'] = str(uuid.uuid4())
    script_data['created_at'] = datetime.now().isoformat()
    scripts.insert(0, script_data)
    os.makedirs(data_dir, exist_ok=True)
    with open(_path(data_dir, 'scripts.json'), 'w') as f:
        json.dump(scripts, f, indent=2)
    return script_data
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
pytest tests/test_storage.py -v
```

Expected: all 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/storage.py tests/test_storage.py
git commit -m "feat: storage module — load/save settings and scripts as JSON"
```

---

## Task 3: Research Module

**Files:**
- Create: `src/research.py`
- Create: `tests/test_research.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_research.py
from unittest.mock import patch, MagicMock
from src.research import research_topic, _build_queries


def _mock_results():
    return [
        {'title': 'Test Title', 'body': 'Test body text here', 'href': 'http://example.com'}
    ]


def test_build_queries_uses_topic_when_provided():
    queries = _build_queries('fitness', 'weight loss tips')
    assert 'weight loss tips' in queries['trends']
    assert 'fitness' in queries['competitors']


def test_build_queries_uses_niche_when_no_topic():
    queries = _build_queries('fitness', None)
    assert 'fitness' in queries['trends']


def test_research_topic_returns_three_categories():
    with patch('src.research.DDGS') as MockDDGS:
        instance = MockDDGS.return_value.__enter__.return_value
        instance.text.return_value = iter(_mock_results())
        result = research_topic('personal brand')
    assert 'trends' in result
    assert 'competitors' in result
    assert 'keywords' in result


def test_research_topic_each_category_has_title_and_body():
    with patch('src.research.DDGS') as MockDDGS:
        instance = MockDDGS.return_value.__enter__.return_value
        instance.text.return_value = iter(_mock_results())
        result = research_topic('personal brand', 'grow your audience')
    for category in ('trends', 'competitors', 'keywords'):
        assert len(result[category]) > 0
        assert 'title' in result[category][0]
        assert 'body' in result[category][0]
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
pytest tests/test_research.py -v
```

Expected: `ModuleNotFoundError: No module named 'src.research'`

- [ ] **Step 3: Implement src/research.py**

```python
from duckduckgo_search import DDGS


def research_topic(niche, topic=None):
    """Search for trending angles, competitor content, and keywords."""
    queries = _build_queries(niche, topic)
    results = {}

    with DDGS() as ddgs:
        for category, query in queries.items():
            hits = list(ddgs.text(query, max_results=4))
            results[category] = [
                {'title': r['title'], 'body': r['body'], 'href': r['href']}
                for r in hits
            ]

    return results


def _build_queries(niche, topic):
    base = topic if topic else niche
    return {
        'trends': f'trending {base} TikTok viral content ideas 2024',
        'competitors': f'top {niche} creators viral content strategy',
        'keywords': f'{base} most searched keywords social media',
    }
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
pytest tests/test_research.py -v
```

Expected: all 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/research.py tests/test_research.py
git commit -m "feat: research module — DuckDuckGo search for trends, competitors, keywords"
```

---

## Task 4: Generator Module

**Files:**
- Create: `src/generator.py`
- Create: `tests/test_generator.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_generator.py
from unittest.mock import patch, MagicMock
from src.generator import generate_script, generate_caption, generate_hashtags


def _mock_claude(text):
    response = MagicMock()
    response.content = [MagicMock(text=text)]
    return response


def _patched_anthropic(text):
    """Context manager that patches anthropic.Anthropic and returns mock response."""
    mock = MagicMock()
    mock.return_value.messages.create.return_value = _mock_claude(text)
    return patch('src.generator.anthropic.Anthropic', mock)


def test_generate_script_returns_string():
    with _patched_anthropic('Here is your script about growth.'):
        result = generate_script('personal brand', 'casual', 'growth tips', 'full_script', 'trending growth content')
    assert isinstance(result, str)
    assert 'growth' in result


def test_generate_script_accepts_all_format_types():
    for fmt in ('full_script', 'outline', 'hook_bullets'):
        with _patched_anthropic('Script content here.'):
            result = generate_script('business', 'professional', 'angle', fmt, 'summary')
        assert isinstance(result, str)


def test_generate_caption_returns_string():
    with _patched_anthropic('This is your caption. Follow for more tips!'):
        result = generate_caption('My video script here.', 'personal brand', 'casual')
    assert isinstance(result, str)
    assert len(result) > 0


def test_generate_hashtags_returns_list_of_tags():
    raw = '#personalbrand\n#growthtips\n#contentcreator'
    with _patched_anthropic(raw):
        result = generate_hashtags('growth tips', 'personal brand', 'brand growth audience')
    assert isinstance(result, list)
    assert all(tag.startswith('#') for tag in result)


def test_generate_hashtags_strips_empty_lines():
    raw = '#tag1\n\n#tag2\n#tag3\n'
    with _patched_anthropic(raw):
        result = generate_hashtags('topic', 'niche', 'keywords')
    assert len(result) == 3
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
pytest tests/test_generator.py -v
```

Expected: `ModuleNotFoundError: No module named 'src.generator'`

- [ ] **Step 3: Implement src/generator.py**

```python
import os
import anthropic


def _client():
    return anthropic.Anthropic(api_key=os.getenv('ANTHROPIC_API_KEY'))


def _call(prompt, max_tokens=1024):
    response = _client().messages.create(
        model='claude-sonnet-4-6',
        max_tokens=max_tokens,
        messages=[{'role': 'user', 'content': prompt}],
    )
    return response.content[0].text


def generate_script(niche, tone, angle, format_type, research_summary):
    format_instructions = {
        'full_script': 'Write a complete word-for-word script.',
        'outline': 'Write a structured outline with key talking points per section.',
        'hook_bullets': 'Write a strong hook, 3-5 bullet points for the body, and a clear CTA.',
    }
    prompt = f"""You are an expert short-form video scriptwriter for TikTok, YouTube Shorts, and Instagram Reels.

Niche: {niche}
Tone: {tone}
Content angle: {angle}
Research findings: {research_summary}

{format_instructions[format_type]}

Requirements:
- Hook must grab attention in the first 3 seconds
- Paced for a 60-90 second video
- Conversational — natural to speak aloud
- End with a clear call to action"""
    return _call(prompt, max_tokens=1024)


def generate_caption(script, niche, tone):
    prompt = f"""Write a platform-optimized social media caption for this short-form video.

Niche: {niche}
Tone: {tone}
Script: {script}

Requirements:
- 2-4 sentences
- Includes a natural call to action
- Matches the tone
- Do NOT include hashtags"""
    return _call(prompt, max_tokens=256)


def generate_hashtags(topic, niche, keywords):
    prompt = f"""Generate 12-15 hashtags for a short-form video.

Topic: {topic}
Niche: {niche}
Keywords: {keywords}

Return ONLY hashtags, one per line, starting with #.
Mix high-volume hashtags with niche-specific ones."""
    raw = _call(prompt, max_tokens=256)
    return [tag.strip() for tag in raw.strip().split('\n') if tag.strip().startswith('#')]
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
pytest tests/test_generator.py -v
```

Expected: all 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/generator.py tests/test_generator.py
git commit -m "feat: generator module — Claude API for script, caption, hashtag generation"
```

---

## Task 5: Base Template + Settings Page

**Files:**
- Create: `templates/base.html`
- Create: `templates/settings.html`
- Create: `tests/test_routes.py` (first tests only)

- [ ] **Step 1: Write failing route tests for settings**

```python
# tests/test_routes.py
def test_get_settings_returns_200(client):
    response = client.get('/settings')
    assert response.status_code == 200
    assert b'Settings' in response.data


def test_post_settings_saves_and_redirects(client, app):
    response = client.post('/settings', data={
        'niche': 'fitness',
        'tone': 'hype',
        'brand_details': 'gym owner',
        'platforms': ['tiktok'],
    }, follow_redirects=False)
    assert response.status_code == 302

    from src.storage import load_settings
    saved = load_settings(app.config['DATA_DIR'])
    assert saved['niche'] == 'fitness'
    assert saved['tone'] == 'hype'
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
pytest tests/test_routes.py -v
```

Expected: `jinja2.exceptions.TemplateNotFound: base.html`

- [ ] **Step 3: Create templates/base.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{% block title %}Scriptwriter{% endblock %}</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
  <nav>
    <a href="{{ url_for('dashboard') }}" class="nav-brand">ScriptWriter</a>
    <div class="nav-links">
      <a href="{{ url_for('create') }}">New Script</a>
      <a href="{{ url_for('settings') }}">Settings</a>
    </div>
  </nav>
  <main>
    {% block content %}{% endblock %}
  </main>
  <script src="{{ url_for('static', filename='app.js') }}"></script>
</body>
</html>
```

- [ ] **Step 4: Create templates/settings.html**

```html
{% extends "base.html" %}
{% block title %}Settings{% endblock %}
{% block content %}
<div class="page">
  <h1>Settings</h1>
  <p class="subtitle">These are used in every script and research query.</p>
  <form method="POST" action="{{ url_for('settings') }}">
    <label>Your Niche
      <input type="text" name="niche" value="{{ settings.niche }}" placeholder="e.g. personal brand and business" required>
    </label>
    <label>Tone
      <select name="tone">
        <option value="casual" {% if settings.tone == 'casual' %}selected{% endif %}>Casual</option>
        <option value="professional" {% if settings.tone == 'professional' %}selected{% endif %}>Professional</option>
        <option value="hype" {% if settings.tone == 'hype' %}selected{% endif %}>Hype / High Energy</option>
      </select>
    </label>
    <label>Brand Details
      <textarea name="brand_details" rows="3" placeholder="e.g. I'm a business coach helping 9-5 workers build side income">{{ settings.brand_details }}</textarea>
    </label>
    <fieldset>
      <legend>Platforms</legend>
      <label><input type="checkbox" name="platforms" value="tiktok" {% if 'tiktok' in settings.platforms %}checked{% endif %}> TikTok</label>
      <label><input type="checkbox" name="platforms" value="youtube_shorts" {% if 'youtube_shorts' in settings.platforms %}checked{% endif %}> YouTube Shorts</label>
      <label><input type="checkbox" name="platforms" value="instagram_reels" {% if 'instagram_reels' in settings.platforms %}checked{% endif %}> Instagram Reels</label>
    </fieldset>
    <button type="submit">Save Settings</button>
  </form>
</div>
{% endblock %}
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
pytest tests/test_routes.py -v
```

Expected: both tests PASS.

- [ ] **Step 6: Commit**

```bash
git add templates/base.html templates/settings.html tests/test_routes.py
git commit -m "feat: settings page — niche, tone, brand details form with routes"
```

---

## Task 6: Create Wizard (Topic Input + Research Results)

**Files:**
- Create: `templates/create.html`
- Modify: `tests/test_routes.py` (add create route tests)

- [ ] **Step 1: Add failing tests to tests/test_routes.py**

```python
def test_get_create_returns_200(client):
    response = client.get('/create')
    assert response.status_code == 200
    assert b'New Script' in response.data


def test_post_create_runs_research_and_shows_results(client):
    from unittest.mock import patch
    mock_research = {
        'trends': [{'title': 'Trend 1', 'body': 'Body 1', 'href': 'http://x.com'}],
        'competitors': [{'title': 'Comp 1', 'body': 'Body 2', 'href': 'http://y.com'}],
        'keywords': [{'title': 'Kw 1', 'body': 'Body 3', 'href': 'http://z.com'}],
    }
    with patch('app.research_topic', return_value=mock_research):
        response = client.post('/create', data={'topic': 'how to grow on TikTok'})
    assert response.status_code == 200
    assert b'Trend 1' in response.data
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
pytest tests/test_routes.py::test_get_create_returns_200 tests/test_routes.py::test_post_create_runs_research_and_shows_results -v
```

Expected: `jinja2.exceptions.TemplateNotFound: create.html`

- [ ] **Step 3: Create templates/create.html**

```html
{% extends "base.html" %}
{% block title %}New Script{% endblock %}
{% block content %}
<div class="page">
  <h1>New Script</h1>

  {% if not research %}
  <!-- Step 1: Topic input -->
  <form method="POST" action="{{ url_for('create') }}">
    <label>What's your video about?
      <input type="text" name="topic" value="{{ topic }}" placeholder="Leave blank to get ideas based on your niche">
    </label>
    <button type="submit">Research This Topic</button>
  </form>
  {% if not settings.niche %}
  <p class="warning">You haven't set your niche yet. <a href="{{ url_for('settings') }}">Set it in Settings</a> for better results.</p>
  {% endif %}

  {% else %}
  <!-- Step 2: Research results + format selection -->
  <h2>Research Findings</h2>
  <p class="subtitle">Topic: <strong>{{ topic or settings.niche }}</strong> — pick an angle below.</p>

  <div class="research-grid">
    <div class="research-col">
      <h3>Trending Now</h3>
      {% for item in research.trends %}
      <div class="research-card">
        <strong>{{ item.title }}</strong>
        <p>{{ item.body[:150] }}...</p>
      </div>
      {% endfor %}
    </div>
    <div class="research-col">
      <h3>Competitor Angles</h3>
      {% for item in research.competitors %}
      <div class="research-card">
        <strong>{{ item.title }}</strong>
        <p>{{ item.body[:150] }}...</p>
      </div>
      {% endfor %}
    </div>
    <div class="research-col">
      <h3>Keywords</h3>
      {% for item in research.keywords %}
      <div class="research-card">
        <strong>{{ item.title }}</strong>
        <p>{{ item.body[:150] }}...</p>
      </div>
      {% endfor %}
    </div>
  </div>

  <form method="POST" action="{{ url_for('generate') }}">
    <input type="hidden" name="topic" value="{{ topic }}">
    <input type="hidden" name="research_summary" value="{{ research | string }}">

    <label>Your angle / hook idea
      <input type="text" name="angle" required placeholder="e.g. 3 mistakes I made growing my personal brand">
    </label>

    <fieldset>
      <legend>Script format</legend>
      <label><input type="radio" name="format_type" value="full_script" checked> Full word-for-word script</label>
      <label><input type="radio" name="format_type" value="outline"> Structured outline</label>
      <label><input type="radio" name="format_type" value="hook_bullets"> Hook + bullet points + CTA</label>
    </fieldset>

    <button type="submit">Generate Script</button>
  </form>
  {% endif %}
</div>
{% endblock %}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
pytest tests/test_routes.py -v
```

Expected: all 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add templates/create.html tests/test_routes.py
git commit -m "feat: create wizard — topic input, research results, format selection"
```

---

## Task 7: Generate Route + Results Page

**Files:**
- Create: `templates/results.html`
- Modify: `tests/test_routes.py` (add generate + results tests)

- [ ] **Step 1: Add failing tests to tests/test_routes.py**

```python
def test_post_generate_saves_script_and_redirects(client, app):
    from unittest.mock import patch
    with patch('app.generate_script', return_value='Your script here.'), \
         patch('app.generate_caption', return_value='Your caption here.'), \
         patch('app.generate_hashtags', return_value=['#test', '#brand']):
        response = client.post('/generate', data={
            'topic': 'grow on TikTok',
            'angle': '3 mistakes I made',
            'format_type': 'full_script',
            'research_summary': 'trending content about growth',
        }, follow_redirects=False)
    assert response.status_code == 302
    assert '/results/' in response.headers['Location']


def test_get_results_shows_script(client, app):
    from unittest.mock import patch
    from src.storage import save_script
    entry = save_script({
        'topic': 'test topic',
        'script': 'My test script.',
        'caption': 'My caption.',
        'hashtags': ['#test'],
        'angle': 'test angle',
        'format': 'full_script',
        'niche': 'personal brand',
        'tone': 'casual',
        'research_summary': 'some research',
    }, app.config['DATA_DIR'])
    response = client.get(f"/results/{entry['id']}")
    assert response.status_code == 200
    assert b'My test script.' in response.data
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
pytest tests/test_routes.py::test_post_generate_saves_script_and_redirects tests/test_routes.py::test_get_results_shows_script -v
```

Expected: `jinja2.exceptions.TemplateNotFound: results.html`

- [ ] **Step 3: Create templates/results.html**

```html
{% extends "base.html" %}
{% block title %}Script — {{ entry.topic }}{% endblock %}
{% block content %}
<div class="page">
  <div class="results-header">
    <h1>{{ entry.topic }}</h1>
    <p class="subtitle">{{ entry.angle }} · {{ entry.format.replace('_', ' ').title() }} · {{ entry.tone.title() }}</p>
    <a href="{{ url_for('create') }}" class="btn-secondary">New Script</a>
  </div>

  <section class="result-block">
    <div class="result-label">
      <h2>Script</h2>
      <button class="copy-btn" data-target="script-content">Copy</button>
    </div>
    <pre id="script-content" class="result-text">{{ entry.script }}</pre>
  </section>

  <section class="result-block">
    <div class="result-label">
      <h2>Caption</h2>
      <button class="copy-btn" data-target="caption-content">Copy</button>
    </div>
    <p id="caption-content" class="result-text">{{ entry.caption }}</p>
  </section>

  <section class="result-block">
    <div class="result-label">
      <h2>Hashtags</h2>
      <button class="copy-btn" data-target="hashtags-content">Copy</button>
    </div>
    <p id="hashtags-content" class="result-text">{{ entry.hashtags | join(' ') }}</p>
  </section>

  <section class="result-block">
    <h2>Regenerate</h2>
    <form method="POST" action="{{ url_for('generate') }}">
      <input type="hidden" name="topic" value="{{ entry.topic }}">
      <input type="hidden" name="angle" value="{{ entry.angle }}">
      <input type="hidden" name="format_type" value="{{ entry.format }}">
      <input type="hidden" name="research_summary" value="{{ entry.research_summary }}">
      <button type="submit">Regenerate All</button>
    </form>
  </section>

  <p class="saved-note">Saved · {{ entry.created_at[:10] }}</p>
</div>
{% endblock %}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
pytest tests/test_routes.py -v
```

Expected: all 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add templates/results.html tests/test_routes.py
git commit -m "feat: generate route and results page — script, caption, hashtags display"
```

---

## Task 8: Dashboard (Script History)

**Files:**
- Create: `templates/dashboard.html`
- Modify: `tests/test_routes.py` (add dashboard test)

- [ ] **Step 1: Add failing test to tests/test_routes.py**

```python
def test_dashboard_shows_saved_scripts(client, app):
    from src.storage import save_script
    save_script({'topic': 'grow on TikTok', 'script': 's', 'caption': 'c',
                 'hashtags': [], 'angle': 'a', 'format': 'full_script',
                 'niche': 'brand', 'tone': 'casual', 'research_summary': 'r'},
                app.config['DATA_DIR'])
    response = client.get('/')
    assert response.status_code == 200
    assert b'grow on TikTok' in response.data


def test_dashboard_empty_state(client):
    response = client.get('/')
    assert response.status_code == 200
    assert b'No scripts yet' in response.data
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
pytest tests/test_routes.py::test_dashboard_shows_saved_scripts tests/test_routes.py::test_dashboard_empty_state -v
```

Expected: `jinja2.exceptions.TemplateNotFound: dashboard.html`

- [ ] **Step 3: Create templates/dashboard.html**

```html
{% extends "base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<div class="page">
  <div class="dashboard-header">
    <h1>Your Scripts</h1>
    <a href="{{ url_for('create') }}" class="btn-primary">+ New Script</a>
  </div>

  {% if scripts %}
  <div class="script-list">
    {% for s in scripts %}
    <a href="{{ url_for('results', script_id=s.id) }}" class="script-card">
      <div class="script-card-title">{{ s.topic }}</div>
      <div class="script-card-meta">{{ s.format.replace('_', ' ').title() }} · {{ s.tone.title() }} · {{ s.created_at[:10] }}</div>
    </a>
    {% endfor %}
  </div>
  {% else %}
  <div class="empty-state">
    <p>No scripts yet. <a href="{{ url_for('create') }}">Create your first one.</a></p>
  </div>
  {% endif %}
</div>
{% endblock %}
```

- [ ] **Step 4: Run all tests — verify they pass**

```bash
pytest -v
```

Expected: all 8 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add templates/dashboard.html tests/test_routes.py
git commit -m "feat: dashboard — script history list with empty state"
```

---

## Task 9: Static Assets (CSS + Copy Button JS)

**Files:**
- Create: `static/style.css`
- Create: `static/app.js`

No automated tests for visual styling. Manual verification instead.

- [ ] **Step 1: Create static/style.css**

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #f8f9fa;
  color: #1a1a2e;
  line-height: 1.6;
}

nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 2rem;
  background: #1a1a2e;
  color: white;
}

.nav-brand { color: white; text-decoration: none; font-size: 1.2rem; font-weight: 700; }
.nav-links a { color: #ccc; text-decoration: none; margin-left: 1.5rem; }
.nav-links a:hover { color: white; }

main { max-width: 900px; margin: 0 auto; padding: 2rem 1rem; }

.page h1 { font-size: 1.8rem; margin-bottom: 0.5rem; }
.subtitle { color: #666; margin-bottom: 1.5rem; }

label { display: block; font-weight: 600; margin-bottom: 1rem; }
label input, label select, label textarea {
  display: block; width: 100%; margin-top: 0.25rem;
  padding: 0.6rem 0.8rem; border: 1px solid #ddd; border-radius: 6px;
  font-size: 1rem; font-family: inherit;
}

fieldset { border: 1px solid #ddd; border-radius: 6px; padding: 1rem; margin-bottom: 1rem; }
legend { font-weight: 600; padding: 0 0.5rem; }
fieldset label { font-weight: normal; display: inline-flex; align-items: center; gap: 0.4rem; margin-right: 1.5rem; margin-bottom: 0; }

button[type="submit"], .btn-primary {
  background: #6c5ce7; color: white; border: none; padding: 0.75rem 1.5rem;
  border-radius: 6px; font-size: 1rem; cursor: pointer; text-decoration: none; display: inline-block;
}
button[type="submit"]:hover, .btn-primary:hover { background: #5a4dd4; }

.btn-secondary {
  color: #6c5ce7; border: 1px solid #6c5ce7; background: transparent;
  padding: 0.5rem 1rem; border-radius: 6px; text-decoration: none; font-size: 0.9rem;
}

.copy-btn {
  background: transparent; border: 1px solid #ddd; color: #666;
  padding: 0.25rem 0.75rem; border-radius: 4px; cursor: pointer; font-size: 0.85rem;
}
.copy-btn:hover { border-color: #6c5ce7; color: #6c5ce7; }
.copy-btn.copied { color: green; border-color: green; }

/* Dashboard */
.dashboard-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; }
.script-list { display: flex; flex-direction: column; gap: 0.75rem; }
.script-card {
  display: block; text-decoration: none; color: inherit;
  background: white; border: 1px solid #eee; border-radius: 8px; padding: 1rem 1.25rem;
}
.script-card:hover { border-color: #6c5ce7; }
.script-card-title { font-weight: 600; margin-bottom: 0.25rem; }
.script-card-meta { font-size: 0.85rem; color: #888; }
.empty-state { text-align: center; padding: 3rem; color: #888; }
.empty-state a { color: #6c5ce7; }

/* Research grid */
.research-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 1rem; margin-bottom: 2rem; }
.research-col h3 { font-size: 0.9rem; text-transform: uppercase; letter-spacing: 0.05em; color: #888; margin-bottom: 0.75rem; }
.research-card { background: white; border: 1px solid #eee; border-radius: 6px; padding: 0.75rem; margin-bottom: 0.5rem; font-size: 0.9rem; }
.research-card strong { display: block; margin-bottom: 0.25rem; }
.research-card p { color: #666; }

/* Results */
.results-header { display: flex; flex-wrap: wrap; align-items: baseline; gap: 1rem; margin-bottom: 2rem; }
.result-block { background: white; border: 1px solid #eee; border-radius: 8px; padding: 1.25rem; margin-bottom: 1.5rem; }
.result-label { display: flex; justify-content: space-between; align-items: center; margin-bottom: 0.75rem; }
.result-label h2 { font-size: 1rem; font-weight: 600; }
.result-text { white-space: pre-wrap; font-size: 0.95rem; line-height: 1.7; font-family: inherit; }
.saved-note { text-align: right; font-size: 0.8rem; color: #aaa; }

.warning { color: #e17055; font-size: 0.9rem; margin-top: 0.5rem; }
```

- [ ] **Step 2: Create static/app.js**

```javascript
document.addEventListener('DOMContentLoaded', function () {
  document.querySelectorAll('.copy-btn').forEach(function (btn) {
    btn.addEventListener('click', function () {
      var targetId = btn.getAttribute('data-target');
      var el = document.getElementById(targetId);
      if (!el) return;
      navigator.clipboard.writeText(el.innerText).then(function () {
        btn.textContent = 'Copied!';
        btn.classList.add('copied');
        setTimeout(function () {
          btn.textContent = 'Copy';
          btn.classList.remove('copied');
        }, 2000);
      });
    });
  });
});
```

- [ ] **Step 3: Run all tests to confirm nothing broke**

```bash
pytest -v
```

Expected: all 8 tests PASS.

- [ ] **Step 4: Manual smoke test**

```bash
python app.py
```

Open `http://localhost:5000` in your browser.

Verify:
1. Dashboard loads with "No scripts yet" message
2. Clicking Settings shows the settings form
3. Saving settings redirects to dashboard
4. Clicking New Script shows the topic input form
5. App looks styled (not raw HTML)

- [ ] **Step 5: Commit**

```bash
git add static/style.css static/app.js
git commit -m "feat: CSS styles and copy-to-clipboard JS"
```

---

## Task 10: End-to-End Smoke Test

This task verifies the full wizard works with a real API call.

- [ ] **Step 1: Confirm .env has your API key**

```bash
python -c "import os; from dotenv import load_dotenv; load_dotenv(); print('Key set:', bool(os.getenv('ANTHROPIC_API_KEY')))"
```

Expected: `Key set: True`

- [ ] **Step 2: Run the app and complete a full flow**

```bash
python app.py
```

1. Go to `http://localhost:5000/settings` — enter your niche, tone, brand details. Save.
2. Go to `http://localhost:5000/create` — type a topic (or leave blank). Click "Research This Topic".
3. Review research results. Type an angle. Choose a format. Click "Generate Script".
4. Verify results page shows script, caption, and hashtags.
5. Click each "Copy" button — verify it copies to clipboard and shows "Copied!".
6. Go back to dashboard — verify your script appears in the list.
7. Click the script card — verify it reopens the results page.

- [ ] **Step 3: Run full test suite one final time**

```bash
pytest -v
```

Expected: all 8 tests PASS.

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "feat: scriptwriter tool complete — research, script, caption, hashtag generation"
```
