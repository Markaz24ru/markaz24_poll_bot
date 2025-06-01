# markaz24_poll_bot
import logging
import random
import os
from aiogram import Bot, Dispatcher, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.utils import executor
from dotenv import load_dotenv

load_dotenv()

BOT_TOKEN = os.getenv("BOT_TOKEN")  # .env fayldan token olinadi
CHANNEL_USERNAME = '@markaz24uz'
GROUP_ID = -1002475664726

logging.basicConfig(level=logging.INFO)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(bot, storage=MemoryStorage())

class Form(StatesGroup):
    phone = State()
    captcha = State()
    vote = State()

@dp.message_handler(commands='start')
async def start_cmd(message: types.Message, state: FSMContext):
    user_id = message.from_user.id
    member = await bot.get_chat_member(chat_id=CHANNEL_USERNAME, user_id=user_id)
    if member.status not in ['member', 'administrator', 'creator']:
        markup = InlineKeyboardMarkup().add(
            InlineKeyboardButton("✅ Obuna bo‘lish", url=f"https://t.me/{CHANNEL_USERNAME[1:]}")
        )
        await message.answer("⛔ Botdan foydalanish uchun avval kanalga obuna bo‘ling!", reply_markup=markup)
        return

    contact_btn = KeyboardButton("📲 Telefon raqamni yuborish", request_contact=True)
    markup = ReplyKeyboardMarkup(resize_keyboard=True).add(contact_btn)
    await message.answer("📞 Iltimos, telefon raqamingizni yuboring:", reply_markup=markup)
    await Form.phone.set()

@dp.message_handler(content_types=types.ContentType.CONTACT, state=Form.phone)
async def get_contact(message: types.Message, state: FSMContext):
    phone = message.contact.phone_number
    await state.update_data(phone=phone)

    code = random.randint(1000, 9999)
    await state.update_data(captcha=code)
    await message.answer(f"🤖 Robot emasligingizni tasdiqlang. Ushbu kodni yuboring: {code}")
    await Form.captcha.set()

@dp.message_handler(state=Form.captcha)
async def check_captcha(message: types.Message, state: FSMContext):
    user_code = message.text.strip()
    data = await state.get_data()
    if user_code != str(data['captcha']):
        await message.answer("❌ Noto‘g‘ri kod. Iltimos, qayta urinib ko‘ring.")
        return
    await message.answer("✅ CAPTCHA tasdiqlandi. Endi so‘rovnomada ovoz bering:")

    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        InlineKeyboardButton("🗳 Variant 1", callback_data="vote_1"),
        InlineKeyboardButton("🗳 Variant 2", callback_data="vote_2"),
        InlineKeyboardButton("🗳 Variant 3", callback_data="vote_3"),
    )
    await message.answer_photo(
        photo="https://via.placeholder.com/600x400.png?text=So%27rovnoma+rasmi",
        caption="📊 Savol: Sizningcha eng yaxshi variant qaysi?",
        reply_markup=markup
    )
    await Form.vote.set()

@dp.callback_query_handler(lambda c: c.data.startswith("vote_"), state=Form.vote)
async def process_vote(callback: types.CallbackQuery, state: FSMContext):
    variant = callback.data.split("_")[1]
    data = await state.get_data()
    phone = data.get("phone")

    await bot.send_message(
        chat_id=GROUP_ID,
        text=f"📥 Yangi ovoz:\n📱 Telefon: {phone}\n✅ Tanlov: Variant {variant}"
    )
    await bot.answer_callback_query(callback.id, "✅ Ovoz qabul qilindi!")
    await bot.send_message(callback.from_user.id, "Rahmat! Siz muvaffaqiyatli ovoz berdingiz ✅")
    await state.finish()

if name == 'main':
    executor.start_polling(dp, skip_updates=True)
