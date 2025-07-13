```python
import json
import logging
import asyncio 
from aiogram import Bot, Dispatcher, types
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import WebAppInfo, ReplyKeyboardMarkup, KeyboardButton
from aiogram.client.default import DefaultBotProperties
from aiogram.filters import Command
from aiogram import F

API_TOKEN = '7690796647:AAHibbEzg3ky14fCNpJM2-_G7m4F_kSlqKI'
MANAGER_CHAT_ID = 6438939468

bot = Bot(token=API_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.MARKDOWN))
dp = Dispatcher(storage=MemoryStorage())

def get_main_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="🛒 Оформить заказ", web_app=WebAppInfo(url="https://tlpshop.ru/"))],
            [KeyboardButton(text="🔍 Навигация"), KeyboardButton(text="🕒 Режим работы")],
            [KeyboardButton(text="💬 Поддержка")]
        ],
        resize_keyboard=True,
        input_field_placeholder="Выберите действие"
    )

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    username = message.from_user.username or message.from_user.first_name
    await message.answer(
        f"☁️ *Добро пожаловать в TLP SHOP* ☁️\n\n"
        f"Привет, {username}! Мы предлагаем:\n"
        "• Премиум качество\n"
        "• Быструю доставку\n"
        "• Конфиденциальность\n\n"
        "_Используйте меню ниже для навигации:_",
        reply_markup=get_main_keyboard(),
        disable_web_page_preview=True
    )

@dp.message(F.text == "🔍 Навигация")
async def show_navigation(message: types.Message):
    navigation_text = (
        "🌐 *Контакты и навигация* 🌐\n\n"
        "▸ Менеджер: @tlp_manager\n"
        "▸ Канал: https://t.me/tlpshopmgdn\n"
        "▸ Отзывы: https://t.me/+VfwmutOo8R9hZWMy\n\n"
        "⏱ Ответ в течение 15 минут"
    )
    await message.answer(
        navigation_text,
        reply_markup=get_main_keyboard(),
        disable_web_page_preview=True
    )

@dp.message(F.text == "🕒 Режим работы")
async def working_hours(message: types.Message):
    await message.answer(
        "⏳ *График работы* ⏳\n\n"
        "• Ежедневно: 13:00 - 23:00\n\n"
        "🚗 Самовывоз по договорённости\n"
        "🚀 Доставка - без выходных",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.text == "💬 Поддержка")
async def support(message: types.Message):
    await message.answer(
        "📩 *Служба поддержки* 📩\n\n"
        "По вопросам и помощи:\n"
        "▸ @tlp_support_bot\n\n"
        "_Мы всегда рады помочь!_",
        reply_markup=get_main_keyboard(),
        disable_web_page_preview=True
    )

@dp.message(F.web_app_data)
async def handle_webapp_data(message: types.Message):
    try:
        data = json.loads(message.web_app_data.data)
        items = data.get('items', [])

        if not items:
            await message.answer("🛒 Ваша корзина пуста", reply_markup=get_main_keyboard())
            return

        items_text = "\n".join(
            f"▸ {item['name']} | {item.get('flavor', '')} | {item['qty']}шт | {item['price'] * item['qty']}₽"
            for item in items
        )

        order_info = {
            'district': data.get('district', 'Не указан'),
            'address': data.get('address', 'Не указан'),
            'total': data.get('total', 0),
            'username': message.from_user.username or message.from_user.first_name
        }

        # Клиенту
        await message.answer(
            f"✅ *Заказ принят!*\n\n"
            f"{items_text}\n\n"
            f"📍 Район: {order_info['district']}\n"
            f"🏠 Адрес: {order_info['address']}\n"
            f"💳 Сумма: {order_info['total']}₽\n\n"
            f"Менеджер свяжется с вами в ближайшее время.",
            reply_markup=get_main_keyboard()
        )

        # Менеджеру
        await bot.send_message(
            chat_id=MANAGER_CHAT_ID,
            text=f"🛒 *Новый заказ!*\n\n"
                 f"{items_text}\n\n"
                 f"📍 {order_info['district']}\n"
                 f"🏠 {order_info['address']}\n"
                 f"👤 @{order_info['username']}\n"
                 f"💵 {order_info['total']}₽"
        )

    except Exception as e:
        logging.error(f"WebApp error: {e}")
        await message.answer(
            "⚠ Ошибка обработки заказа\nПопробуйте позже",
            reply_markup=get_main_keyboard()
        )

async def main():
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main()) 
```
