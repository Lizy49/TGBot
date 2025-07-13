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
MANAGER_CHAT_ID = 6438939468  # Замените на ваш ID чата менеджера

bot = Bot(token=API_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.MARKDOWN))
dp = Dispatcher(storage=MemoryStorage())

def get_main_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="🛍️ Оформить заказ", web_app=WebAppInfo(url="https://tlpshop.ru/"))],
            [KeyboardButton(text="📱 Навигация"), KeyboardButton(text="⏰ Режим работы")]
        ],
        resize_keyboard=True
    )

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    username = message.from_user.username or message.from_user.first_name
    await message.answer(
        f"🌟 *Добро пожаловать в TLP | SHOP, {username}!* 🌟\n\n"
        "💎 Премиальные товары с быстрой доставкой\n"
        "🚀 Лучшие цены в вашем районе\n"
        "🔒 Гарантия качества и конфиденциальности\n\n"
        "Нажмите кнопку ниже, чтобы начать покупки:",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.text == "📱 Навигация")
async def contacts(message: types.Message):
    await message.answer(
        "📱 *КОНТАКТЫ TLP | SHOP* 📱\n\n"
        "🔹 Менеджер: @tlp_shop\n"
        "🔹 Важная информация: https://t.me/tlpshopmgdn\n"
        "🔹 Отзывы: https://t.me/+VfwmutOo8R9hZWMy\n\n"
        "⏳ *Время ответа: до 15 минут*\n"
        "Мы всегда рады помочь вам с выбором!",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.text == "⏰ Режим работы")
async def working_hours(message: types.Message):
    await message.answer(
        "⏰ *РЕЖИМ РАБОТЫ* ⏰\n\n"
        "▫️Ежедневно: с 13:00 до 23:00\n\n"
        "🚚 *Самовывоз уточнять у менеджера*\n"
        "🚚 *Доставка без выходных*\n"
        "Мы работаем для вас каждый день!",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.web_app_data)
async def handle_webapp_data(message: types.Message):
    try:
        data = json.loads(message.web_app_data.data)
        items = data.get('items', [])

        if not items:
            await message.answer("❗ Ваша корзина пуста. Пожалуйста, добавьте товары перед оформлением заказа.")
            return

        items_text = "\n".join(
            f"▫️ {item['name']} | Вкус: *{item.get('flavor', 'не указан')}* | Кол-во: {item['qty']} | Сумма: {item['price'] * item['qty']}₽"
            for item in items
        )

        address = data.get('address', 'Не указан')
        district = data.get('district', 'Не указан')
        total = data.get('total', 0)
        username = message.from_user.username or message.from_user.first_name

        # Сообщение клиенту
        await message.answer(
            f"✅ *Ваш заказ принят!* ✅\n\n"
            f"{items_text}\n\n"
            f"📍 Район: {district}\n"
            f"🏠 Адрес: {address}\n"
            f"💰 Итого: *{total} ₽*\n\n"
            f"Наш менеджер @tlpmanager свяжется с вами для подтверждения заказа.\n"
            f"Спасибо за выбор TLP | SHOP!",
            reply_markup=get_main_keyboard()
        )

        # Сообщение менеджеру
        await bot.send_message(
            chat_id=MANAGER_CHAT_ID,
            text=(
                f"🛍️ *НОВЫЙ ЗАКАЗ!* 🛍️\n\n"
                f"{items_text}\n\n"
                f"📍 Район: {district}\n"
                f"🏠 Адрес: {address}\n"
                f"💰 Сумма: *{total} ₽*\n"
                f"👤 От: @{username}\n\n"
                f"Ожидает обработки"
            )
        )

    except Exception as e:
        logging.exception("Ошибка при обработке WebAppData")
        await message.answer(
            "⚠ Произошла ошибка при обработке вашего заказа.\n"
            "Пожалуйста, попробуйте ещё раз или свяжитесь с нами в @tlpsupport",
            reply_markup=get_main_keyboard()
        )

async def main():
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())
```
