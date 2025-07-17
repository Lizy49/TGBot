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
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
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
        logger.info("Получены данные из WebApp")
        
        # Закрываем клавиатуру
        await message.answer("🔄 Обрабатываем ваш заказ...", reply_markup=ReplyKeyboardRemove())
        
        # Парсим JSON данные
        try:
            data = json.loads(message.web_app_data.data)
            logger.info(f"Данные заказа: {json.dumps(data, indent=2, ensure_ascii=False)}")
        except json.JSONDecodeError as e:
            logger.error(f"Ошибка парсинга JSON: {e}")
            await message.answer("❌ Ошибка обработки данных заказа", reply_markup=get_main_keyboard())
            return

        # Проверяем обязательные поля
        required_fields = ['items', 'address', 'district', 'total']
        for field in required_fields:
            if field not in data:
                logger.error(f"Отсутствует обязательное поле: {field}")
                await message.answer(f"❌ В данных заказа отсутствует {field}", reply_markup=get_main_keyboard())
                return

        # Проверяем наличие товаров
        if not isinstance(data['items'], list) or len(data['items']) == 0:
            logger.warning("Получена пустая корзина")
            await message.answer("❌ Ваша корзина пуста", reply_markup=get_main_keyboard())
            return

        # Формируем детали заказа
        order_details = []
        for item in data['items']:
            try:
                order_details.append(
                    f"▫️ {item.get('name', 'Без названия')}\n"
                    f"   - Вкус: {item.get('flavor', 'не указан')}\n"
                    f"   - Кол-во: {item.get('qty', 1)}\n"
                    f"   - Цена: {float(item.get('price', 0)) * int(item.get('qty', 1)):.2f}₽\n"
                )
            except (KeyError, ValueError) as e:
                logger.error(f"Ошибка обработки товара: {e}")
                continue

        if not order_details:
            await message.answer("❌ Некорректные данные товаров", reply_markup=get_main_keyboard())
            return

        # Формируем сообщение для клиента
        client_message = (
            "✅ <b>Ваш заказ принят!</b>\n\n"
            f"{''.join(order_details)}\n"
            f"<b>Район:</b> {data['district']}\n"
            f"<b>Адрес:</b> {data['address']}\n"
            f"<b>Итого:</b> {float(data['total']):.2f}₽\n\n"
            "📞 Менеджер свяжется с вами для подтверждения."
        )

        # Формируем сообщение для менеджера
        manager_message = (
            "🛒 <b>НОВЫЙ ЗАКАЗ!</b>\n\n"
            f"{''.join(order_details)}\n"
            f"<b>Район:</b> {data['district']}\n"
            f"<b>Адрес:</b> {data['address']}\n"
            f"<b>Итого:</b> {float(data['total']):.2f}₽\n\n"
            f"👤 <b>Клиент:</b> @{message.from_user.username or 'нет username'}\n"
            f"🆔 <b>ID:</b> {message.from_user.id}\n"
            f"📱 <b>Телефон:</b> {data.get('phone', 'не указан')}"
        )

        # Отправляем сообщения
        await message.answer(client_message, reply_markup=get_main_keyboard())
        await bot.send_message(
            chat_id=MANAGER_CHAT_ID,
            text=manager_message,
            parse_mode=ParseMode.HTML
        )

        logger.info(f"Заказ успешно обработан для пользователя {message.from_user.id}")

    except Exception as e:
        logger.error(f"Критическая ошибка: {str(e)}", exc_info=True)
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
