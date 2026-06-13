import os
import anthropic
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes

ANTHROPIC_API_KEY = os.environ.get("ANTHROPIC_API_KEY")
TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN")

client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)

# 카테고리 정의
CATEGORIES = {
    "mood": {
        "label": "지금 기분이 어때요?",
        "emoji": "💚",
        "options": ["지쳐있어요", "불안해요", "외로워요", "무기력해요", "화가 나요", "슬퍼요", "설레요", "그냥 괜찮아요", "뭔가 허전해요", "복잡해요", "후회가 돼요", "두려워요"]
    },
    "moment": {
        "label": "지금 어떤 순간이에요?",
        "emoji": "🌙",
        "options": ["하루를 시작하는 아침", "하루를 마무리하는 밤", "힘든 일이 있었어요", "중요한 결정 앞에 있어요", "새로운 시작을 앞두고 있어요", "오랫동안 혼자였어요", "누군가와 다퉜어요", "실패하거나 포기했어요", "잘 해냈는데 기쁘지 않아요", "아무 이유 없이 울고 싶어요"]
    },
    "personality": {
        "label": "나는 어떤 사람이에요?",
        "emoji": "🧡",
        "options": ["생각이 많아요", "감수성이 풍부해요", "남에게 잘 맞춰줘요", "혼자 삭이는 편이에요", "완벽하고 싶어요", "쉽게 포기 못해요", "상처받기 쉬워요", "책임감이 강해요", "자유롭고 싶어요", "인정받고 싶어요", "변화가 두려워요", "나를 잘 모르겠어요"]
    },
    "need": {
        "label": "지금 가장 필요한 게 뭐예요?",
        "emoji": "💛",
        "options": ["위로받고 싶어요", "용기가 필요해요", "그냥 괜찮다는 말", "명언이나 철학적인 말", "현실적인 조언", "나를 이해해줬으면", "잠깐 웃고 싶어요", "혼자가 아니란 느낌"]
    }
}

CATEGORY_ORDER = ["mood", "moment", "personality", "need"]

user_sessions = {}

def get_session(user_id):
    if user_id not in user_sessions:
        user_sessions[user_id] = {
            "step": 0,
            "selections": {k: [] for k in CATEGORIES}
        }
    return user_sessions[user_id]

def reset_session(user_id):
    user_sessions[user_id] = {
        "step": 0,
        "selections": {k: [] for k in CATEGORIES}
    }

def build_keyboard(category_key, selections):
    options = CATEGORIES[category_key]["options"]
    keyboard = []
    for i, option in enumerate(options):
        is_selected = option in selections
        label = ("✅ " if is_selected else "") + option
        keyboard.append([InlineKeyboardButton(label, callback_data=f"toggle|{category_key}|{i}")])
    keyboard.append([InlineKeyboardButton("➡️ 다음으로", callback_data=f"next|{category_key}")])
    return InlineKeyboardMarkup(keyboard)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    reset_session(user_id)
    await update.message.reply_text(
        "🌿 *오늘의 한 마디*\n\n지금 내 상태와 필요한 것을 골라보세요.\n딱 맞는 말을 건네드릴게요.\n\n/select 로 시작하세요!",
        parse_mode="Markdown"
    )

async def select(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    reset_session(user_id)
    session = get_session(user_id)
    category_key = CATEGORY_ORDER[0]
    cat = CATEGORIES[category_key]
    await update.message.reply_text(
        f"{cat['emoji']} *{cat['label']}*\n\n해당하는 것을 모두 선택하고 ➡️ 다음으로 눌러주세요.",
        parse_mode="Markdown",
        reply_markup=build_keyboard(category_key, session["selections"][category_key])
    )

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    session = get_session(user_id)
    data = query.data.split("|")
    action = data[0]

    if action == "toggle":
        category_key = data[1]
        option_idx = int(data[2])
        option = CATEGORIES[category_key]["options"][option_idx]
        selections = session["selections"][category_key]
        if option in selections:
            selections.remove(option)
        else:
            selections.append(option)
        cat = CATEGORIES[category_key]
        await query.edit_message_reply_markup(
            reply_markup=build_keyboard(category_key, selections)
        )

    elif action == "next":
        current_key = data[1]
        current_idx = CATEGORY_ORDER.index(current_key)
        next_idx = current_idx + 1

        if next_idx < len(CATEGORY_ORDER):
            next_key = CATEGORY_ORDER[next_idx]
            cat = CATEGORIES[next_key]
            await query.edit_message_text(
                f"{cat['emoji']} *{cat['label']}*\n\n해당하는 것을 모두 선택하고 ➡️ 다음으로 눌러주세요.",
                parse_mode="Markdown",
                reply_markup=build_keyboard(next_key, session["selections"][next_key])
            )
        else:
            await query.edit_message_text("✨ 선택 완료! 분석 중이에요...")
            await generate_message(query, user_id, session)

async def generate_message(query, user_id, session):
    sel = session["selections"]
    parts = []
    if sel["mood"]: parts.append("현재 기분: " + ", ".join(sel["mood"]))
    if sel["moment"]: parts.append("지금 순간: " + ", ".join(sel["moment"]))
    if sel["personality"]: parts.append("나라는 사람: " + ", ".join(sel["personality"]))
    if sel["need"]: parts.append("필요한 것: " + ", ".join(sel["need"]))

    if not parts:
        await query.message.reply_text("아무것도 선택하지 않으셨어요. /select 로 다시 시작해보세요!")
        return

    prompt = (
        "[오늘의 한 마디 요청]\n" + "\n".join(parts) +
        "\n\n위 내용을 바탕으로 이 사람에게 딱 맞는 말을 건네줘. "
        "형식적이거나 교과서 같은 말 말고, 진짜 옆에 있는 친한 사람이 해주는 것처럼 따뜻하고 솔직하게. "
        "철학적 명언이 필요하다고 했으면 명언도 자연스럽게 녹여줘. "
        "너무 길지 않게, 근데 울림 있게. 3~5문단 정도."
    )

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    )

    result = response.content[0].text
    await query.message.reply_text(
        f"🌿\n\n{result}\n\n---\n/select 로 다시 시작할 수 있어요.",
        parse_mode="Markdown"
    )
    reset_session(user_id)

def main():
    app = Application.builder().token(TELEGRAM_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("select", select))
    app.add_handler(CallbackQueryHandler(button_handler))
    print("봇 실행 중...")
    app.run_polling()

if __name__ == "__main__":
    main()
