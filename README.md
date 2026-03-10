<think>
**Preparing final code structure**

I need to deliver the final code, which includes server.js, package.json, public/index.html, and possibly .gitignore and .env.example. The example should be runnable with modern Node 18+ and include security features. I'll make sure to format code blocks correctly and remove emojis unless specified otherwise, particularly since the user's site is in Arabic. The final code should support auto-detection, clipboard copying, a responsive grid layout, proper routing, and CORS settings. I'll also include rate limiting and language validation.**Finalizing language detection and translation features**

I’m working on the regex pattern to handle language codes, particularly allowing for codes like 'zh-CN' along with other 2-3 character languages. I’ll implement an auto language detection option and ensure that timeout handling works using an abort controller for upstream calls. In the frontend, I’ll show the detected language, debounce input, and disable the translate button while loading. I’ll keep track of the last detected language and manage character counting. I’ll also make sure to include basic instructions for running the project along with the necessary files.
</think>فيما يلي **الكود النهائي الكامل** (خدمة واحدة على Render: الواجهة + API في نفس السيرفر). انسخه كما هو بنفس البنية.

---

## هيكل المشروع
```
translator-pro/
  package.json
  server.js
  .env.example
  .gitignore
  public/
    index.html
```

---

## 1) package.json
```json
{
  "name": "translator-pro",
  "version": "1.0.0",
  "private": true,
  "type": "commonjs",
  "main": "server.js",
  "engines": {
    "node": ">=18"
  },
  "scripts": {
    "dev": "node --watch server.js",
    "start": "node server.js"
  },
  "dependencies": {
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "express-rate-limit": "^7.4.1",
    "he": "^1.2.0",
    "helmet": "^7.1.0",
    "lru-cache": "^11.0.1"
  }
}
```

---

## 2) server.js
```js
const path = require('path');
const express = require('express');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const { LRUCache } = require('lru-cache');
const he = require('he');
require('dotenv').config();

const app = express();

// مهم على Render مع rate-limit (لأن وراء Proxy)
app.set('trust proxy', 1);

const PORT = process.env.PORT || 3000;
const API_KEY = process.env.GOOGLE_TRANSLATE_API_KEY;

if (!API_KEY) {
  console.error('Missing GOOGLE_TRANSLATE_API_KEY in environment variables.');
  process.exit(1);
}

app.use(helmet());
app.use(express.json({ limit: '1mb' }));

// Rate limit لحماية الـ API وتقليل الإسراف
app.use('/api/', rateLimit({
  windowMs: 60 * 1000,
  limit: 60,
  standardHeaders: 'draft-7',
  legacyHeaders: false
}));

// Cache لتقليل التكلفة (10 دقائق)
const cache = new LRUCache({
  max: 500,
  ttl: 1000 * 60 * 10
});

function isValidLangCode(code) {
  // يدعم: en, ar, fr, zh-CN, pt, fil... (2-3 أحرف + اختيارياً -XX)
  return typeof code === 'string' && /^[a-z]{2,3}(-[A-Z]{2})?$/.test(code);
}

async function googleTranslate({ text, from, to }) {
  const url = `https://translation.googleapis.com/language/translate/v2?key=${encodeURIComponent(API_KEY)}`;

  const payload = {
    q: text,
    target: to,
    format: 'text'
  };

  // إذا from=auto لا نرسل source لتفعيل الكشف التلقائي
  if (from && from !== 'auto') payload.source = from;

  // Timeout للطلب (10 ثواني)
  const controller = new AbortController();
  const t = setTimeout(() => controller.abort(), 10_000);

  try {
    const resp = await fetch(url, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify(payload),
      signal: controller.signal
    });

    if (!resp.ok) {
      const details = await resp.text();
      const err = new Error(`Upstream error: HTTP ${resp.status}`);
      err.details = details;
      throw err;
    }

    const data = await resp.json();
    const translationObj = data?.data?.translations?.[0];

    const translatedText = translationObj?.translatedText;
    const detectedSourceLanguage = translationObj?.detectedSourceLanguage || null;

    if (!translatedText) {
      const err = new Error('Bad response from upstream');
      err.details = JSON.stringify(data);
      throw err;
    }

    return {
      translatedText: he.decode(translatedText), // فك HTML entities إن وجدت
      detectedSourceLanguage
    };
  } finally {
    clearTimeout(t);
  }
}

// API
app.post('/api/translate', async (req, res) => {
  try {
    const { text, from, to } = req.body || {};

    if (typeof text !== 'string' || !text.trim()) {
      return res.status(400).json({ error: 'TEXT_REQUIRED' });
    }
    if (text.length > 5000) {
      return res.status(400).json({ error: 'TEXT_TOO_LONG' });
    }

    const normalizedFrom = (from && typeof from === 'string') ? from : 'auto';

    if (normalizedFrom !== 'auto' && !isValidLangCode(normalizedFrom)) {
      return res.status(400).json({ error: 'INVALID_SOURCE_LANG' });
    }
    if (!isValidLangCode(to)) {
      return res.status(400).json({ error: 'INVALID_TARGET_LANG' });
    }

    // لو نفس اللغة يدويًا: رجّع النص كما هو
    if (normalizedFrom !== 'auto' && normalizedFrom === to) {
      return res.json({ translatedText: text, detectedSourceLanguage: normalizedFrom });
    }

    const cacheKey = `${normalizedFrom}:${to}:${text}`;
    const cached = cache.get(cacheKey);
    if (cached) return res.json(cached);

    const result = await googleTranslate({ text, from: normalizedFrom, to });
    cache.set(cacheKey, result);

    return res.json(result);
  } catch (e) {
    console.error(e);
    return res.status(502).json({
      error: 'TRANSLATE_FAILED',
      details: e.details || e.message
    });
  }
});

app.get('/health', (req, res) => res.json({ ok: true }));

// Frontend (خدمة واحدة)
app.use(express.static(path.join(__dirname, 'public')));

// SPA fallback (ضعه آخر شيء حتى لا يكسر /api)
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => {
  console.log(`Running on http://localhost:${PORT}`);
});
```

---

## 3) public/index.html
```html
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>مترجم النصوص</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }

    body {
      font-family: 'Segoe UI', Tahoma, sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 20px;
    }

    .container {
      background: white;
      border-radius: 20px;
      box-shadow: 0 20px 60px rgba(0,0,0,0.3);
      max-width: 1000px;
      width: 100%;
      padding: 40px;
    }

    h1 {
      text-align: center;
      color: #764ba2;
      margin-bottom: 10px;
      font-size: 34px;
    }

    .subtitle {
      text-align: center;
      color: #666;
      margin-bottom: 30px;
      font-size: 16px;
    }

    .translate-box {
      display: grid;
      grid-template-columns: 1fr auto 1fr;
      gap: 20px;
      align-items: start;
    }

    .input-section, .output-section {
      display: flex;
      flex-direction: column;
      min-width: 0;
    }

    .lang-selector {
      display: flex;
      align-items: center;
      gap: 10px;
      margin-bottom: 12px;
    }

    select {
      padding: 12px 16px;
      border: 2px solid #ddd;
      border-radius: 10px;
      font-size: 16px;
      background: white;
      cursor: pointer;
      transition: 0.2s;
      width: 100%;
    }

    select:focus { border-color: #764ba2; outline: none; }

    .swap-btn {
      background: #764ba2;
      color: white;
      border: none;
      padding: 12px 14px;
      border-radius: 10px;
      cursor: pointer;
      font-size: 18px;
      transition: 0.2s;
      height: fit-content;
      margin-top: 44px;
    }

    .swap-btn:hover { background: #667eea; transform: rotate(180deg); }

    textarea {
      width: 100%;
      height: 250px;
      padding: 18px;
      border: 2px solid #ddd;
      border-radius: 15px;
      font-size: 16px;
      resize: none;
      transition: 0.2s;
      font-family: inherit;
    }

    textarea:focus { border-color: #764ba2; outline: none; }

    #output {
      background: #f8f9fa;
      color: #333;
    }

    .meta {
      display: flex;
      justify-content: space-between;
      gap: 10px;
      color: #999;
      font-size: 13px;
      margin-top: 8px;
      flex-wrap: wrap;
    }

    .loading {
      text-align: center;
      color: #667eea;
      font-size: 14px;
      margin-top: 10px;
      display: none;
    }

    .loading.active { display: block; }

    @keyframes spin { to { transform: rotate(360deg); } }

    .spinner {
      display: inline-block;
      width: 16px;
      height: 16px;
      border: 3px solid #ddd;
      border-top-color: #667eea;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
      vertical-align: middle;
      margin-left: 8px;
    }

    .buttons {
      display: flex;
      gap: 12px;
      justify-content: center;
      margin-top: 18px;
      flex-wrap: wrap;
    }

    button.action {
      padding: 14px 34px;
      border: none;
      border-radius: 12px;
      font-size: 16px;
      font-weight: 700;
      cursor: pointer;
      transition: 0.2s;
    }

    .translate-btn {
      background: linear-gradient(135deg, #667eea, #764ba2);
      color: white;
    }

    .translate-btn:disabled {
      opacity: 0.7;
      cursor: not-allowed;
    }

    .copy-btn, .clear-btn {
      background: white;
      color: #764ba2;
      border: 2px solid #764ba2;
    }

    .copy-btn:hover, .clear-btn:hover {
      background: #764ba2;
      color: white;
    }

    @media (max-width: 768px) {
      .container { padding: 24px; }
      h1 { font-size: 26px; }
      .translate-box { grid-template-columns: 1fr; }
      .swap-btn { width: 100%; margin-top: 0; transform: rotate(90deg); }
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>مترجم النصوص</h1>
    <p class="subtitle">نسخة احترافية (واجهة + سيرفر + API رسمي)</p>

    <div class="translate-box">
      <div class="input-section">
        <div class="lang-selector">
          <select id="fromLang" aria-label="لغة المصدر">
            <option value="auto">كشف تلقائي</option>
            <option value="ar">العربية</option>
            <option value="en">الإنجليزية</option>
            <option value="fr">الفرنسية</option>
            <option value="es">الإسبانية</option>
            <option value="de">الألمانية</option>
            <option value="it">الإيطالية</option>
            <option value="tr">التركية</option>
            <option value="ru">الروسية</option>
            <option value="zh-CN">الصينية</option>
            <option value="ja">اليابانية</option>
            <option value="ko">الكورية</option>
            <option value="hi">الهندية</option>
            <option value="pt">البرتغالية</option>
            <option value="nl">الهولندية</option>
            <option value="sv">السويدية</option>
          </select>
        </div>

        <textarea id="input" placeholder="اكتب النص هنا للترجمة..." maxlength="5000"></textarea>

        <div class="meta">
          <div id="charCount">0 / 5000</div>
          <div id="detectedLang">اللغة المكتشفة: -</div>
        </div>
      </div>

      <button class="swap-btn" id="swapBtn" type="button" title="تبديل اللغات" aria-label="تبديل اللغات">⇄</button>

      <div class="output-section">
        <div class="lang-selector">
          <select id="toLang" aria-label="لغة الهدف">
            <option value="en">الإنجليزية</option>
            <option value="ar">العربية</option>
            <option value="fr">الفرنسية</option>
            <option value="es">الإسبانية</option>
            <option value="de">الألمانية</option>
            <option value="it">الإيطالية</option>
            <option value="tr">التركية</option>
            <option value="ru">الروسية</option>
            <option value="zh-CN">الصينية</option>
            <option value="ja">اليابانية</option>
            <option value="ko">الكورية</option>
            <option value="hi">الهندية</option>
            <option value="pt">البرتغالية</option>
            <option value="nl">الهولندية</option>
            <option value="sv">السويدية</option>
          </select>
        </div>

        <textarea id="output" placeholder="الترجمة ستظهر هنا..." readonly></textarea>
      </div>
    </div>

    <div class="loading" id="loading">
      جاري الترجمة... <span class="spinner"></span>
    </div>

    <div class="buttons">
      <button class="action translate-btn" id="translateBtn" type="button">ترجم</button>
      <button class="action copy-btn" id="copyBtn" type="button">نسخ</button>
      <button class="action clear-btn" id="clearBtn" type="button">مسح</button>
    </div>
  </div>

  <script>
    const inputText = document.getElementById('input');
    const outputText = document.getElementById('output');
    const fromLang = document.getElementById('fromLang');
    const toLang = document.getElementById('toLang');
    const charCount = document.getElementById('charCount');
    const detectedLangEl = document.getElementById('detectedLang');
    const loading = document.getElementById('loading');

    const translateBtn = document.getElementById('translateBtn');
    const copyBtn = document.getElementById('copyBtn');
    const clearBtn = document.getElementById('clearBtn');
    const swapBtn = document.getElementById('swapBtn');

    let lastDetectedLang = null;

    function debounce(fn, delay = 900) {
      let t;
      return (...args) => {
        clearTimeout(t);
        t = setTimeout(() => fn(...args), delay);
      };
    }

    const autoTranslate = debounce(() => {
      if (inputText.value.trim()) translateText();
    }, 900);

    inputText.addEventListener('input', () => {
      charCount.textContent = `${inputText.value.length} / 5000`;
      autoTranslate();
    });

    translateBtn.addEventListener('click', translateText);
    swapBtn.addEventListener('click', swapLanguages);
    copyBtn.addEventListener('click', copyTranslation);
    clearBtn.addEventListener('click', clearAll);

    inputText.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' && e.ctrlKey) translateText();
    });

