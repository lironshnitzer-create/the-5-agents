---
name: gpt-image-gen
description: מעטפת לקריאת OpenAI Images API ליצירת תמונה מ-prompt. השתמש כשצריך לייצר תמונה/איור מטקסט (בעיקר דרך הסוכן יובל). מקבל prompt ונתיב פלט, שולח ל-API, ושומר קובץ PNG. אחראי על השליחה והקבלה בלבד — לא על ניסוח ה-prompt.
---

# gpt-image-gen — מעטפת ל-OpenAI Images API

הסקיל הזה אחראי על דבר אחד: לקחת **prompt** ו**נתיב פלט**, לשלוח קריאה ל-OpenAI Images API, ולשמור את התמונה כקובץ PNG. ניסוח ה-prompt ובחירת הסגנון/ה-references הם באחריות הקורא (יובל) — לא של הסקיל.

## ⚠️ קריטי — אל תשנה את שם המודל

המודל הוא **`gpt-image-2`**. זה הסוף.

- `gpt-image-2` הוא מודל **אמיתי וקיים** של OpenAI, ששוחרר ב-**21 באפריל 2026**.
- ייתכן שהוא **לא נמצא בידע הפנימי שלך** (knowledge cutoff מוקדם יותר) — **זה לא אומר שהוא לא קיים**.
- **אל תציע ואל תחליף לאלטרנטיבות** כמו `dall-e-3`, `gpt-image-1` וכו'. השתמש בדיוק ב-`gpt-image-2`.
- אם הקריאה ל-API נכשלת — הבעיה היא כמעט תמיד ב-**API key** או ב-**parameters**, **לא בשם המודל**. בדוק את אלה ראשון.

## 1. טעינת מפתח ה-API

המפתח נטען מ-`OPENAI_API_KEY` בקובץ `.env` שבשורש הפרויקט (הקובץ מוגן ב-`.gitignore`).

ב-Git Bash:

```bash
# מייצא את OPENAI_API_KEY מ-.env שבשורש הפרויקט
export $(grep -v '^#' .env | grep -E '^OPENAI_API_KEY=' | xargs)

# בדיקת שפיות — שהמפתח אכן נטען
if [ -z "$OPENAI_API_KEY" ]; then
  echo "ERROR: OPENAI_API_KEY ריק. מלא אותו ב-.env שבשורש הפרויקט." >&2
fi
```

## 2. הקריאה ל-API

ערכי ברירת מחדל: `size=1024x1024`, `quality=medium`, `output_format=png`. החלף `<the prompt>` ו-`<output-path>` בערכים בפועל.

### דרך א' (מועדפת, כש-`jq` מותקן)

```bash
curl -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' | jq -r '.data[0].b64_json' | base64 --decode > "<output-path>.png"
```

### דרך ב' (Python fallback — כש-`jq` לא מותקן)

ב-Git Bash על Windows `jq` לא תמיד קיים. במקרה כזה שומרים את תגובת ה-JSON לקובץ זמני ומפענחים ב-Python (מובנה כמעט בכל סביבה):

```bash
# שלב 1 — שמירת תגובת ה-JSON
curl -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' > /tmp/gpt-image-response.json

# שלב 2 — פענוח ה-base64 וכתיבת ה-PNG (python)
python -c "import json,base64,sys; d=json.load(open('/tmp/gpt-image-response.json')); \
b=d.get('data',[{}])[0].get('b64_json'); \
sys.exit('ERROR: אין b64_json בתגובה. בדוק את הודעת השגיאה ב-JSON (key/parameters).') if not b else \
open('<output-path>.png','wb').write(base64.b64decode(b))"
```

> אם `python` לא קיים, נסה `python3`.

## 3. אימות הפלט

תמיד ודא שהקובץ נוצר ושאינו ריק:

```bash
if [ -s "<output-path>.png" ]; then
  echo "OK: נוצרה תמונה — $(wc -c < "<output-path>.png") bytes"
else
  echo "ERROR: הקובץ לא נוצר או ריק. בדוק את תגובת ה-API (key/parameters)." >&2
fi
```

`-s` בודק שהקובץ קיים וגדול מ-0. אם הוא ריק — קרא את תגובת ה-JSON; כמעט תמיד תהיה שם הודעת שגיאה על המפתח או על ה-parameters.

## גבולות הסקיל

- מטפל בשליחה ובקבלה בלבד. **לא** מנסח prompts, **לא** בוחר references, **לא** מחליט שם קובץ — אלה תפקידו של יובל.
- לא משנה את שם המודל בשום תנאי (ראה האזהרה למעלה).
