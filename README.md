```python
import json
import logging
from aiogram import Bot, Dispatcher, types
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import WebAppInfo, ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove
from aiogram.client.default import DefaultBotProperties
from aiogram.filters import Command
from aiogram import F

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

API_TOKEN = '7690796647:AAHibbEzg3ky14fCNpJM2-_G7m4F_kSlqKI'
MANAGER_CHAT_ID = 906177509  # Убедись, что бот добавлен в этот чат!
WEBAPP_URL = 'https://tlpshop.ru'  # Должен быть HTTPS!

bot = Bot(token=API_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher(storage=MemoryStorage())

def get_main_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="🛍️ Оформить заказ", web_app=WebAppInfo(url=WEBAPP_URL))],
            [KeyboardButton(text="📱 Навигация"), KeyboardButton(text="⏱ Часы работы")]
        ],
        resize_keyboard=True
    )

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    await message.answer(
        "<b>TLP | SHOP</b>\n\n"
        "Добро пожаловать в наш магазин!\n\n"
        "Используйте кнопки ниже для навигации:",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.web_app_data)
async def handle_webapp_data(message: types.Message):
    try:
        logger.info(f"Получены данные: {message.web_app_data.data}")
        
        data = json.loads(message.web_app_data.data)
        
        # Проверяем, что есть товары
        if not data.get('items'):
            await message.answer("❌ Корзина пуста!", reply_markup=get_main_keyboard())
            return

        # Формируем сообщение для клиента
        order_text = "✅ <b>Ваш заказ принят!</b>\n\n"
        for item in data['items']:
            order_text += (
                f"▫️ <b>{item.get('name', 'Без названия')}</b>\n"
                f"   - Вкус: {item.get('flavor', 'не указан')}\n"
                f"   - Кол-во: {item.get('qty', 1)}\n"
                f"   - Цена: {float(item.get('price', 0)) * int(item.get('qty', 1)):.2f}₽\n\n"
            )
        
        order_text += (
            f"📍 <b>Район:</b> {data.get('district', 'не указан')}\n"
            f"🏠 <b>Адрес:</b> {data.get('address', 'не указан')}\n"
            f"💵 <b>Итого:</b> {float(data.get('total', 0)):.2f}₽\n\n"
            "📞 Менеджер свяжется с вами в течение 15 минут."
        )

        # Формируем сообщение для менеджера
        manager_text = (
            "🛒 <b>НОВЫЙ ЗАКАЗ!</b>\n\n"
            f"{order_text}\n"
            f"👤 <b>Клиент:</b> @{message.from_user.username or 'нет username'}\n"
            f"🆔 <b>ID:</b> {message.from_user.id}\n"
            f"📱 <b>Телефон:</b> {data.get('phone', 'не указан')}"
        )

        # Отправляем клиенту
        await message.answer(order_text, reply_markup=get_main_keyboard())
        
        # Отправляем менеджеру
        await bot.send_message(
            chat_id=MANAGER_CHAT_ID,
            text=manager_text,
            parse_mode=ParseMode.HTML
        )

    except Exception as e:
        logger.error(f"Ошибка: {e}")
        await message.answer(
            "❌ Ошибка при обработке заказа! Свяжитесь с менеджером.",
            reply_markup=get_main_keyboard()
        )

if __name__ == '__main__':
    from aiogram import executor
    executor.start_polling(dp, skip_updates=True)
```
