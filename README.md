[from aiogram import Bot, Dispatchertxt.txt](https://github.com/user-attachments/files/19095202/from.aiogram.import.Bot.Dispatchertxt.txt)# [Uploadinfrom aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup
from aiogram.utils.keyboard import InlineKeyboardBuilder
from aiogram.enums import ParseMode
from aiogram.utils.markdown import hbold
import random
import logging
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup

# Настройка логирования
logging.basicConfig(level=logging.INFO)

# Токен вашего бота
BOT_TOKEN = 7708712415:AAG64R9V3lxFRSwilBCgGKcFdDN9NX-c-jw

# ID админа
ADMIN_ID = 6492167030  # Указанный ID админа

# Ссылка на поддержку
SUPPORT_LINK = "https://t.me/Holdsuppp"

# Канал для публикации результатов и проверки подписки
CHANNEL_ID = "@casinoholda"  # Замените на ID вашего канала

# Реквизиты для оплаты
CRYPTO_PAYMENT_LINK = "https://t.me/send?start=IVOMFgECv6vy"  # Оплата криптовалютой
CARD_PAYMENT_DETAILS = "2204 3107 7671 9943"  # Оплата картой

# Минимальные суммы для пополнения (в долларах)
MIN_DEPOSIT_CRYPTO = 1.0  # Минимальная сумма для криптовалюты
MIN_DEPOSIT_CARD = 1.11  # Минимальная сумма для карты

# Инициализация бота и диспетчера
bot = Bot(token=BOT_TOKEN, parse_mode=ParseMode.HTML)
dp = Dispatcher()

# База данных для хранения баланса, статистики и реквизитов
users = {}
requisites = {
    "card": CARD_PAYMENT_DETAILS,  # Реквизиты карты
    "cryptobot": CRYPTO_PAYMENT_LINK,  # Реквизиты Cryptobot
}
withdraw_requests = []  # Список заявок на вывод

# Минимальная сумма для вывода (в долларах)
MIN_WITHDRAWAL = 5

# Коэффициенты и шансы для рулетки
ROULETTE_OPTIONS = {
    "red": {"coefficient": 1.75, "chance": 0.40},  # Красный
    "blue": {"coefficient": 1.95, "chance": 0.40},  # Синий
    "green": {"coefficient": 25, "chance": 0.03},  # Зеленый
}

# Клавиатура для главного меню
def main_menu():
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="🎲 Dice", callback_data="dice"),
        InlineKeyboardButton(text="🎡 Рулетка", callback_data="roulette"),
    )
    builder.row(InlineKeyboardButton(text="👤 Личный кабинет", callback_data="account"))
    builder.row(InlineKeyboardButton(text="🆘 Поддержка", url=SUPPORT_LINK))  # Кнопка поддержки
    return builder.as_markup()

# Клавиатура для выбора цвета в рулетке
def roulette_menu():
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="🔴 Красный (x1.75)", callback_data="roulette_red"),
        InlineKeyboardButton(text="🔵 Синий (x1.95)", callback_data="roulette_blue"),
    )
    builder.row(InlineKeyboardButton(text="🟢 Зеленый (x25)", callback_data="roulette_green"))
    return builder.as_markup()

# Клавиатура для админ-панели
def admin_menu():
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="Реквизиты", callback_data="edit_requisites"),
        InlineKeyboardButton(text="Просмотреть заявки на вывод", callback_data="view_withdraw_requests"),
    )
    builder.row(InlineKeyboardButton(text="Настроить канал", callback_data="set_channel"))
    return builder.as_markup()

# Клавиатура для выбора способа пополнения
def deposit_menu():
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="Карта", callback_data="deposit_card"),
        InlineKeyboardButton(text="Cryptobot", callback_data="deposit_cryptobot"),
    )
    return builder.as_markup()

# Клавиатура для проверки оплаты
def check_payment_menu():
    builder = InlineKeyboardBuilder()
    builder.row(InlineKeyboardButton(text="Проверить оплату", callback_data="check_payment"))
    return builder.as_markup()

# Проверка подписки на канал
async def check_subscription(user_id: int) -> bool:
    try:
        chat_member = await bot.get_chat_member(CHANNEL_ID, user_id)
        return chat_member.status in ["member", "administrator", "creator"]
    except Exception as e:
        logging.error(f"Ошибка при проверке подписки: {e}")
        return False

# Команда /start
@dp.message(Command("start"))
async def start(message: types.Message):
    user_id = message.from_user.id

    # Проверка подписки на канал
    if not await check_subscription(user_id):
        await message.answer(
            f"📢 Для использования бота необходимо подписаться на канал {CHANNEL_ID}.\n\n"
            "После подписки нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )
        return

    await message.answer(
        "Добро пожаловать в казино! Выберите игру:",
        reply_markup=main_menu(),
    )

# Обработка кнопки "Проверить подписку"
@dp.callback_query(F.data == "check_subscription")
async def check_subscription_handler(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    if await check_subscription(user_id):
        await callback.message.edit_text(
            "Спасибо за подписку! Теперь вы можете использовать бота.",
            reply_markup=main_menu(),
        )
    else:
        await callback.message.edit_text(
            f"📢 Вы не подписаны на канал {CHANNEL_ID}. Пожалуйста, подпишитесь и нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )

# Команда /admin (только для админа)
@dp.message(Command("admin"))
async def admin_panel(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        await message.answer(
            "Админ-панель:",
            reply_markup=admin_menu(),
        )
    else:
        await message.answer("У вас нет доступа к этой команде.")

# Обработка нажатий на кнопки
@dp.callback_query(F.data == "dice")
async def dice_handler(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    # Проверка подписки на канал
    if not await check_subscription(user_id):
        await callback.message.edit_text(
            f"📢 Для использования бота необходимо подписаться на канал {CHANNEL_ID}.\n\n"
            "После подписки нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )
        return

    if user_id not in users or users[user_id].get("balance", 0) < 1:  # Минимальная ставка 1$
        await callback.message.edit_text(
            "Для игры в Dice необходим баланс не менее 1$. Пополните баланс.",
            reply_markup=main_menu(),
        )
    else:
        await callback.message.edit_text(
            "Вы выбрали игру 🎲 Dice. Выберите чет или нечет:",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Чет", callback_data="dice_even")],
                [InlineKeyboardButton(text="Нечет", callback_data="dice_odd")],
            ]),
        )

@dp.callback_query(F.data == "roulette")
async def roulette_handler(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    # Проверка подписки на канал
    if not await check_subscription(user_id):
        await callback.message.edit_text(
            f"📢 Для использования бота необходимо подписаться на канал {CHANNEL_ID}.\n\n"
            "После подписки нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )
        return

    if user_id not in users or users[user_id].get("balance", 0) < 1:  # Минимальная ставка 1$
        await callback.message.edit_text(
            "Для игры в Рулетку необходим баланс не менее 1$. Пополните баланс.",
            reply_markup=main_menu(),
        )
    else:
        await callback.message.edit_text(
            "Вы выбрали игру 🎡 Рулетка. Выберите цвет:",
            reply_markup=roulette_menu(),
        )

@dp.callback_query(F.data == "account")
async def account_handler(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    # Проверка подписки на канал
    if not await check_subscription(user_id):
        await callback.message.edit_text(
            f"📢 Для использования бота необходимо подписаться на канал {CHANNEL_ID}.\n\n"
            "После подписки нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )
        return

    balance = users.get(user_id, {}).get("balance", 0)
    await callback.message.edit_text(
        f"👤 Личный кабинет:\n"
        f"Ваш ID: {hbold(user_id)}\n"
        f"Баланс: {hbold(balance)}$\n\n"
        "Выберите действие:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="Пополнить баланс", callback_data="deposit")],
            [InlineKeyboardButton(text="Вывести средства", callback_data="withdraw")],
        ]),
    )

# Логика игры Dice
@dp.callback_query(F.data.startswith("dice_"))
async def dice_game(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    # Проверка подписки на канал
    if not await check_subscription(user_id):
        await callback.message.edit_text(
            f"📢 Для использования бота необходимо подписаться на канал {CHANNEL_ID}.\n\n"
            "После подписки нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )
        return

    choice = callback.data.split("_")[1]  # even или odd
    dice_roll = random.randint(1, 6)
    result = "even" if dice_roll % 2 == 0 else "odd"

    if user_id not in users:
        users[user_id] = {"balance": 0}  # Начальный баланс 0

    if choice == result:
        users[user_id]["balance"] += 1 * 1.85  # Ставка 1$, коэффициент 1.85
        result_text = f"🎲 Выпало: {dice_roll} ({result})\nВы выиграли! Ваш баланс: {users[user_id]['balance']}$"
    else:
        users[user_id]["balance"] -= 1  # Ставка 1$
        result_text = f"🎲 Выпало: {dice_roll} ({result})\nВы проиграли. Ваш баланс: {users[user_id]['balance']}$"

    await callback.message.edit_text(result_text, reply_markup=main_menu())

    # Публикация результата в канале
    await bot.send_message(
        CHANNEL_ID,
        f"Результат игры 🎲 Dice:\n{result_text}",
    )

# Логика игры Рулетка
@dp.callback_query(F.data.startswith("roulette_"))
async def roulette_game(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    # Проверка подписки на канал
    if not await check_subscription(user_id):
        await callback.message.edit_text(
            f"📢 Для использования бота необходимо подписаться на канал {CHANNEL_ID}.\n\n"
            "После подписки нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )
        return

    if user_id not in users:
        users[user_id] = {"balance": 0}  # Начальный баланс 0

    color = callback.data.split("_")[1]  # red, blue или green
    option = ROULETTE_OPTIONS[color]

    # Генерация результата
    if random.random() < option["chance"]:  # Проверка на выигрыш
        users[user_id]["balance"] += 1 * option["coefficient"]  # Ставка 1$, умноженная на коэффициент
        result_text = f"🎡 Выпал {color.capitalize()}! Вы выиграли! Ваш баланс: {users[user_id]['balance']}$"
    else:
        users[user_id]["balance"] -= 1  # Ставка 1$
        result_text = f"🎡 Выпал другой цвет. Вы проиграли. Ваш баланс: {users[user_id]['balance']}$"

    await callback.message.edit_text(result_text, reply_markup=main_menu())

    # Публикация результата в канале
    await bot.send_message(
        CHANNEL_ID,
        f"Результат игры 🎡 Рулетка:\n{result_text}",
    )

# Обработка заявок на вывод
@dp.callback_query(F.data == "withdraw")
async def withdraw_request(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    # Проверка подписки на канал
    if not await check_subscription(user_id):
        await callback.message.edit_text(
            f"📢 Для использования бота необходимо подписаться на канал {CHANNEL_ID}.\n\n"
            "После подписки нажмите кнопку ниже, чтобы проверить подписку.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="Проверить подписку", callback_data="check_subscription")]
            ]),
        )
        return

    balance = users.get(user_id, {}).get("balance", 0)

    if balance < MIN_WITHDRAWAL:
        await callback.message.edit_text(
            f"Минимальная сумма для вывода: {MIN_WITHDRAWAL}$.",
            reply_markup=main_menu(),
        )
    else:
        withdraw_requests.append({"user_id": user_id, "amount": balance})
        users[user_id]["balance"] = 0  # Обнуляем баланс после заявки

        # Уведомление админа
        await bot.send_message(
            ADMIN_ID,
            f"Новая заявка на вывод:\n"
            f"ID пользователя: {user_id}\n"
            f"Сумма: {balance}$",
        )

        await callback.message.edit_text(
            "Ваша заявка на вывод отправлена. Ожидайте обработки.",
            reply_markup=main_menu(),
        )

# Админ: Просмотр заявок на вывод
@dp.callback_query(F.data == "view_withdraw_requests")
async def view_withdraw_requests(callback: types.CallbackQuery):
    if callback.from_user.id == ADMIN_ID:
        if not withdraw_requests:
            await callback.message.edit_text("Заявок на вывод нет.")
        else:
            requests_text = "\n".join(
                f"ID: {req['user_id']}, Сумма: {req['amount']}$"
                for req in withdraw_requests
            )
            await callback.message.edit_text(
                f"Заявки на вывод:\n{requests_text}",
                reply_markup=admin_menu(),
            )
    else:
        await callback.message.edit_text("У вас нет доg from aiogram import Bot, Dispatchertxt.txt…]()
