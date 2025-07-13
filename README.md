```python
import json
import logging
import asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import WebAppInfo, ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove
from aiogram.client.default import DefaultBotProperties
from aiogram.filters import Command
from aiogram import F

# Настройка логирования
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Конфигурация
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
        "<b>График работы</b>\n\n"
        "🕐 Ежедневно с 13:00 до 23:00\n\n"
        "🚚 Доставка работает без выходных\n"
        "🚗 Самовывоз по договорённости",
        reply_markup=get_main_keyboard()
    )

@dp.message(F.web_app_data)
async def handle_webapp_data(message: types.Message):
    try:
        logger.info(f"Получены данные из WebApp: {message.web_app_data.data}")
        
        # Закрываем клавиатуру
        await message.answer("🔄 Обрабатываем ваш заказ...", reply_markup=ReplyKeyboardRemove())
        
        # Парсим JSON данные
        try:
            data = json.loads(message.web_app_data.data)
            logger.info(f"Данные заказа: {data}")
        except json.JSONDecodeError as e:
            logger.error(f"Ошибка парсинга JSON: {e}")
            await message.answer("❌ Ошибка обработки данных заказа", reply_markup=get_main_keyboard())
            return

        # Проверяем наличие товаров
        items = data.get('items', [])
        if not items:
            logger.warning("Получена пустая корзина")
            await message.answer("❌ Ваша корзина пуста", reply_markup=get_main_keyboard())
            return

        # Формируем детали заказа
        order_details = []
        total = 0.0
        
        for item in items:
            try:
                name = item.get('name', 'Без названия')
                price = float(item['price'])
                qty = int(item['qty'])
                flavor = item.get('flavor', 'не указан')
                
                item_total = price * qty
                total += item_total
                
                order_details.append(
                    f"▫️ {name}\n"
                    f"   - Вкус: {flavor}\n"
                    f"   - Кол-во: {qty}\n"
                    f"   - Цена: {item_total:.2f}₽\n"
                )
            except (KeyError, ValueError) as e:
                logger.error(f"Ошибка обработки товара: {e}")
                continue

        if not order_details:
            await message.answer("❌ Некорректные данные товаров", reply_markup=get_main_keyboard())
            return

        # Получаем данные доставки
        district = data.get('district', 'не указан')
        address = data.get('address', 'не указан')
        username = message.from_user.username or f"id{message.from_user.id}"

        # Формируем сообщение для клиента
        client_message = (
            "✅ <b>Ваш заказ принят!</b>\n\n"
            f"{''.join(order_details)}\n"
            f"<b>Итого:</b> {total:.2f}₽\n"
            f"<b>Район:</b> {district}\n"
            f"<b>Адрес:</b> {address}\n\n"
            "📞 Менеджер свяжется с вами для подтверждения."
        )

        # Формируем сообщение для менеджера
        manager_message = (
            "🛒 <b>НОВЫЙ ЗАКАЗ!</b>\n\n"
            f"{''.join(order_details)}\n"
            f"<b>Сумма:</b> {total:.2f}₽\n"
            f"<b>Район:</b> {district}\n"
            f"<b>Адрес:</b> {address}\n\n"
            f"👤 <b>Клиент:</b> @{username}\n"
            f"🆔 <b>ID:</b> {message.from_user.id}"
        )

        # Отправляем сообщения
        await message.answer(client_message, reply_markup=get_main_keyboard())
        await bot.send_message(MANAGER_CHAT_ID, manager_message)

        logger.info(f"Заказ успешно обработан для пользователя {username}")

    except Exception as e:
        logger.error(f"Критическая ошибка: {e}", exc_info=True)
        await message.answer(
            "⚠️ Произошла ошибка при обработке заказа\n"
            "Пожалуйста, попробуйте позже или свяжитесь с менеджером",
            reply_markup=get_main_keyboard()
        )

async def main():
    logger.info("Бот запускается...")
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())
```
