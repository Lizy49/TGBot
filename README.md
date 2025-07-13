```python
import json
import logging
import asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    WebAppInfo, 
    ReplyKeyboardMarkup, 
    KeyboardButton,
    ReplyKeyboardRemove
)
from aiogram.client.default import DefaultBotProperties
from aiogram.filters import Command
from aiogram import F

# Настройки
API_TOKEN = '7690796647:AAHibbEzg3ky14fCNpJM2-_G7m4F_kSlqKI'
MANAGER_CHAT_ID = 906177509
WEBAPP_URL = 'https://tlpshop.ru'

# Инициализация бота
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

@dp.message(F.text == "📱 Навигация")
async def show_contacts(message: types.Message):
    await message.answer(
        "<b>Наши контакты:</b>\n\n"
        "👨‍💼 Менеджер: @tlp_shop\n"
        "📢 Канал: t.me/tlpshopmgdn\n"
        "📝 Отзывы: t.me/+VfwmutOo8R9hZWMy\n\n"
        "⏳ Время ответа: до 15 минут",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.text == "⏱ Часы работы")
async def working_hours(message: types.Message):
    await message.answer(
        "<b>⏳*График работы*⏳</b>\n\n"
        "🕐 Ежедневно с 13:00 до 23:00\n\n"
        "🚚 Доставка работает без выходных\n"
        "🚗 Самовывоз по договорённости",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.web_app_data)
async def handle_webapp_data(message: types.Message):
    try:
        # Закрываем клавиатуру
        await message.answer(
            "Обрабатываем ваш заказ...",
            reply_markup=ReplyKeyboardRemove()
        )
        
        # Парсим данные из WebApp
        data = json.loads(message.web_app_data.data)
        
        # Формируем информацию о заказе
        items = data.get('items', [])

```
