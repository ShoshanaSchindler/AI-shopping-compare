# דו"ח טכני — SHOPICS: מערכת השוואת מחירים חכמה

**מגישות:** מלכה מרילס, מלכה שטרן, שושנה שינדלר

---

## 1. תיאור מפורט של הפרויקט

### 1.1 מהות המערכת

SHOPICS היא מערכת השוואת מחירים שמשקללת **מחיר** מול **מרחק נסיעה** מול **דיוק התאמה**, ומדרגת חנויות לפי ציון כדאיות אמיתי. בניגוד למערכות השוואה קיימות שמציגות רק "איפה הכי זול" — SHOPICS עונה על השאלה "איפה הכי **משתלם**, כשמביאים בחשבון את המרחק".

### 1.2 ארכיטקטורה טכנית

המערכת בנויה בארכיטקטורת client-server:

- **Frontend:** React 18 + Vite (port 5173), 2 עמודים (SearchPage, ScanPage), RTL
- **Backend:** Python 3.13 + FastAPI + Uvicorn (port 8000), async endpoints
- **Database:** PostgreSQL 16 — טבלאות products (36,681), prices (955,205), stores (99)
- **Vector Index:** FAISS IndexFlatIP — 482,594 vectors × 384 dimensions (~750MB RAM)
- **External APIs:** Gemini Vision (OCR), Google Routes (distance), Google Geocoding (one-time)

### 1.3 Pipeline עיבוד — מקלט עד פלט

```
[INPUT] query="חלב 3 אחוז", currentStoreKey="7290696200003-001-071"
  │
  ├─[1] NORMALIZE ─── lowercase, remove [׳״'.,], "ליטר"→"l", "קולה"→"coca cola"
  ├─[2] EMBED ──────── MiniLM model → float32[384], normalized to unit length
  ├─[3] FAISS ──────── IndexFlatIP.search(vec, k=200) → top 200 by inner product
  ├─[4] FILTER ─────── score≥0.45, dedup(item_code,store_key), band=0.06
  ├─[5] PRICE ──────── lookup current_price in my store, remove my store, keep cheaper
  ├─[6] GEO ────────── Haversine(user_lat,user_lng, store_lat,store_lng) ≤ 70km
  ├─[7] SCORE ──────── final = 0.5×match + 0.3×(1-dist/max_dist) + 0.2×(saving/price/0.3)
  └─[8] SORT ──────── final_score DESC → return top results + recommendation string
```

### 1.4 מודל ה-Scoring

```python
match_score    = similarityScore                    # FAISS inner product (0-1)
distance_score = 1 - (aerial_km / max_dist)         # normalized (1=closest, 0=farthest)
price_score    = min((raw_saving / current_price) / 0.3, 1.0)  # normalized, capped

final_score    = match_score * 0.5 + distance_score * 0.3 + price_score * 0.2
```

משקלים: match=50% (מוצר לא נכון = חסר ערך), distance=30% (נוחות), price=20% (תמריץ). נקבעו ע"י כיול ידני על 10 שאילתות. הסיבה שלא השתמשנו ב-ML: אין click data ממשתמשים אמיתיים לאימון.

### 1.5 נתונים

- **מקור:** קבצי XML מאתר שקיפות מחירים (חובת דיווח לפי חוק, משרד הכלכלה)
- **רשתות:** רמי לוי, שופרסל, ויקטורי, טיב טעם, מחסני השוק, וולט מרקט, ברקת
- **היקף:** 422 קבצי XML → 36,681 מוצרים ייחודיים, 955,205 רשומות מחיר, 99 חנויות עם GPS
- **Embedding model:** `paraphrase-multilingual-MiniLM-L12-v2` (Hugging Face, 384-dim, multilingual)
- **Index type:** FAISS IndexFlatIP (exact inner product search, no approximation)

### 1.6 מדדי ביצוע

| מדד | ערך |
|------|------|
| זמן חיפוש (embed + FAISS + filter + score) | ~300ms |
| זמן OCR (Gemini Vision round-trip) | ~2 שניות |
| זמן בניית אינדקס (482K vectors) | ~45 דקות |
| RAM usage (FAISS + mapping) | ~750MB |
| דיוק חיפוש (Core Score — רלוונטיות) | ~85% |
| דיוק OCR (שם מוצר) | ~90% |

---

## 2. תיאור מפורט של השימוש בכלי AI

### 2.1 AI בתוך המוצר (Production)

#### sentence-transformers — Embedding Model

- **מודל:** `paraphrase-multilingual-MiniLM-L12-v2`
- **מטרה:** המרת שם מוצר (טקסט חופשי בעברית/אנגלית) לוקטור 384-dim שמייצג משמעות סמנטית
- **שימוש:** כל חיפוש → embedding אחד (~50ms). בניית אינדקס → 482K embeddings (batch, ~45 דקות)
- **למה מודל זה:** multilingual (עברית+אנגלית), קל (90MB, CPU only), מהיר, חינמי
- **אלטרנטיבות שנבדקו:** intfloat/multilingual-e5-small (דורש prefix protocol, מעט איטי יותר), all-MiniLM-L6-v2 (אנגלית בלבד — לא מתאים)

#### FAISS — Vector Similarity Search

- **ספרייה:** faiss-cpu 1.8 (Meta/Facebook AI Research)
- **מטרה:** מציאת top-K וקטורים דומים ביותר מתוך 482K — ב-100ms
- **Index type:** IndexFlatIP (Flat = exact, IP = Inner Product = cosine for normalized vecs)
- **למה לא ANN (approximate):** 482K vectors זה קטן מספיק ל-exact search. ANN (IVF, HNSW) שווה רק מעל ~10M vectors
- **תחליפים שנבדקו:** numpy dot product (fallback — 500ms, איטי ×5), PostgreSQL pgvector (דורש extension)

#### Gemini Vision — זיהוי מוצר מתמונה

- **מודל:** gemini-3-flash-preview (Google Generative AI API)
- **מטרה:** המשתמש מצלם מוצר → מקבל שם+מותג+כמות כטקסט → משמש לחיפוש סמנטי
- **Prompt:** ראה סעיף 3 (רשימת Prompts)
- **עלות:** ~$0.001 לקריאה
- **אלטרנטיבות שנכשלו:** TensorFlow MobileNetV2 (לא קורא טקסט), Tesseract OCR (עברית גרועה), Google Cloud Vision (dump ללא הבנה)

### 2.2 AI בתהליך הפיתוח

#### Kiro (AI IDE)

- **שימוש עיקרי:** כתיבת קוד, debug, הוספת הערות, בניית Test Set, תכנון specs
- **חוזק:** מבין הקשר של כל הפרויקט, כותב קוד שלם ומתועד
- **חולשה:** מוגבל בזמן (context window), עולה כסף

#### Claude (Anthropic)

- **שימוש עיקרי:** תכנון ארכיטקטורה, קבלת מידע אמין, פתרון בעיות עקרוניות
- **חוזק:** המידע הכי אמין מבין כלי ה-AI, הבנה עמוקה של trade-offs
- **חולשה:** תשובות באנגלית, לא מחובר ל-IDE

#### ChatGPT (OpenAI)

- **שימוש עיקרי:** מידע מעשי (כתובות חנויות, API docs), הפעלת Google/Gemini APIs
- **חוזק:** עולם הרשת המעשי, פרקטי
- **חולשה:** hallucinations, עלות (Plus)

#### Gemini (Google AI Studio)

- **שימוש עיקרי:** פתרון בעיות ספציפיות, בעיקר הקשורות ל-Google APIs
- **חוזק:** מכיר את ה-APIs של Google
- **שיטה:** כתבנו prompts אליו באמצעות Kiro — ככה השאלות היו מנוסחות מדויק

#### Amazon Q

- **שימוש עיקרי:** ניסיון אחרון כשנתקענו
- **תוצאה:** לא עזר ברוב המקרים. עזר קצת בסידור קוד.

---

## 3. רשימת Prompts מרכזיים

### Prompt 1: זיהוי מוצר מתמונה (Gemini Vision)

```
אתה מערכת לזיהוי מוצרים מתמונות. 
תסתכל על התמונה ותזהה את המוצר. 
החזר **רק** את הפרטים הבאים בפורמט הזה בדיוק (בלי שום טקסט נוסף):

שם: [שם המוצר בעברית]
מותג: [שם המותג]
כמות: [כמות + יחידה, למשל 500 מ"ל או 1 ק"ג]

אם אתה לא בטוח בפרט מסוים, כתוב "לא ידוע".
```

**למה עובד:** פורמט קבוע → parsing אוטומטי. "לא ידוע" → נסנן בקוד. 3 שדות בלבד → מונע dump.

### Prompt 2: E5 Prefix Protocol (Embedding)

```python
# לשאילתות (query time):
"query: " + normalized_text

# למסמכים (index time):
"passage: " + normalized_text
```

**למה:** מודלי E5 אומנו עם prefix — שאילתות מקבלות "query:" ומסמכים "passage:". בלי prefix — הדיוק יורד ~10%.

### Prompt 3: נרמול טקסט (Pre-processing)

```python
# Unit normalization:
"ליטר" | "l" | "liter" | "litre" → "l"
"מ״ל" | "ml" | "מיליליטר" → "ml"
"ק״ג" | "kg" | "קילוגרם" → "kg"

# Brand synonyms:
"קוקה קולה" | "קולה" → "coca cola"
"תנובה" → "tnuva"
"אסם" → "osem"

# Punctuation removal:
[׳״'\".,\-_/\\|] → " "
```

**למה:** ללא נרמול, "חלב 1.5 ליטר" ו-"חלב 1.5L" מקבלים similarity 0.75 במקום 0.95.

### Prompt 4: Confidence Tiers

```python
if best_score >= 0.82: confidence = "high"    # "נמצאה התאמה טובה"
if best_score >= 0.65: confidence = "medium"  # "נמצאו מספר אפשרויות"
if best_score >= 0.45: confidence = "low"     # "התאמות חלשות"
if best_score <  0.45: → filtered out         # לא מוצג למשתמש
```

### Prompt 5: Band Selection

```python
BAND_TIGHT = 0.03  # לבחירת item_code ראשי (prefix match priority)
BAND_SCAN  = 0.06  # לאיסוף item_codes דומים (וריאציות)

# אוסף כל item_code עם score בטווח 0.06 מה-best
# → מכסה: "חלב 3% 1L", "חלב 3% 0.5L", "חלב 3% שקית"
```

---

## 4. מקורות

### נתונים
- אתר שקיפות מחירים — משרד הכלכלה והתעשייה: https://prices.shufersal.co.il (ודומיו לכל רשת)
- חוק שקיפות מחירים (2014) — חובת פרסום XML יומי

### מודלים וספריות
- sentence-transformers: https://www.sbert.net/ — Reimers & Gurevych, 2019
- paraphrase-multilingual-MiniLM-L12-v2: https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2
- FAISS: https://github.com/facebookresearch/faiss — Johnson, Douze, Jégou (Meta AI, 2019)
- Gemini Vision API: https://ai.google.dev/docs — Google DeepMind

### Frameworks
- FastAPI: https://fastapi.tiangolo.com/ — Sebastián Ramírez
- React: https://react.dev/ — Meta
- Vite: https://vitejs.dev/ — Evan You
- PostgreSQL: https://www.postgresql.org/
- httpx: https://www.python-httpx.org/

### APIs
- Google Routes API: https://developers.google.com/maps/documentation/routes
- Google Geocoding API: https://developers.google.com/maps/documentation/geocoding
- Google Generative AI (Gemini): https://ai.google.dev/

### כלי פיתוח AI
- Kiro (AWS): https://kiro.dev/
- Claude (Anthropic): https://claude.ai/
- ChatGPT (OpenAI): https://chat.openai.com/
- Gemini (Google): https://gemini.google.com/
- Amazon Q: https://aws.amazon.com/q/

### מושגים תיאורטיים
- Cosine Similarity & Inner Product Search — Manning, Raghavan & Schütze, "Introduction to Information Retrieval", 2008
- Haversine Formula — https://en.wikipedia.org/wiki/Haversine_formula
- Approximate Nearest Neighbors — Malkov & Yashunin, "Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs", 2018
