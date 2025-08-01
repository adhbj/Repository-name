import time
from datetime import timedelta, datetime
from aiogram import Bot, Dispatcher, executor, types
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.dispatcher import FSMContext

import firebase_admin
from firebase_admin import credentials, firestore

# --- إعدادات ---
BOT_TOKEN = "8205607343:AAHiesn79YRyFk59G7MLKrsD_whFkunANtM"
ADMIN_ID = 5519696777
SUPPORT_USERNAME = "@SX_k1"
WALLET_ADDRESS = "TVbwydaHcC8QugSDSe8xqwCm7yUCdTpWoz"

# --- تهيئة Firebase ---
cred = credentials.Certificate("firebase_credentials.json")  # ملف JSON الخاص بـ Firebase
firebase_admin.initialize_app(cred)
db = firestore.client()

# --- بوت تيليجرام ---
storage = MemoryStorage()
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(bot, storage=storage)

# --- حالات FSM للإيداع ---
class DepositStates(StatesGroup):
    waiting_for_amount = State()
    waiting_for_proof = State()

# --- حالات FSM للسحب ---
class WithdrawStates(StatesGroup):
    waiting_for_amount = State()

# --- تخزين طلبات مؤقتة للإدارة ---
deposit_requests = {}   # user_id(str) : amount(float)
withdraw_requests = {}  # user_id(str) : amount(float)

# --- دوال مساعدة ---
def get_user_doc(user_id):
    return db.collection("users").document(str(user_id))

async def user_exists(user_id):
    doc = get_user_doc(user_id).get()
    return doc.exists

async def create_user(user_id):
    doc_ref = get_user_doc(user_id)
    doc_ref.set({
        "invested": 0.0,
        "total_profit": 0.0,
        "withdrawals": 0.0,
        "subscribed": False,
        "last_profit_time": 0
    })

async def get_user_data(user_id):
    doc = get_user_doc(user_id).get()
    if doc.exists:
        return doc.to_dict()
    else:
        await create_user(user_id)
        return get_user_doc(user_id).get().to_dict()

async def update_user_data(user_id, data):
    get_user_doc(user_id).update(data)

# --- لوحة تحكم المدير بالأزرار ---
def admin_keyboard():
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    keyboard.add(
        types.InlineKeyboardButton("📥 طلبات الإيداع", callback_data="admin_deposits"),
        types.InlineKeyboardButton("📤 طلبات السحب", callback_data="admin_withdraws")
    )
    return keyboard

def activate_subscription_button(user_id, amount):
    keyboard = types.InlineKeyboardMarkup()
    keyboard.add(types.InlineKeyboardButton(f"✅ تفعيل اشتراك {user_id} بـ {amount}$", callback_data=f"activate_{user_id}_{amount}"))
    return keyboard

def confirm_withdrawal_button(user_id, amount):
    keyboard = types.InlineKeyboardMarkup()
    keyboard.add(types.InlineKeyboardButton(f"✅ تأكيد سحب {amount}$ من {user_id}", callback_data=f"confirm_{user_id}_{amount}"))
    return keyboard

# --- بدء المحادثة ---
@dp.message_handler(commands=["start"])
async def start_handler(message: types.Message):
    user_id = message.from_user.id
    if not await user_exists(user_id):
        await create_user(user_id)
    keyboard = types.InlineKeyboardMarkup(row_width=1)
    keyboard.add(
        types.InlineKeyboardButton("💸 تشغيل الربح اليومي", callback_data="daily_profit"),
        types.InlineKeyboardButton("💼 حالة اشتراكي", callback_data="status"),
        types.InlineKeyboardButton("📤 طلب سحب", callback_data="withdraw"),
        types.InlineKeyboardButton("💰 طلب إيداع", callback_data="deposit"),
        types.InlineKeyboardButton("📞 الدعم الفني", url=f"https://t.me/{SUPPORT_USERNAME.lstrip('@')}")
    )
    await message.answer(
        f"👋 مرحباً بك في بوت الاستثمار!\n\n"
        f"✅ اربح 3$ يوميًا لكل 100$ تستثمرها.\n"
        f"💳 لإيداع المبلغ، اضغط على 'طلب إيداع' وأرسل المبلغ مع إثبات.\n"
        f"⏳ بعد الإيداع، سيتم تفعيل اشتراكك من قبل الإدارة.",
        reply_markup=keyboard,
        parse_mode="Markdown"
    )

# --- التعامل مع أزرار المستخدم ---
@dp.callback_query_handler(lambda c: True)
async def callback_handler(callback_query: types.CallbackQuery, state: FSMContext):
    user_id = callback_query.from_user.id
    user_data = await get_user_data(user_id)
    now = time.time()
    data = callback_query.data

    if data == "daily_profit":
        if not user_data.get("subscribed", False):
            await callback_query.answer("🚫 غير مشترك. الرجاء إرسال طلب إيداع أولًا.", show_alert=True)
            return
        last_time = user_data.get("last_profit_time", 0)
        if now - last_time < 86400:
            rem = timedelta(seconds=int(86400 - (now - last_time)))
            await callback_query.answer(f"⏳ يمكنك التفعيل بعد: {rem}", show_alert=True)
            return
        profit = round(user_data["invested"] * 0.03, 2)
        user_data["total_profit"] += profit
        user_data["last_profit_time"] = now
        await update_user_data(user_id, {
            "total_profit": user_data["total_profit"],
            "last_profit_time": now
        })
        await callback_query.answer(f"✅ ربح {profit}$ تمت إضافته اليوم!", show_alert=True)

    elif data == "status":
        subscribed = "✅ نعم" if user_data.get("subscribed", False) else "❌ لا"
        invested = user_data.get("invested", 0.0)
        profit_day = round(invested * 0.03, 2)
        total_profit = user_data.get("total_profit", 0.0)
        withdrawals = user_data.get("withdrawals", 0.0)
        last_profit_time = user_data.get("last_profit_time", 0)
        last_profit_str = datetime.utcfromtimestamp(last_profit_time).strftime('%Y-%m-%d %H:%M:%S') if last_profit_time else "لم يتم التفعيل"
        await callback_query.message.edit_text(
            f"💼 حالة اشتراكك:\n"
            f"• مشترك: {subscribed}\n"
            f"• المبلغ المستثمر: {invested}$\n"
            f"• الربح اليومي: {profit_day}$\n"
            f"• إجمالي الأرباح: {total_profit}$\n"
            f"• إجمالي السحوبات: {withdrawals}$\n"
            f"• آخر تفعيل ربح يومي: {last_profit_str}",
            reply_markup=callback_query.message.reply_markup
        )

    elif data == "withdraw":
        await callback_query.message.answer("📥 أرسل المبلغ الذي تريد سحبه:")
        await WithdrawStates.waiting_for_amount.set()

    elif data == "deposit":
        await callback_query.message.answer(f"💰 أرسل مبلغ الإيداع الذي قمت به (بالدولار) إلى المحفظة:\n`{WALLET_ADDRESS}`", parse_mode="Markdown")
        await DepositStates.waiting_for_amount.set()

    # --- أزرار لوحة تحكم المدير ---
    elif user_id == ADMIN_ID:
        if data == "admin_deposits":
            if not deposit_requests:
                await callback_query.answer("لا توجد طلبات إيداع حالياً.", show_alert=True)
                return
            for uid, amount in deposit_requests.items():
                await bot.send_message(ADMIN_ID, f"طلب إيداع:\nالمستخدم: {uid}\nالمبلغ: {amount}$", reply_markup=activate_subscription_button(uid, amount))
            await callback_query.answer("تم عرض طلبات الإيداع.")

        elif data == "admin_withdraws":
            if not withdraw_requests:
                await callback_query.answer("لا توجد طلبات سحب حالياً.", show_alert=True)
                return
            for uid, amount in withdraw_requests.items():
                await bot.send_message(ADMIN_ID, f"طلب سحب:\nالمستخدم: {uid}\nالمبلغ: {amount}$", reply_markup=confirm_withdrawal_button(uid, amount))
            await callback_query.answer("تم عرض طلبات السحب.")

        elif data.startswith("activate_"):
            parts = data.split("_")
            uid = parts[1]
            amount = float(parts[2])
            await get_user_doc(uid).set({
                "invested": amount,
                "total_profit": 0.0,
                "withdrawals": 0.0,
                "subscribed": True,
                "last_profit_time": 0
            }, merge=True)
            if uid in deposit_requests:
                del deposit_requests[uid]
            await callback_query.answer(f"✅ تم تفعيل اشتراك المستخدم {uid} بمبلغ {amount}$")
            await bot.send_message(int(uid), f"✅ تم تفعيل اشتراكك بمبلغ {amount}$. يمكنك الآن تفعيل الربح اليومي.")
            await callback_query.message.edit_reply_markup(reply_markup=None)

        elif data.startswith("confirm_"):
            parts = data.split("_")
            uid = parts[1]
            amount = float(parts[2])
            user_data = await get_user_data(uid)
            new_withdrawals = user_data.get("withdrawals", 0) + amount
            await update_user_data(uid, {"withdrawals": new_withdrawals})
            if uid in withdraw_requests:
                del withdraw_requests[uid]
            await callback_query.answer(f"✅ تم تأكيد إرسال مبلغ {amount}$ للمستخدم {uid}")
            await bot.send_message(int(uid), f"✅ تم إرسال مبلغ {amount}$، تحقق من محفظتك.")
            await callback_query.message.edit_reply_markup(reply_markup=None)

# --- استقبال مبلغ السحب ---
@dp.message_handler(lambda m: m.text.replace('.', '', 1).isdigit(), state=WithdrawStates.waiting_for_amount)
async def process_withdraw(message: types.Message, state: FSMContext):
    user_id = message.from_user.id
    user_data = await get_user_data(user_id)
    amount = float(message.text)
    available = user_data.get("total_profit", 0) - user_data.get("withdrawals", 0)
    if amount > available:
        await message.reply(f"🚫 لا يمكنك سحب {amount}$، الرصيد المتاح هو {available}$.")
        return
    withdraw_requests[str(user_id)] = amount
    await message.reply("✅ تم طلب السحب بنجاح، قيد المعالجة (0 - 24 ساعة).")
    username = message.from_user.username or message.from_user.full_name or str(user_id)
    await bot.send_message(ADMIN_ID,
        f"🚨 طلب سحب جديد:\n👤 المستخدم: @{username}\n💰 المبلغ: {amount}$\n🆔 معرف: {user_id}\n\n"
        "✅ يمكن تأكيد السحب من خلال لوحة تحكم المدير."
    )
    await state.finish()

# --- استقبال مبلغ الإيداع ---
@dp.message_handler(state=DepositStates.waiting_for_amount)
async def deposit_amount_received(message: types.Message, state: FSMContext):
    try:
        amount = float(message.text)
        await state.update_data(amount=amount)
        await message.reply("📸 الآن أرسل صورة إثبات الإيداع (صورة أو ملف):")
        await DepositStates.waiting_for_proof.set()
    except:
        await message.reply("🚫 الرجاء إدخال رقم صحيح للمبلغ.")

# --- استقبال إثبات الإيداع ---
@dp.message_handler(content_types=["photo", "document"], state=DepositStates.waiting_for_proof)
async def deposit_proof_received(message: types.Message, state: FSMContext):
    data = await state.get_data()
    amount = data.get("amount")
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.full_name or str(user_id)

    caption = (
        f"📥 طلب إيداع جديد:\n"
        f"👤 المستخدم: @{username}\n"
        f"💰 المبلغ: {amount}$\n"
        f"🆔 المعرف: {user_id}\n\n"
        f"✅ تفعيل الاشتراك سيكون عبر لوحة تحكم المدير."
    )

    if message.content_type == "photo":
        photo = message.photo[-1].file_id
        await bot.send
