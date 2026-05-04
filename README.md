# CyberSafe Class Analyzer — Premium Streamlit Design
# Run locally:
#   pip install streamlit python-docx qrcode pillow
#   streamlit run cyber_safe_analyzer_python_streamlit.py

import base64
import io
import re
from collections import Counter

import qrcode
import streamlit as st
from docx import Document
from docx.shared import Pt
from PIL import Image

st.set_page_config(
    page_title="CyberSafe Class Analyzer",
    page_icon="🛡️",
    layout="wide",
    initial_sidebar_state="collapsed",
)

APP_URL = "https://cybersafe-analyzer-evpymhc7fvnuyubc9wrqta.streamlit.app/#cyber-safe-class-analyzer"

RISK_LEXICON = {
    "high": [
        "убью", "өлтірем", "побью", "изобью", "зарежу", "сдохни", "умри", "угрожаю",
        "на три буквы", "посылает", "иди на", "шантаж", "суицид", "повешусь"
    ],
    "medium": [
        "тупая", "тупой", "дура", "дурак", "идиот", "лох", "заткнись", "не спрашивала",
        "не у тебя", "зачем ты", "ты сказала", "не брала", "как она у тебя", "гонишь",
        "врала", "обман", "неге алдың", "алдың"
    ],
    "low": [
        "ахах", "хаха", "😂", "🤣", "ммм", "ого", "копеец", "емма", "ема", "🤦", "лол"
    ],
    "self_defense": [
        "я не", "мен емес", "не брала", "я знаю не", "не тупая", "мен алмадым",
        "не виновата", "я не могла", "я не делала"
    ],
    "system": [
        "сообщения и звонки защищены", "добавил(-а)", "создал(-а) группу", "изменил(-а)",
        "закрепил(-а)", "теперь вы админ", "исчезающие сообщения", "удалил", "удалила",
        "присоединился", "настройки группы", "вы удалили", "данное сообщение удалено",
        "изображение отсутствует", "аудиофайл отсутствует", "видео отсутствует",
        "видеозаметка опущена", "<сообщение изменено>"
    ]
}

CHAT_PATTERNS = [
    re.compile(r"^\[(\d{1,2}\.\d{1,2}\.\d{4}),\s*(\d{1,2}:\d{2}(?::\d{2})?)\]\s*([^:]+):\s*(.*)$"),
    re.compile(r"^(\d{1,2}\.\d{1,2}\.\d{4}),\s*(\d{1,2}:\d{2}(?::\d{2})?)\s*-\s*([^:]+):\s*(.*)$"),
    re.compile(r"^(\d{1,2}/\d{1,2}/\d{2,4}),\s*(\d{1,2}:\d{2}(?::\d{2})?\s*(?:AM|PM|am|pm)?)\s*-\s*([^:]+):\s*(.*)$"),
    re.compile(r"^\[(\d{1,2}/\d{1,2}/\d{2,4}),\s*(\d{1,2}:\d{2}(?::\d{2})?\s*(?:AM|PM|am|pm)?)\]\s*([^:]+):\s*(.*)$"),
]


def inject_css():
    st.markdown(
        """
        <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&display=swap');

        html, body, [class*="css"] { font-family: 'Inter', sans-serif; }

        .stApp {
            background:
                radial-gradient(circle at 20% 10%, rgba(0, 255, 179, .18), transparent 30%),
                radial-gradient(circle at 80% 20%, rgba(30, 144, 255, .20), transparent 28%),
                linear-gradient(135deg, #06142b 0%, #072345 45%, #063b2b 100%);
            color: #ffffff;
        }

        .block-container { padding-top: 1.5rem; padding-bottom: 2rem; }

        .hero {
            position: relative;
            overflow: hidden;
            border: 1px solid rgba(99, 255, 202, .35);
            border-radius: 34px;
            padding: 34px;
            background:
                linear-gradient(135deg, rgba(5, 22, 50, .92), rgba(6, 67, 54, .82)),
                repeating-linear-gradient(60deg, rgba(255,255,255,.05) 0px, rgba(255,255,255,.05) 1px, transparent 1px, transparent 18px);
            box-shadow: 0 24px 70px rgba(0,0,0,.35);
        }

        .hero::before {
            content: "";
            position: absolute;
            inset: 0;
            background-image:
                linear-gradient(rgba(76, 255, 196, .08) 1px, transparent 1px),
                linear-gradient(90deg, rgba(76, 255, 196, .08) 1px, transparent 1px);
            background-size: 42px 42px;
            mask-image: linear-gradient(to bottom, black, transparent);
            pointer-events: none;
        }

        .hero-content { position: relative; z-index: 2; }
        .badge {
            display: inline-flex;
            align-items: center;
            gap: 10px;
            padding: 9px 14px;
            border-radius: 999px;
            background: rgba(75, 255, 190, .13);
            border: 1px solid rgba(75, 255, 190, .35);
            color: #9dffd5;
            font-weight: 800;
            letter-spacing: .02em;
        }

        .hero-title {
            margin-top: 22px;
            font-size: clamp(40px, 6vw, 78px);
            line-height: .95;
            font-weight: 900;
            letter-spacing: -0.05em;
            color: #ffffff;
        }

        .green { color: #73f35b; }
        .cyan { color: #63fff0; }

        .hero-subtitle {
            margin-top: 20px;
            max-width: 850px;
            font-size: 21px;
            line-height: 1.55;
            color: rgba(255,255,255,.82);
        }

        .robot-card {
            border-radius: 28px;
            padding: 24px;
            background: linear-gradient(160deg, rgba(255,255,255,.13), rgba(255,255,255,.05));
            border: 1px solid rgba(255,255,255,.18);
            box-shadow: inset 0 1px 0 rgba(255,255,255,.12);
            text-align: center;
        }

        .robot-head {
            width: 210px;
            height: 145px;
            margin: 0 auto 16px;
            border-radius: 48% 48% 40% 40%;
            background: linear-gradient(145deg, #22b7ff, #0058cb 55%, #003d93);
            position: relative;
            box-shadow: 0 20px 55px rgba(0, 176, 255, .32), inset 0 4px 16px rgba(255,255,255,.55);
        }

        .robot-head::before {
            content: "";
            position: absolute;
            top: 36px;
            left: 28px;
            right: 28px;
            height: 70px;
            border-radius: 30px;
            background: #061222;
            box-shadow: inset 0 0 20px rgba(0,0,0,.8);
        }

        .eye {
            position: absolute;
            top: 57px;
            width: 28px;
            height: 38px;
            border-radius: 999px;
            background: #ffd72f;
            box-shadow: 0 0 18px rgba(255, 231, 70, .9);
            z-index: 3;
        }
        .eye.left { left: 70px; }
        .eye.right { right: 70px; }

        .robot-antenna {
            position: absolute;
            top: -36px;
            left: 50%;
            width: 8px;
            height: 38px;
            background: #071a2e;
            transform: translateX(-50%);
        }
        .robot-antenna::before {
            content: "";
            position: absolute;
            top: -18px;
            left: 50%;
            width: 28px;
            height: 28px;
            border-radius: 50%;
            background: #ffcd1d;
            transform: translateX(-50%);
            box-shadow: 0 0 18px rgba(255, 205, 29, .8);
        }

        .qr-wrap {
            display: inline-block;
            padding: 12px;
            border-radius: 24px;
            background: white;
            border: 6px solid #63fff0;
            box-shadow: 0 0 28px rgba(99,255,240,.55);
        }

        .glass-card {
            min-height: 160px;
            border-radius: 26px;
            padding: 22px;
            background: rgba(255,255,255,.10);
            border: 1px solid rgba(255,255,255,.16);
            box-shadow: 0 18px 50px rgba(0,0,0,.18);
            backdrop-filter: blur(10px);
        }

        .glass-card h3 {
            margin: 0 0 10px 0;
            font-size: 22px;
            font-weight: 900;
            color: white;
        }

        .glass-card p, .glass-card li {
            color: rgba(255,255,255,.82);
            font-size: 16px;
            line-height: 1.55;
        }

        .section-title {
            margin: 28px 0 14px;
            font-size: 30px;
            font-weight: 900;
            color: #ffffff;
        }

        .metric-box {
            border-radius: 24px;
            padding: 22px;
            background: linear-gradient(135deg, rgba(255,255,255,.16), rgba(255,255,255,.07));
            border: 1px solid rgba(255,255,255,.16);
            text-align: center;
        }

        .metric-box .num {
            font-size: 38px;
            font-weight: 900;
            color: #73f35b;
        }

        .metric-box .label {
            color: rgba(255,255,255,.75);
            font-weight: 700;
        }

        .stButton>button, .stDownloadButton>button {
            border-radius: 18px !important;
            border: 1px solid rgba(99,255,240,.35) !important;
            background: linear-gradient(135deg, #12a85a, #007cf0) !important;
            color: white !important;
            font-weight: 900 !important;
            box-shadow: 0 14px 34px rgba(0, 124, 240, .25) !important;
            min-height: 52px;
        }

        textarea, .stTextArea textarea {
            border-radius: 20px !important;
            border: 1px solid rgba(99,255,240,.25) !important;
            background: rgba(255,255,255,.94) !important;
        }

        .stFileUploader {
            border-radius: 26px;
            padding: 18px;
            background: rgba(255,255,255,.10);
            border: 1px dashed rgba(99,255,240,.45);
        }

        div[data-testid="stMetric"] {
            background: rgba(255,255,255,.10);
            border: 1px solid rgba(255,255,255,.14);
            padding: 16px;
            border-radius: 22px;
        }

        div[data-testid="stMetricLabel"], div[data-testid="stMetricValue"] { color: white !important; }

        .footer-banner {
            margin-top: 28px;
            padding: 22px 28px;
            border-radius: 26px;
            background: linear-gradient(90deg, #063b86, #0d8b57);
            border: 1px solid rgba(255,255,255,.18);
            text-align: center;
            font-size: 24px;
            font-weight: 900;
            color: white;
        }
        </style>
        """,
        unsafe_allow_html=True,
    )


def make_qr(data: str) -> str:
    qr = qrcode.QRCode(error_correction=qrcode.constants.ERROR_CORRECT_H, box_size=8, border=2)
    qr.add_data(data)
    qr.make(fit=True)
    img = qr.make_image(fill_color="#06142b", back_color="white").convert("RGB")
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    return base64.b64encode(buffer.getvalue()).decode("utf-8")


def normalize_name(name: str) -> str:
    name = name.replace("~", "").replace("🍫", "")
    name = re.sub(r"5\s?б", "", name, flags=re.IGNORECASE)
    name = re.sub(r"\s+", " ", name)
    return name.strip()


def parse_chat(text: str):
    messages = []
    unmatched = []
    for line in text.splitlines():
        clean = line.replace("\u200e", "").replace("\u200f", "").strip()
        if not clean:
            continue
        match = None
        for pattern in CHAT_PATTERNS:
            match = pattern.match(clean)
            if match:
                break
        if match:
            date, time, sender, message = match.groups()
            messages.append({
                "date": date.replace("/", "."),
                "time": time.strip(),
                "sender": normalize_name(sender),
                "message": (message or "").strip(),
                "raw": clean,
            })
        elif messages:
            messages[-1]["message"] += "\n" + clean
        else:
            unmatched.append(clean)
    return messages, unmatched


def contains_any(text: str, words: list[str]):
    lower = text.lower()
    return [word for word in words if word.lower() in lower]


def is_system_or_media(message: str) -> bool:
    lower = message.lower()
    return any(word.lower() in lower for word in RISK_LEXICON["system"])


def classify_message(message: str):
    high = contains_any(message, RISK_LEXICON["high"])
    medium = contains_any(message, RISK_LEXICON["medium"])
    low = contains_any(message, RISK_LEXICON["low"])
    self_defense = contains_any(message, RISK_LEXICON["self_defense"])

    if high:
        return "High", "Қорлау / қауіп / вербалды агрессия", "Жеке сөйлесу, ата-анамен байланыс, қайталанса әкімшілік шара", high
    if medium:
        return "Medium", "Агрессивті коммуникация / қысым", "Чатта тоқтату, жеке түсіндіру, конфликтті офлайн шешу", medium
    if self_defense:
        return "Medium", "Өзін қорғау реакциясы", "Оқушымен жеке сөйлесу, эмоционалдық қолдау көрсету", self_defense
    if low:
        return "Low", "Күлкі / мазақ болуы мүмкін", "Контекстті бақылау, мазаққа айналса ескерту", low
    return "None", "Қалыпты", "Бақылау қажет емес", []


def filter_keywords(messages, keywords):
    return [m for m in messages if any(k.lower() in m["message"].lower() for k in keywords)]


def unique_senders(items):
    return ", ".join(sorted(set(item["sender"] for item in items if item.get("sender"))))


def quote_items(items, limit=5):
    quotes = []
    for item in items[:limit]:
        text = re.sub(r"\s+", " ", item["message"])[:160]
        quotes.append(f"“{text}”")
    return quotes


def identify_situations(messages):
    situations = []

    correction = filter_keywords(messages, ["замаз", "не брала", "зачем ты", "как она у тебя", "ты сказала"])
    if len(correction) >= 2:
        situations.append({
            "title": "Замазка оқиғасы",
            "participants": unique_senders(correction),
            "summary": "Жоғалған затқа байланысты бір оқушыға бірнеше сұрақ қойылып, айыптау сипаты байқалған.",
            "evaluation": "Толық буллинг емес, бірақ микро-буллинг және топтық қысым белгісі бар.",
            "quotes": quote_items(correction),
        })

    admin = filter_keywords(messages, ["админ", "староста", "зам", "слежу", "поряд"])
    if len(admin) >= 3:
        situations.append({
            "title": "Админ/староста рөліне байланысты жағдай",
            "participants": unique_senders(admin),
            "summary": "Оқушылар арасында чаттағы басқару рөлі, админ болу және жауапкершілікке байланысты пікірталас болған.",
            "evaluation": "Буллинг емес, бірақ лидерлік пен билікке таласу арқылы конфликт тудыруы мүмкін.",
            "quotes": quote_items(admin),
        })

    aggression = filter_keywords(messages, ["посылает", "на три буквы", "иди на", "тупая", "тупой", "дура", "дурак"])
    if len(aggression) >= 1:
        situations.append({
            "title": "Вербалды агрессия белгілері",
            "participants": unique_senders(aggression),
            "summary": "Чатта немесе чат арқылы сыныптағы дөрекі сөйлеу, боқтау туралы хабарламалар кездеседі.",
            "evaluation": "Бұл ең маңызды қауіп факторы. Қайталанса буллингке айналуы мүмкін.",
            "quotes": quote_items(aggression),
        })

    return situations


def analyze_chat(raw_text: str):
    parsed, unmatched = parse_chat(raw_text)
    text_messages = [m for m in parsed if m["message"] and not is_system_or_media(m["message"])]

    enriched = []
    for msg in text_messages:
        risk, category, recommendation, keywords = classify_message(msg["message"])
        item = dict(msg)
        item.update({"risk": risk, "category": category, "recommendation": recommendation, "keywords": keywords})
        enriched.append(item)

    participants = sorted(set(m["sender"] for m in text_messages if m["sender"]))
    risky = [m for m in enriched if m["risk"] != "None"]
    high = [m for m in risky if m["risk"] == "High"]
    medium = [m for m in risky if m["risk"] == "Medium"]
    low = [m for m in risky if m["risk"] == "Low"]

    sender_counts = Counter(m["sender"] for m in enriched)
    active_threshold = max(3, int(len(text_messages) * 0.03))
    active = [(name, count) for name, count in sender_counts.most_common() if count >= active_threshold]

    score = len(high) * 3 + len(medium) * 2 + len(low)
    risk_level = "🟢 Төмен"
    if len(high) > 0 or len(medium) >= 5 or score >= 12:
        risk_level = "🟡 Орта"
    if len(high) >= 3 or len(medium) >= 15 or score >= 40:
        risk_level = "🔴 Жоғары"

    return {
        "parsed": parsed,
        "unmatched": unmatched,
        "text_messages": text_messages,
        "enriched": enriched,
        "participants": participants,
        "risky": risky,
        "high": high,
        "medium": medium,
        "low": low,
        "active": active,
        "situations": identify_situations(enriched),
        "risk_level": risk_level,
    }


def quote_line(items):
    if not items:
        return "- Нақты мәтіндік хабарлама анықталмады."
    lines = []
    for item in items[:6]:
        text = re.sub(r"\s+", " ", item["message"])[:160]
        lines.append(f"- “{text}” ({item['sender']}, {item['date']} {item['time']})")
    return "\n".join(lines)


def generate_report(a):
    situation_text = "Нақты конфликт ситуациялары анықталмады."
    if a["situations"]:
        blocks = []
        for i, s in enumerate(a["situations"], 1):
            quotes = "\n".join(f"- {q}" for q in s["quotes"])
            blocks.append(
                f"{i}) {s['title']}\n"
                f"Қатысушылар: {s['participants'] or 'нақты анықталмады'}.\n"
                f"Не болды: {s['summary']}\n"
                f"Бағалау: {s['evaluation']}\n"
                f"Мысалдар:\n{quotes}"
            )
        situation_text = "\n\n".join(blocks)

    active_text = ", ".join(f"{name} ({count})" for name, count in a["active"]) or "анықталмады"

    return f"""5 «Б» СЫНЫБЫНЫҢ WHATSAPP ЧАТЫ БОЙЫНША ПСИХОЛОГИЯЛЫҚ ЕСЕП

1. Жалпы мәлімет

Бұл есеп WhatsApp чатындағы мәтіндік хабарламаларды талдау негізінде жасалды. Жүйелік хабарламалар, аудио, видео және сурет орынбасарлары есепке алынбады. Негізгі назар оқушылардың өзара қарым-қатынасына, сөйлеу мәдениетіне және кибербуллинг белгілеріне аударылды.

Қатысушылар саны: {len(a['participants'])}. Мәтіндік хабарламалар саны: {len(a['text_messages'])}. Белсенді қатысушылар саны: {len(a['active'])}.

2. Жалпы сипаттама

Чаттың негізгі мазмұны оқу процесіне және ұйымдастыруға байланысты. Оқушылар үй тапсырмасын сұрайды, бір-біріне жауап береді, ақпарат алмасады. Сонымен қатар чат эмоционалды және белсенді сипатта.

3. Анықталған мәселелер

Қауіпті хабарламалар саны: {len(a['risky'])}. Жоғары қауіп: {len(a['high'])}, орта қауіп: {len(a['medium'])}, төмен қауіп: {len(a['low'])}.

Жоғары қауіп деңгейіндегі хабарламалар:
{quote_line(a['high'])}

Орта қауіп деңгейіндегі хабарламалар:
{quote_line(a['medium'])}

Төмен қауіп деңгейіндегі хабарламалар:
{quote_line(a['low'])}

4. Нақты ситуациялар

{situation_text}

5. Психологиялық анализ

Чаттың жалпы атмосферасы аралас сипатта: негізгі бөлігі қалыпты, бірақ кей жерлерде конфликт, қысым және агрессивті сөйлеу элементтері байқалады. Сынып белсенді, тез жауап беретін және эмоционалды ұжым ретінде көрінеді.

Белсенді қатысушылар: {active_text}. Бұл оқушылар чат атмосферасына көбірек әсер етеді.

6. Қауіп деңгейі

Жалпы қауіп деңгейі: {a['risk_level']}. Бұл деңгей кейбір хабарламаларда дөрекі сөйлеу, топтық қысым, мазақ немесе өзін қорғау реакциялары байқалуына байланысты қойылды.

7. Ұсыныстар

Бірінші, чат ережесін бекіту қажет: қорлау, айыптау, мазақ ету және конфликтті көпшілік чатта талқылауға болмайды.

Екінші, қауіпті немесе дөрекі хабарлама жазған оқушылармен жеке сөйлесу керек.

Үшінші, қысымға түскен немесе өзін қорғауға мәжбүр болған оқушымен жеке сөйлесіп, эмоционалдық жағдайын анықтау қажет.

Профилактика ретінде «Онлайн қарым-қатынас мәдениеті» тақырыбында сынып сағатын өткізу ұсынылады.

8. Қорытынды

Жалпы алғанда, чат оқу және ұйымдастыру мақсатында қолданылып отыр. Дегенмен кейбір хабарламаларда дөрекі сөйлеу, қысым және конфликт белгілері бар. Қазіргі жағдайда толық жүйелі буллинг анықталмады, бірақ ерте кезеңдегі қауіп белгілері бар.
"""


def create_word(report_text: str):
    document = Document()
    style = document.styles["Normal"]
    style.font.name = "Times New Roman"
    style.font.size = Pt(12)

    title = document.add_heading("CyberSafe Class Analyzer", level=1)
    title.alignment = 1

    for block in report_text.split("\n"):
        if block.strip():
            document.add_paragraph(block)
        else:
            document.add_paragraph("")

    buffer = io.BytesIO()
    document.save(buffer)
    buffer.seek(0)
    return buffer


# ---------------- UI ----------------
inject_css()
qr_b64 = make_qr(APP_URL)

st.markdown(
    f"""
    <div class="hero" id="cyber-safe-class-analyzer">
      <div class="hero-content">
        <div class="badge">🛡️ CyberSafe WhatsApp · AI қауіпсіздік жүйесі</div>
        <div class="hero-title">Кибербуллингті <span class="green">анықтау</span><br/>және алдын алу</div>
        <div class="hero-subtitle">
          WhatsApp TXT чаттарын автоматты талдап, қауіпті сөздер мен психологиялық қысым белгілерін анықтайды. Мұғалімге дайын Word есеп береді.
        </div>
      </div>
    </div>
    """,
    unsafe_allow_html=True,
)

st.markdown('<div class="section-title">🤖 CyberSafe робот жүйесі</div>', unsafe_allow_html=True)
hero_left, hero_right = st.columns([0.95, 1.45], gap="large")

with hero_left:
    st.markdown(
        f"""
        <div class="robot-card">
          <div class="robot-head">
            <div class="robot-antenna"></div>
            <div class="eye left"></div>
            <div class="eye right"></div>
          </div>
          <div style="font-size:24px;font-weight:900;color:white;margin-bottom:12px;">Сайтқа өту үшін сканерлеңіз</div>
          <div class="qr-wrap"><img src="data:image/png;base64,{qr_b64}" width="210" /></div>
          <div style="margin-top:14px;color:#9dffd5;font-weight:800;">cybersafe-analyzer.streamlit.app</div>
        </div>
        """,
        unsafe_allow_html=True,
    )

with hero_right:
    c1, c2 = st.columns(2)
    with c1:
        st.markdown('<div class="glass-card"><h3>💬 Чатты талдау</h3><p>WhatsApp TXT файлын оқып, хабарламаларды қатысушы, уақыт және мәтін бойынша жүйелейді.</p></div>', unsafe_allow_html=True)
        st.markdown('<div style="height:16px"></div>', unsafe_allow_html=True)
        st.markdown('<div class="glass-card"><h3>🧠 Психологиялық қорытынды</h3><p>Жалпы атмосфераны, қысым белгілерін және қауіпті қарым-қатынас үлгілерін көрсетеді.</p></div>', unsafe_allow_html=True)
    with c2:
        st.markdown('<div class="glass-card"><h3>⚠️ Қауіпті сөздерді табу</h3><p>Қорлау, мазақ, агрессия, топтық қысым және өзін қорғау реакцияларын бөледі.</p></div>', unsafe_allow_html=True)
        st.markdown('<div style="height:16px"></div>', unsafe_allow_html=True)
        st.markdown('<div class="glass-card"><h3>📄 Word есеп</h3><p>Мектепке тапсыруға дайын қазақша психологиялық есепті Word форматында жүктейді.</p></div>', unsafe_allow_html=True)

st.markdown('<div class="section-title">📥 Чатты енгізу және анализ жасау</div>', unsafe_allow_html=True)
left, right = st.columns([1, 1], gap="large")

with left:
    uploaded_file = st.file_uploader("WhatsApp .txt файл жүктеу", type=["txt"])
    pasted_text = st.text_area(
        "Немесе TXT мәтінді осында вставить етіңіз",
        height=280,
        placeholder="[12.11.2025, 22:36:55] Аты: Хабарлама...",
    )

    raw_text = ""
    if uploaded_file is not None:
        raw_text = uploaded_file.read().decode("utf-8", errors="ignore")
        st.success(f"Файл жүктелді: {uploaded_file.name}")
    elif pasted_text.strip():
        raw_text = pasted_text
        st.success("Мәтін қабылданды")

    analyze_button = st.button("🔍 Анализ жасау", use_container_width=True, type="primary")

with right:
    st.markdown('<div class="glass-card"><h3>⚙️ Қалай жұмыс істейді?</h3><p>1. TXT жүктеңіз немесе мәтінді қойыңыз.<br/>2. Анализ батырмасын басыңыз.<br/>3. Қауіп деңгейі мен нақты мысалдарды көріңіз.<br/>4. Word есепті жүктеп алыңыз.</p></div>', unsafe_allow_html=True)
    st.markdown('<div style="height:16px"></div>', unsafe_allow_html=True)
    st.markdown('<div class="glass-card"><h3>🎯 Мақсатымыз</h3><p>Қауіпсіз, қолдау көрсететін және достық атмосферадағы сынып құру.</p></div>', unsafe_allow_html=True)

if analyze_button:
    if not raw_text.strip():
        st.error("Алдымен TXT файл жүктеңіз немесе мәтінді вставить етіңіз.")
    else:
        analysis = analyze_chat(raw_text)
        report = generate_report(analysis)
        word_file = create_word(report)
        st.session_state["analysis"] = analysis
        st.session_state["report"] = report
        st.session_state["word_file"] = word_file

if "analysis" in st.session_state:
    analysis = st.session_state["analysis"]
    report = st.session_state["report"]

    st.markdown('<div class="section-title">📊 Анализ нәтижесі</div>', unsafe_allow_html=True)
    m1, m2, m3, m4 = st.columns(4)
    m1.metric("Қатысушылар", len(analysis["participants"]))
    m2.metric("Мәтіндік хабарлама", len(analysis["text_messages"]))
    m3.metric("Қауіпті хабарлама", len(analysis["risky"]))
    m4.metric("Қауіп деңгейі", analysis["risk_level"])

    d1, d2 = st.columns(2)
    with d1:
        st.download_button(
            "📄 Word отчет жүктеу",
            data=st.session_state["word_file"],
            file_name="cybersafe_report.docx",
            mime="application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            use_container_width=True,
        )
    with d2:
        st.download_button(
            "📝 TXT отчет жүктеу",
            data=report.encode("utf-8"),
            file_name="cybersafe_report.txt",
            mime="text/plain",
            use_container_width=True,
        )

    st.text_area("Дайын есеп", report, height=520)

    st.markdown('<div class="section-title">🚨 Қауіпті хабарламалар кестесі</div>', unsafe_allow_html=True)
    risky_table = [
        {
            "Дата": m["date"],
            "Уақыт": m["time"],
            "Қатысушы": m["sender"],
            "Хабарлама": m["message"][:180],
            "Белгі": m["category"],
            "Қауіп": m["risk"],
            "Ұсыныс": m["recommendation"],
        }
        for m in analysis["risky"][:100]
    ]
    st.dataframe(risky_table, use_container_width=True)

st.markdown('<div class="footer-banner">ҚАУІПСІЗ СЫНЫП — САПАЛЫ БІЛІМ НЕГІЗІ!</div>', unsafe_allow_html=True)
