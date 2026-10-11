# SHOPICS — Pic It. We'll Pick Smarter. 🛒

מערכת AI חכמה להשוואת מחירים — משקללת **מחיר** מול **מרחק נסיעה**.
מחפשים או מצלמים מוצר → המערכת מזהה → משווה מחירים ב-7 רשתות → מדרגת חנויות לפי כדאיות אמיתית.

---

## טכנולוגיות

| שכבה | טכנולוגיה |
|------|-----------|
| Backend | Python 3.13, FastAPI, Uvicorn |
| Database | PostgreSQL (מוצרים, מחירים, חנויות) |
| Embedding | sentence-transformers (paraphrase-multilingual-MiniLM-L12-v2) |
| Vector Search | FAISS IndexFlatIP (482K vectors, 384 dim) |
| OCR | Google Gemini Vision API (gemini-3-flash) |
| Distance | Google Routes API (TRAFFIC_AWARE) |
| Geocoding | Google Geocoding API (חד-פעמי) |
| Frontend | React 18, Vite, React Router v6 |

---

## מבנה תיקיות

```
shopics/
├── backend/
│   ├── app/
│   │   ├── db/              # pg.py — חיבור PostgreSQL + שליפת נתונים
│   │   ├── search/          # embeddings, faiss_index, normalizer, searcher
│   │   ├── recommend/       # scoring, smart_recommend, routes_service
│   │   ├── vision/          # ocr.py — Gemini Vision API
│   │   └── routes/          # search.py, vision.py, recommend.py
│   ├── config.py            # הגדרות סביבה
│   ├── main.py              # FastAPI server + FAISS loading
│   ├── scripts/             # build_index.py, geocode_stores.py
│   ├── scoring/             # scoring.ts + scoring.test.ts (TypeScript validation)
│   ├── tests/test_set/      # Test Set (60 structural + 100 smart validation)
│   ├── models/              # FAISS index + mapping JSON
│   ├── requirements.txt
│   └── .env
├── frontend/
│   ├── src/
│   │   ├── pages/           # SearchPage.jsx, ScanPage.jsx
│   │   ├── services/        # api.js — קריאות API
│   │   ├── styles/          # global.css
│   │   ├── App.jsx          # ניתוב + nav
│   │   └── main.jsx         # entry point
│   ├── index.html
│   ├── package.json
│   └── vite.config.js
├── data/extracted_xml/      # 422 קבצי XML (שקיפות מחירים)
├── ARCHITECTURE.md          # תיעוד ארכיטקטורה מפורט
├── presentation.html        # מצגת HTML (29 שקפים)
└── README.md
```

---

## התקנה והרצה

### דרישות מקדימות

- Python 3.13+
- Node.js 18+
- PostgreSQL 16 פעיל (עם DB `shopics`)

---

### Backend

```bash
cd backend

# 1. צור סביבה וירטואלית
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # macOS/Linux

# 2. התקן תלויות
pip install -r requirements.txt

# 3. הגדר משתני סביבה
copy .env.example .env
# ערוך .env: PG_DB, PG_USER, PG_PASSWORD, GEMINI_API_KEY, GOOGLE_ROUTES_API_KEY

# 4. הרץ את השרת
python main.py
```

ה-API זמין בכתובת: http://localhost:8000
Swagger docs: http://localhost:8000/docs

---

### Frontend

```bash
cd frontend

# 1. התקן תלויות
npm install

# 2. הרץ בפיתוח
npm run dev
```

הפרונטאנד זמין בכתובת: http://localhost:5173
(Proxy אוטומטי ל-backend דרך Vite)

---

## בניית FAISS Index

```bash
cd backend

# כל השורות (לוקח ~45 דקות):
python scripts/build_index.py

# הרצת בדיקה מהירה (5000 שורות):
python scripts/build_index.py --limit 5000
```

---

## Geocoding חנויות (חד-פעמי)

```bash
cd backend
python scripts/geocode_stores.py
```

דורש `GOOGLE_ROUTES_API_KEY` ב-.env. מוסיף lat/lng לכל 99 חנויות.

---

## הרצת בדיקות

```bash
cd backend

# 1. בדיקות structural (60 test cases):
python tests/test_set/run_tests.py

# 2. Smart Validation (100 מוצרים, DB comparison):
python tests/test_set/run_smart_validation.py

# 3. TypeScript scoring tests:
cd scoring && npx jest
```

תוצאות נשמרות ב-`tests/test_set/results/`.

---

## נתוני המערכת

| מדד | ערך |
|------|------|
| מוצרים ייחודיים | 36,681 |
| רשומות מחיר | 980,700 |
| חנויות (עם GPS) | 99 |
| רשתות | 7 (רמי לוי, שופרסל, ויקטורי, טיב טעם, מחסני השוק, וולט מרקט, ברקת) |
| FAISS vectors | 482,594 |
| Embedding dimensions | 384 |
| זמן חיפוש | ~300ms |

---

## API Endpoints

| Method | Path | תיאור |
|--------|------|-------|
| POST | /api/ai/search-products | חיפוש סמנטי לפי טקסט |
| POST | /api/ai/recommend | המלצת חנות (מחיר + מרחק + התאמה) |
| POST | /api/ai/ocr | זיהוי מוצר מתמונה (Gemini Vision) |
| POST | /api/ai/ocr-search | זיהוי + חיפוש בקריאה אחת |
| POST | /api/ai/recognize-product | זיהוי (תואם frontend ישן) |
| GET | /api/ai/stores | רשימת חנויות (לבחירת חנות נוכחית) |
| POST | /api/ai/rebuild-index | בניית FAISS index מחדש |

---

## נוסחת הדירוג

```
match_score    = FAISS similarity (0-1)
distance_score = 1 - (aerial_km / 70)
price_score    = (saving / current_price) / 0.3, capped at 1

final_score    = 0.5 × match + 0.3 × distance + 0.2 × price
```

---

## משתני סביבה (.env)

```env
# Database
PG_DB=shopics
PG_USER=postgres
PG_PASSWORD=your_password
PG_HOST=localhost
PG_PORT=5432

# AI
FAISS_INDEX_PATH=models/product_index.faiss
EMBEDDING_MODEL=paraphrase-multilingual-MiniLM-L12-v2
USE_MOCK_AI=false

# APIs
GEMINI_API_KEY=your_gemini_key
GOOGLE_ROUTES_API_KEY=your_routes_key

# Server
APP_HOST=0.0.0.0
APP_PORT=8000
DEBUG=true
```

---

## מצב Mock

כאשר `USE_MOCK_AI=true` ב-.env:
- Embedding מחזיר וקטורים אקראיים (דטרמיניסטיים)
- לא דורש GPU או מודל sentence-transformers
- מתאים לפיתוח frontend בלבד

---

## מקור הנתונים

קבצי XML מאתר **שקיפות המחירים** של משרד הכלכלה (חובה חוקית על רשתות).
422 קבצים (gzipped) עם מחירי כל המוצרים בכל הסניפים.
