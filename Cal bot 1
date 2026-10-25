import os
import sqlite3
from telegram import Update
from telegram.ext import ApplicationBuilder, MessageHandler, CommandHandler, ContextTypes, filters
from google import genai
from google.genai import types

# ထည့်သွင်းပြီးသား Bot Token နှင့် သင့်ရဲ့ Gemini API Key ထည့်ရန်
TELEGRAM_BOT_TOKEN = "8982232944:AAGc7L49tYteGCbF4KjM-LI4VCYaKWqrUNg"
GEMINI_API_KEY = "YOUR_GEMINI_API_KEY"

client = genai.Client(api_key=GEMINI_API_KEY)

# Database တည်ဆောက်ခြင်း
def init_db():
    conn = sqlite3.connect("weight_gain_bot.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS food_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            username TEXT,
            food_description TEXT,
            calories INTEGER,
            protein REAL,
            date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    conn.close()

init_db()

# ဓာတ်ပုံပို့လာပါက တွက်ချက်ပြီး Database သို့ သိမ်းမည့် လုပ်ဆောင်ချက်
async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    photo = update.message.photo[-1]
    file = await context.bot.get_file(photo.file_id)
    
    photo_path = f"food_{user.id}.jpg"
    await file.download_to_drive(photo_path)
    
    special_prompt = """
    ဒီအစားအစာပုံကို ကြည့်ပြီး Weight Gain (ကိုယ်အလေးချိန်တိုးရန်) အတွက် အောက်ပါအတိုင်း တွက်ချက်ပေးပါ၊ ပြီးလျှင် အောက်ပါ JSON ပုံစံအတိုင်း တိကျစွာ ပထမဆုံး ဖော်ပြပေးပါ -
    {
        "calories": စုစုပေါင်းကယ်လိုရီဂဏန်းသီးသန့် (ဥပမာ - 450),
        "protein": ပရိုတင်းဂရမ်ဂဏန်းသီးသန့် (ဥပမာ - 25.5)
    }
    အဲ့ဒီနောက်မှာမှ ပါဝင်ပစ္စည်းတစ်ခုချင်းစီရဲ့ အသေးစိတ်နဲ့ အကြံပြုချက်တွေကို မြန်မာလို ဆက်လက် ရေးသားပေးပါ။
    """

    with open(photo_path, "rb") as f:
        image_bytes = f.read()

    response = client.models.generate_content(
        model='gemini-2.5-flash',
        contents=[
            types.Part.from_bytes(data=image_bytes, mime_type='image/jpeg'),
            special_prompt
        ]
    )
    
    result_text = response.text
    
    calories = 400
    protein = 20.0
    try:
        import json
        json_start = result_text.find("{")
        json_end = result_text.find("}") + 1
        if json_start != -1 and json_end != -1:
            data = json.loads(result_text[json_start:json_end])
            calories = int(data.get("calories", 400))
            protein = float(data.get("protein", 20.0))
    except Exception as e:
        print("Parsing Error:", e)

    conn = sqlite3.connect("weight_gain_bot.db")
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO food_logs (user_id, username, food_description, calories, protein) VALUES (?, ?, ?, ?, ?)",
        (user.id, user.username or user.first_name, "Photo Meal", calories, protein)
    )
    conn.commit()
    conn.close()

    await update.message.reply_text(f"✅ **မှတ်တမ်းတင်ပြီးပါပြီ!** (+{calories} kcal, +{protein}g Protein)\n\n" + result_text)

# 📊 နေ့စဉ် စုစုပေါင်း စာရင်းချုပ်ကြည့်မည့် Command (/summary)
async def summary_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    
    conn = sqlite3.connect("weight_gain_bot.db")
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT SUM(calories), SUM(protein), COUNT(*) 
        FROM food_logs 
        WHERE user_id = ? AND DATE(date) = DATE('now')
    """, (user.id,))
    
    total_cal, total_protein, meals_count = cursor.fetchone()
    conn.close()
    
    total_cal = total_cal or 0
    total_protein = total_protein or 0
    meals_count = meals_count or 0
    
    summary_msg = (
        f"📊 **ယနေ့ စားသောက်မှု စာရင်းချုပ် (Daily Summary)**\n\n"
        f"🍽 စုစုပေါင်း စားသုံးခဲ့သော အကြိမ်ရေ: {meals_count} ကြိမ်\n"
        f"🔥 စုစုပေါင်း ကယ်လိုရီ: **{total_cal} kcal**\n"
        f"🥩 စုစုပေါင်း ပရိုတင်း: **{total_protein:.1f} g**\n\n"
        f"💡 Weight Gain ပန်းတိုင်အတွက် ဆက်လက် ကြိုးစားပါ!"
    )
    
    await update.message.reply_text(summary_msg)

def main():
    app = ApplicationBuilder().token(TELEGRAM_BOT_TOKEN).build()
    
    app.add_handler(MessageHandler(filters.PHOTO, handle_photo))
    app.add_handler(CommandHandler("summary", summary_command))
    
    print("Weight Gain Bot စတင်အလုပ်လုပ်နေပါပြီ...")
    app.run_polling()

if __name__ == '__main__':
    main()

