from pathlib import Path
import telebot
from telebot import types
from telethon import TelegramClient
from telethon.errors import SessionPasswordNeededError
from telethon.tl.functions.account import ReportPeerRequest
from telethon.tl.types import InputReportReasonSpam, InputReportReasonFake, InputReportReasonOther
import asyncio
import os
import json
import time
import threading

# ======= إعدادات بوت تيليجرام =======
BOT_TOKEN = "7582123449:AAHdJdWMI8btCsaWTXRnQfaxRP3SSH1l-SA"
ADMIN_ID = 7062555969
MODS_FILE = "mods.json"
SUBS_FILE = "subs.json"
SESSIONS_INFO_FILE = "sessions_info.json"

bot = telebot.TeleBot(BOT_TOKEN)
session_data = {}

@bot.callback_query_handler(func=lambda call: call.data == "back_to_home")
def go_home(call):
    bot.answer_callback_query(call.id)
    # إعادة تنفيذ /start يدويًا
    start_bot(call.message)
    
# ======= تحميل وحفظ البيانات =======
def load_json(file, default):
    if os.path.exists(file):
        with open(file, "r") as f:
            return json.load(f)
    return default

def save_json(file, data):
    with open(file, "w") as f:
        json.dump(data, f)

MODS = load_json(MODS_FILE, [])
SUBS = load_json(SUBS_FILE, {})
SESSIONS_INFO = load_json(SESSIONS_INFO_FILE, {})

def is_admin(user_id):
    return user_id == ADMIN_ID

def is_mod(user_id):
    return user_id in MODS

def has_subscription(user_id):
    expire = SUBS.get(str(user_id))
    return expire and time.time() < expire

def is_authorized(user_id):
    return is_admin(user_id) or is_mod(user_id) or has_subscription(user_id)

# ======= أوامر وبداية البوت =======
@bot.message_handler(commands=['start'])
def start_bot(message):
    user_id = message.from_user.id
    if not is_authorized(user_id):
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("👤 المطور", url=f"https://t.me/MeSaEkO"))
        bot.reply_to(message, "❌ هذا البوت مخصص للمطور والمشرفين والمشتركين فقط.", reply_markup=markup)
        return

    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("➕ إضافة حساب", callback_data="add_account"))
    markup.add(types.InlineKeyboardButton("🚨 إرسال بلاغ", callback_data="start_report"))
    markup.add(types.InlineKeyboardButton("🧾 حساباتي", callback_data="my_accounts"))
    markup.add(types.InlineKeyboardButton("👤 المطور", url=f"https://t.me/MeSaEkO"))
   
    if is_admin(user_id) or is_mod(user_id):
        markup.add(types.InlineKeyboardButton("➕ إضافة مستخدم", callback_data="add_mod"))
        markup.add(types.InlineKeyboardButton("➖ حذف مستخدم", callback_data="remove_mod"))
        markup.add(types.InlineKeyboardButton("➕ إضافة اشتراك", callback_data="add_sub"))
        markup.add(types.InlineKeyboardButton("➖ حذف اشتراك", callback_data="remove_sub"))
    
    markup.add(types.InlineKeyboardButton("🗑 حذف حساب", callback_data="delete_account"))
    markup.add(types.InlineKeyboardButton("ℹ️ عرض المعلومات", callback_data="info"))
    bot.send_message(message.chat.id, "🎮 تحكم كامل في البوت:", reply_markup=markup)
    
# ======= إضافة حساب (تسجيل دخول) =======
from telethon import TelegramClient
from telethon.errors import SessionPasswordNeededError

# مكان لتخزين مؤقت بيانات المستخدم أثناء التفاعل
session_data = {}

@bot.callback_query_handler(func=lambda c: c.data == "add_account")
def add_account_step1(call):
    bot.send_message(call.message.chat.id, "📞 أرسل رقم الهاتف مع رمز الدولة:")
    bot.register_next_step_handler(call.message, send_code_request_step)
    
def run_async_task(coro):
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop.run_until_complete(coro)
    loop.close()

def send_code_request_step(message):
    phone = message.text.strip()
    user_id = message.from_user.id
    session_name = f"{user_id}_{int(time.time())}" # Generate a session name early

    # Temporarily store phone, and a placeholder for client until api_id/hash are known
    session_data[user_id] = {"phone": phone, "session_name": session_name}
    
    # We need a dummy client to send code request without api_id/hash first.
    # The actual client with correct api_id/hash will be created later.
    # For initial code request, Telethon can infer some parameters.
    # However, to properly send a code and handle responses, api_id/api_hash are typically needed from the start.
    # To simplify, we'll ask for phone, then code, then API details.
    # This means the code request happens *after* we have API ID/Hash.
    # So, the flow needs to be: Phone -> API ID -> API Hash -> Send Code -> Code -> 2FA.

    # Let's re-arrange the flow to make sense with Telethon's requirements:
    # 1. Ask for Phone
    # 2. Ask for API ID
    # 3. Ask for API Hash
    # 4. Attempt to send code request
    # 5. Ask for code
    # 6. Ask for 2FA password (if needed)

    bot.send_message(message.chat.id, "🔢 أرسل API ID:")
    bot.register_next_step_handler(message, add_account_step2)

def add_account_step2(message): # Now this asks for API ID
    user_id = message.from_user.id
    if user_id not in session_data or "phone" not in session_data[user_id]:
        bot.send_message(message.chat.id, "❌ لم يتم إرسال الرقم أولاً، أعد إضافة الحساب من البداية.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return
    try:
        api_id = int(message.text.strip())
        session_data[user_id]["api_id"] = api_id
    except ValueError:
        bot.send_message(message.chat.id, "❌ يرجى إدخال رقم صحيح لـ API ID.")
        bot.register_next_step_handler(message, add_account_step2) # Re-ask for API ID
        return
    bot.send_message(message.chat.id, "🔑 أرسل API HASH:")
    bot.register_next_step_handler(message, add_account_step3) # Now this asks for API Hash

def add_account_step3(message): # Now this asks for API Hash
    user_id = message.from_user.id
    api_hash = message.text.strip()
    data = session_data.get(user_id)
    if not data or "phone" not in data or "api_id" not in data:
        bot.send_message(message.chat.id, "❌ لا توجد بيانات كافية لإضافة الحساب.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return

    api_id = data.get("api_id")
    phone = data.get("phone")
    session_name = data.get("session_name") # Retrieve session_name

    session_data[user_id]["api_hash"] = api_hash # Store api_hash

    async def send_code_request_async():
        try:
            # Create a new client with actual api_id and api_hash
            client = TelegramClient(session_name, api_id, api_hash)
            await client.connect()
            if not await client.is_user_authorized():
                await client.send_code_request(phone)
                session_data[user_id]["client"] = client # Store client instance
                bot.send_message(message.chat.id, "📨 تم إرسال الكود، أرسله:")
                bot.register_next_step_handler(message, confirm_code_step)
            else:
                await client.disconnect()
                bot.send_message(message.chat.id, "✅ الحساب متصل مسبقًا.")
                markup = types.InlineKeyboardMarkup()
                markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
                bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        except Exception as e:
            bot.send_message(message.chat.id, f"❌ فشل إرسال الكود: {e}")
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
            bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
    
    threading.Thread(target=run_async_task, args=(send_code_request_async(),)).start()

def confirm_code_step(message):
    user_id = message.from_user.id
    code = message.text.strip()
    data = session_data.get(user_id)
    if not data or "client" not in data:
        bot.send_message(message.chat.id, "❌ لا توجد جلسة تسجيل دخول نشطة.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return

    client = data["client"]
    phone = data["phone"]
    session_name = data["session_name"]

    async def complete_sign_in():
        try:
            await client.sign_in(phone, code)
            SESSIONS_INFO[session_name] = {
                "api_id": data["api_id"],
                "api_hash": data["api_hash"]
            }
            save_json(SESSIONS_INFO_FILE, SESSIONS_INFO)
            await client.disconnect()
            bot.send_message(message.chat.id, "✅ تم تسجيل الجلسة بنجاح.")
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
            bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        except SessionPasswordNeededError:
            bot.send_message(message.chat.id, "🔐 أرسل كلمة المرور (التحقق بخطوتين):")
            bot.register_next_step_handler(message, two_step_password_step)
            
        except Exception as e:
            bot.send_message(message.chat.id, f"❌ فشل التحقق بالكود: {e}")
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
            bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)

    threading.Thread(target=run_async_task, args=(complete_sign_in(),)).start()

def two_step_password_step(message):
    user_id = message.from_user.id
    password = message.text.strip()
    data = session_data.get(user_id)
    if not data or "client" not in data:
        bot.send_message(message.chat.id, "❌ لا توجد جلسة تسجيل دخول نشطة.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return

    client = data["client"]
    session_name = data["session_name"]

    async def complete_2fa():
        try:
            await client.sign_in(password=password)
            SESSIONS_INFO[session_name] = {
                "api_id": data["api_id"],
                "api_hash": data["api_hash"]
            }
            save_json(SESSIONS_INFO_FILE, SESSIONS_INFO)
            await client.disconnect()
            bot.send_message(message.chat.id, "✅ تم تسجيل الدخول وحفظ الجلسة.")
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
            bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        except Exception as e:
            bot.send_message(message.chat.id, f"❌ فشل تسجيل الدخول: {e}")
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
            bot.send_message(message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)

    threading.Thread(target=run_async_task, args=(complete_2fa(),)).start()


# ======= إرسال بلاغ =======
@bot.callback_query_handler(func=lambda call: call.data == "start_report")
def ask_report_session(call):
    bot.answer_callback_query(call.id)
    user_id = call.from_user.id
    sessions = [f for f in os.listdir() if f.startswith(str(user_id)) and f.endswith(".session")]
    if not sessions:
        bot.send_message(call.message.chat.id, "❌ لا توجد جلسات.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(call.message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return
    markup = types.InlineKeyboardMarkup()
    for i, f in enumerate(sessions):
        markup.add(types.InlineKeyboardButton(f"📤 بلاغ من جلسة {i+1}", callback_data=f"report_{f}"))
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🗂️ اختر الجلسة:", reply_markup=markup)
    
@bot.callback_query_handler(func=lambda call: call.data.startswith("report_"))
def handle_report_session(call):
    bot.answer_callback_query(call.id)
    user_id = call.from_user.id
    session_data[user_id] = {"session": call.data.replace("report_", "")}
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("👤 حساب", callback_data="target_user"))
    markup.add(types.InlineKeyboardButton("📢 قناة", callback_data="target_channel"))
    markup.add(types.InlineKeyboardButton("👥 مجموعة", callback_data="target_group"))
    markup.add(types.InlineKeyboardButton("🤖 بوت", callback_data="target_bot"))
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🎯 اختر نوع الهدف:", reply_markup=markup)
    
@bot.callback_query_handler(func=lambda call: call.data.startswith("target_"))
def handle_target_type(call):
    bot.answer_callback_query(call.id)
    user_id = call.from_user.id
    session_data[user_id]["target_type"] = call.data.replace("target_", "")
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🚫 سبام", callback_data="reason_spam"))
    markup.add(types.InlineKeyboardButton("🕵️ وهمي", callback_data="reason_fake"))
    markup.add(types.InlineKeyboardButton("📛 محتوى مخالف", callback_data="reason_other"))
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "📌 اختر نوع البلاغ:", reply_markup=markup)
    
@bot.callback_query_handler(func=lambda call: call.data.startswith("reason_"))
def handle_reason(call):
    bot.answer_callback_query(call.id)
    user_id = call.from_user.id
    session_data[user_id]["reason"] = call.data.replace("reason_", "")
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🔁 أرسل عدد مرات البلاغ (مثل 5 أو 10):", reply_markup=markup)
    bot.register_next_step_handler(call.message, ask_target_and_report)
   
def ask_target_and_report(message):
    user_id = message.from_user.id
    try:
        count = int(message.text.strip())
        session_data[user_id]["report_count"] = count
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "📎 أرسل رابط أو @username أو ID للهدف:", reply_markup=markup)
        bot.register_next_step_handler(message, process_target)
       
    except:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ يرجى إدخال رقم صحيح.", reply_markup=markup)
        bot.register_next_step_handler(message, ask_target_and_report)
      
def process_target(message):
    user_id = message.from_user.id
    data = session_data.get(user_id)
    if not data:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ حدث خطأ، يرجى المحاولة مجددًا.", reply_markup=markup)
        return

    target = message.text.strip()
    session_file = data.get("session")
    reason_key = data.get("reason")
    report_count = data.get("report_count", 1)

    if not session_file or not reason_key:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ بيانات غير كاملة.", reply_markup=markup)
        return

    sess_info = SESSIONS_INFO.get(session_file)
    if not sess_info:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ لم يتم العثور على معلومات الجلسة.", reply_markup=markup)
        return

    api_id = sess_info["api_id"]
    api_hash = sess_info["api_hash"]

    reason_map = {
        "spam": InputReportReasonSpam(),
        "fake": InputReportReasonFake(),
        "other": InputReportReasonOther()
    }
    reason = reason_map.get(reason_key, InputReportReasonOther())

    async def report():
        try:
            client = TelegramClient(session_file, api_id, api_hash)
            await client.start()

            # الانضمام إذا كان الهدف قناة أو مجموعة
            try:
                await client.join_chat(target)
            except:
                pass

            entity = await client.get_entity(target)

            for _ in range(report_count):
                await client(ReportPeerRequest(
                    peer=entity,
                    reason=reason,
                    message="تم الإبلاغ عبر البوت"
                ))
                await asyncio.sleep(1)

            await client.disconnect()
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
            bot.send_message(message.chat.id, f"✅ تم إرسال {report_count} بلاغ.", reply_markup=markup)
        except Exception as e:
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
            bot.send_message(message.chat.id, f"❌ فشل البلاغ: {e}", reply_markup=markup)

    threading.Thread(target=run_async_task, args=(report(),)).start()

# ======= إدارة الجلسات =======
@bot.callback_query_handler(func=lambda call: call.data == "my_accounts")
def list_sessions(call):
    bot.answer_callback_query(call.id)
    user_id = call.from_user.id
    files = [f for f in os.listdir() if f.startswith(str(user_id)) and f.endswith(".session")]
    if not files:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(call.message.chat.id, "❌ لا توجد حسابات.", reply_markup=markup)
        return
    msg = "📲 حساباتك:\n" + "\n".join([f"{i+1}. {f}" for i, f in enumerate(files)])
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, msg, reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data == "delete_account")
def delete_session_prompt(call):
    bot.answer_callback_query(call.id)
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🗑 أرسل رقم الجلسة التي تريد حذفها:", reply_markup=markup)
    bot.register_next_step_handler(call.message, delete_session)
    
def delete_session(message):
    user_id = message.from_user.id
    try:
        num = int(message.text.strip())
    except:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ يرجى إرسال رقم صحيح.", reply_markup=markup)
        return
    files = [f for f in os.listdir() if f.startswith(str(user_id)) and f.endswith(".session")]
    if 0 < num <= len(files):
        session_file = files[num-1]
        os.remove(session_file)
        # حذف بيانات الجلسة من SESSIONS_INFO
        if session_file in SESSIONS_INFO:
            del SESSIONS_INFO[session_file]
            save_json(SESSIONS_INFO_FILE, SESSIONS_INFO)
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "✅ تم حذف الجلسة.", reply_markup=markup)
    else:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ رقم غير صحيح.", reply_markup=markup)
    

# ======= إدارة المشرفين =======
@bot.callback_query_handler(func=lambda call: call.data == "add_mod")
def ask_add_mod(call):
    bot.answer_callback_query(call.id)
    if not (is_admin(call.from_user.id) or is_mod(call.from_user.id)):
        bot.answer_callback_query(call.id, "❌ صلاحية مرفوضة.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(call.message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🔹 أرسل ID المستخدم لإضافته كمشرف:", reply_markup=markup)
    bot.register_next_step_handler(call.message, add_mod)
    

def add_mod(message):
    try:
        user_id = int(message.text.strip())
    except:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ يرجى إدخال رقم صحيح.", reply_markup=markup)
        return
    if user_id not in MODS:
        MODS.append(user_id)
        save_json(MODS_FILE, MODS)
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "✅ تم إضافة المستخدم كمشرف.", reply_markup=markup)
    else:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "⚠️ المستخدم موجود مسبقًا كمشرف.", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data == "remove_mod")
def ask_remove_mod(call):
    bot.answer_callback_query(call.id)
    if not (is_admin(call.from_user.id) or is_mod(call.from_user.id)):
        bot.answer_callback_query(call.id, "❌ صلاحية مرفوضة.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(call.message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🔸 أرسل ID المستخدم لحذفه من المشرفين:", reply_markup=markup)
    bot.register_next_step_handler(call.message, remove_mod)
    
def remove_mod(message):
    try:
        user_id = int(message.text.strip())
    except:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ يرجى إدخال رقم صحيح.", reply_markup=markup)
        return
    if user_id in MODS:
        MODS.remove(user_id)
        save_json(MODS_FILE, MODS)
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "✅ تم حذف المستخدم من المشرفين.", reply_markup=markup)
    else:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "⚠️ المستخدم غير موجود ضمن المشرفين.", reply_markup=markup)

# ======= إدارة الاشتراكات =======
@bot.callback_query_handler(func=lambda call: call.data == "add_sub")
def ask_sub_id(call):
    bot.answer_callback_query(call.id)
    if not (is_admin(call.from_user.id) or is_mod(call.from_user.id)):
        bot.answer_callback_query(call.id, "❌ صلاحية مرفوضة.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(call.message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🆔 أرسل ID المستخدم لإضافة اشتراك له:", reply_markup=markup)
    bot.register_next_step_handler(call.message, ask_sub_duration)
    
def ask_sub_duration(message):
    try:
        user_id = int(message.text.strip())
    except:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ يرجى إدخال ID صحيح.", reply_markup=markup)
        return
    session_data[message.from_user.id] = {"sub_user": user_id}
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(message.chat.id, "⏳ أرسل المدة بالأيام:", reply_markup=markup)
    bot.register_next_step_handler(message, add_subscription)
    
def add_subscription(message):
    try:
        days = int(message.text.strip())
    except:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ يرجى إدخال مدة صحيحة.", reply_markup=markup)
        return

    uid = session_data[message.from_user.id].get("sub_user")
    if not uid:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "❌ خطأ في البيانات، أعد المحاولة.", reply_markup=markup)
        return

    SUBS[str(uid)] = time.time() + days * 86400
    save_json(SUBS_FILE, SUBS)
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(message.chat.id, "✅ تم إضافة الاشتراك.", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data == "remove_sub")
def ask_remove_sub(call):
    bot.answer_callback_query(call.id)
    if not (is_admin(call.from_user.id) or is_mod(call.from_user.id)):
        bot.answer_callback_query(call.id, "❌ صلاحية مرفوضة.")
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(call.message.chat.id, "العودة للقائمة الرئيسية:", reply_markup=markup)
        return
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, "🆔 أرسل ID المستخدم لحذف اشتراكه:", reply_markup=markup)
    bot.register_next_step_handler(call.message, remove_subscription)
    
def remove_subscription(message):
    uid = message.text.strip()
    if uid in SUBS:
        del SUBS[uid]
        save_json(SUBS_FILE, SUBS)
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "✅ تم حذف الاشتراك.", reply_markup=markup)
    else:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
        bot.send_message(message.chat.id, "⚠️ لا يوجد اشتراك لهذا المستخدم.", reply_markup=markup)

# ======= عرض معلومات المستخدم =======
@bot.callback_query_handler(func=lambda call: call.data == "info")
def show_info(call):
    bot.answer_callback_query(call.id)
    user_id = call.from_user.id
    subs = SUBS.get(str(user_id), None)
    mod_status = "نعم" if is_mod(user_id) else "لا"
    
    if subs and time.time() < subs:
        remaining = int(subs - time.time())
        days = remaining // 86400
        hours = (remaining % 86400) // 3600
        sub_text = f"✅ مشترك ({days} يوم، {hours} ساعة متبقية)"
    else:
        sub_text = "❌ غير مشترك"
    
    info_msg = f"""🧾 معلوماتك:
👤 ID: `{user_id}`
🛡 مشرف: {mod_status}
📅 اشتراك: {sub_text}
"""
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🔙 رجوع", callback_data="back_to_home"))
    bot.send_message(call.message.chat.id, info_msg, parse_mode="Markdown", reply_markup=markup)
        
# ======= تشغيل البوت =======
print("🤖 البوت يعمل الآن...")
bot.infinity_polling()
