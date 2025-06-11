import logging
import random

from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, Message, CallbackQuery
from aiogram.utils.keyboard import ReplyKeyboardBuilder, InlineKeyboardBuilder
from aiogram import F

# === Настройки ===
API_TOKEN = '8172157318:AAE56QRizWhZFuA2hyGxWW-wY68CYaUaxJ4'  # ← Замени на свой токен

# === Логирование ===
logging.basicConfig(level=logging.INFO)

# === Бот и диспетчер ===
bot = Bot(token=API_TOKEN)
dp = Dispatcher()

# === Клавиатуры ===

# Главное меню
def get_main_menu_kb():
    kb = ReplyKeyboardBuilder()
    kb.row(
        types.KeyboardButton(text="🎮 Играть"),
        types.KeyboardButton(text="👥 Пригласить друга")
    )
    kb.row(
        types.KeyboardButton(text="👥 Список игроков")
    )
    kb.row(
        types.KeyboardButton(text="❓ Помощь")
    )
    return kb.as_markup(resize_keyboard=True)

# Меню выбора режима игры
game_mode_kb = InlineKeyboardMarkup(inline_keyboard=[
    [InlineKeyboardButton(text="🔍 Поиск соперника", callback_data="find_rps")],
    [InlineKeyboardButton(text="🤖 С ботом", callback_data="rps_bot")],
    [InlineKeyboardButton(text="🪙 Орёл или решка", callback_data="coin_flip")],
    [InlineKeyboardButton(text="🎰 Рулетка", callback_data="roulette_game")]
])

# Игровая клавиатура RPS
rps_keyboard = InlineKeyboardMarkup(inline_keyboard=[
    [InlineKeyboardButton(text="🪨", callback_data="rps_rock"),
     InlineKeyboardButton(text="✂️", callback_data="rps_scissors"),
     InlineKeyboardButton(text="📄", callback_data="rps_paper")]
])

# Игровая клавиатура — орёл или решка
coin_flip_keyboard = InlineKeyboardMarkup(inline_keyboard=[
    [InlineKeyboardButton(text="🦅 Орёл", callback_data="coin_heads"),
     InlineKeyboardButton(text="🪙 Решка", callback_data="coin_tails")]
])

# Клавиатура принятия приглашения
accept_kb = InlineKeyboardMarkup(inline_keyboard=[
    [InlineKeyboardButton(text="✅ Принять", callback_data="accept_invite"),
     InlineKeyboardButton(text="❌ Отклонить", callback_data="decline_invite")]
])

# Клавиатура для рулетки
roulette_keyboard = InlineKeyboardMarkup(inline_keyboard=[
    [InlineKeyboardButton(text="🔴 Красное", callback_data="roulette_red"),
     InlineKeyboardButton(text="⚫ Чёрное", callback_data="roulette_black")],
    [InlineKeyboardButton(text="🟣 Чётное", callback_data="roulette_even"),
     InlineKeyboardButton(text="🔵 Нечётное", callback_data="roulette_odd")],
    [InlineKeyboardButton(text="🔢 0", callback_data="roulette_0"),
     InlineKeyboardButton(text="🔢 1-18", callback_data="roulette_1to18"),
     InlineKeyboardButton(text="🔢 19-36", callback_data="roulette_19to36")],
    [InlineKeyboardButton(text="🎲 Случайное число", callback_data="roulette_spin")]
])

# === Глобальные переменные ===
users = {}  # {user_id: username}
pending_invites = {}  # {invite_id: {'from': user_id, 'to': target_id, 'game': game_type}}
games = {}  # {user_id: opponent_id}
game_data = {}  # {user_id: choice}

# === Команды и хэндлеры ===

@dp.message(Command("start"))
async def start(message: Message):
    user_id = message.from_user.id
    username = message.from_user.username or str(user_id)
    users[user_id] = username
    logging.info(f"[START] Пользователь @{username} ({user_id}) запустил бота.")
    await message.answer(f"Привет, {username}!", reply_markup=get_main_menu_kb())

@dp.message(Command("myid"))
async def show_my_id(message: Message):
    user_id = message.from_user.id
    username = message.from_user.username or str(user_id)
    await message.answer(f"Ваш ID: `{user_id}`\nВаш никнейм: @{username}")

@dp.message(Command("users"))
async def cmd_users(message: Message):
    if not users:
        await message.answer("Пока нет активных пользователей.")
        return

    user_list = "\n".join([f"@{username}" for uid, username in users.items()])
    await message.answer(f"Все пользователи, запустившие бота:\n\n{user_list}")

@dp.message(F.text == "👥 Список игроков")
async def show_users_list(message: Message):
    if not users:
        await message.answer("Пока нет активных пользователей.")
        return

    builder = InlineKeyboardBuilder()
    for uid, username in users.items():
        builder.add(InlineKeyboardButton(text=f"{username}", callback_data=f"user_info_{uid}"))
    builder.adjust(1)

    await message.answer("Список всех пользователей, запустивших бота:", reply_markup=builder.as_markup())

@dp.callback_query(F.data.startswith("user_info_"))
async def handle_user_selection(callback: CallbackQuery):
    selected_uid = int(callback.data.split("_")[2])
    selected_username = users.get(selected_uid, str(selected_uid))
    await callback.message.edit_text(f"Выбран пользователь: @{selected_username} (ID: {selected_uid})")

@dp.message(F.text == "❓ Помощь")
async def help_cmd(message: Message):
    await message.reply(
        "Используй:\n"
        "- Нажмите \"Играть\" чтобы начать\n"
        "- Нажмите \"Пригласить\" чтобы выбрать игрока из списка\n"
        "- Выберите игру и нажмите на имя пользователя"
    )

@dp.message(F.text == "🎮 Играть")
async def find_opponent_or_invite(message: Message):
    await message.answer("Выберите режим игры:", reply_markup=game_mode_kb)

@dp.callback_query(F.data == "find_rps")
async def find_opponent(callback: CallbackQuery):
    message = callback.message
    user_id = callback.from_user.id
    for opponent_id in games:
        if games[opponent_id] is None and opponent_id != user_id:
            games[opponent_id] = user_id
            games[user_id] = opponent_id
            await bot.send_message(opponent_id, "Найден соперник!", reply_markup=rps_keyboard)
            await message.edit_text("Найден соперник!", reply_markup=rps_keyboard)
            return

    games[user_id] = None
    await message.edit_text("Ищу соперника...")

@dp.message(F.text == "👥 Пригласить друга")
async def invite_friend(message: Message):
    user_id = message.from_user.id
    available_users = [u for u in users if u != user_id]

    if not available_users:
        await message.answer("Нет доступных пользователей для приглашения.")
        return

    builder = InlineKeyboardBuilder()
    for uid in available_users:
        builder.add(InlineKeyboardButton(text=f"{users[uid]}", callback_data=f"invite_select_{uid}_rps"))
    builder.adjust(1)

    await message.answer("Выберите игрока для приглашения:", reply_markup=builder.as_markup())

@dp.callback_query(F.data.startswith("invite_select_"))
async def process_selected_user(callback: CallbackQuery):
    data = callback.data.split("_")
    from_id = callback.from_user.id
    to_id = int(data[2])
    game_type = data[3]

    if to_id not in users:
        await callback.answer("Ошибка: пользователь не найден")
        return

    invite_id = f"{from_id}_{to_id}_{game_type}"
    pending_invites[invite_id] = {
        'from': from_id,
        'to': to_id,
        'game': game_type
    }

    from_name = callback.from_user.first_name or "Игрок"
    to_username = users[to_id]

    try:
        await bot.send_message(to_id,
                               f"{from_name} приглашает тебя в игру '{game_type}'",
                               reply_markup=accept_kb)
        await callback.message.edit_text(f"✅ Приглашение отправлено пользователю @{to_username}!")
    except Exception as e:
        logging.error(f"[INVITE] Не удалось отправить сообщение: {e}")
        await callback.message.edit_text("❌ Не удалось отправить приглашение.")

@dp.callback_query(F.data == "accept_invite")
async def accept_invite(callback: CallbackQuery):
    user_id = callback.from_user.id
    invite_id = next((k for k in pending_invites if pending_invites[k]['to'] == user_id), None)

    if not invite_id:
        await callback.answer("Приглашение не найдено.")
        return

    data = pending_invites.pop(invite_id)
    from_id = data['from']
    game_type = data['game']

    if game_type == 'rps':
        games[from_id] = user_id
        games[user_id] = from_id
        await bot.send_message(from_id, "Игрок принял приглашение!", reply_markup=rps_keyboard)
        await callback.message.edit_text("Вы приняли приглашение!", reply_markup=rps_keyboard)

@dp.callback_query(F.data == "rps_bot")
async def rps_vs_bot(callback: CallbackQuery):
    await callback.message.edit_text("Выберите свой ход против бота:", reply_markup=rps_keyboard)

@dp.callback_query(F.data.startswith("rps_"))
async def process_rps_choice_vs_bot(callback: CallbackQuery):
    user_choice = callback.data.split('_')[1]
    bot_choice = random.choice(['rock', 'scissors', 'paper'])

    result = determine_winner(user_choice, bot_choice)

    emoji = {'rock': '🪨', 'scissors': '✂️', 'paper': '📄'}
    await callback.message.edit_text(
        f"Вы выбрали: {emoji[user_choice]}\n"
        f"Бот выбрал: {emoji[bot_choice]}\n"
        f"Результат: {result}"
    )

@dp.callback_query(F.data == "coin_flip")
async def coin_flip(callback: CallbackQuery):
    await callback.message.edit_text("Выберите: Орёл или Решка?", reply_markup=coin_flip_keyboard)

@dp.callback_query(F.data.startswith("coin_"))
async def process_coin_flip(callback: CallbackQuery):
    user_choice = callback.data.split("_")[1]
    result = random.choice(['heads', 'tails'])

    if user_choice == result:
        msg = "🎉 Вы угадали!"
    else:
        msg = "😢 Не угадали."

    icons = {'heads': '🦅', 'tails': '🪙'}
    await callback.message.edit_text(
        f"Вы выбрали: {icons[user_choice]}\n"
        f"Выпало: {icons[result]}\n"
        f"{msg}"
    )

# === Рулетка ===

@dp.callback_query(F.data == "roulette_game")
async def roulette_menu(callback: CallbackQuery):
    await callback.message.edit_text("🎰 Добро пожаловать в рулетку!\nВыберите, на что вы хотите сделать ставку:", reply_markup=roulette_keyboard)

@dp.callback_query(F.data.startswith("roulette_"))
async def process_roulette(callback: CallbackQuery):
    bet_type = callback.data.split("_")[1]
    number = random.randint(0, 36)
    color = "red" if number in red_numbers else "black" if number in black_numbers else "green"

    result_text = f"Выпало число: {number}\nЦвет: {'🔴 Красное' if color == 'red' else '⚫ Чёрное' if color == 'black' else '🟢 Зелёное'}\n"

    if bet_type == "spin":
        await callback.message.edit_text(result_text)
        return

    if bet_type == "red" and color == "red" and number != 0:
        result_text += "Вы победили! 🔴 Красное"
    elif bet_type == "black" and color == "black" and number != 0:
        result_text += "Вы победили! ⚫ Чёрное"
    elif bet_type == "even" and number % 2 == 0 and number != 0:
        result_text += "Вы победили! 🟣 Чётное"
    elif bet_type == "odd" and number % 2 == 1:
        result_text += "Вы победили! 🔵 Нечётное"
    elif bet_type == "0" and number == 0:
        result_text += "Вы победили! 🟡 Выпало 0"
    elif bet_type.startswith("1to18") and 1 <= number <= 18:
        result_text += "Вы победили! 🔢 Число от 1 до 18"
    elif bet_type.startswith("19to36") and 19 <= number <= 36:
        result_text += "Вы победили! 🔢 Число от 19 до 36"
    else:
        result_text += "😢 К сожалению, вы проиграли."

    await callback.message.edit_text(result_text)

# Список красных и чёрных чисел
red_numbers = [1, 3, 5, 7, 9, 12, 14, 16, 18, 19, 21, 23, 25, 27, 30, 32, 34, 36]
black_numbers = [2, 4, 6, 8, 10, 11, 13, 15, 17, 20, 22, 24, 26, 28, 29, 31, 33, 35]

# === Обработка обычного RPS ===
@dp.callback_query(F.data.startswith("rps_"))
async def process_rps_choice(callback: CallbackQuery):
    user_id = callback.from_user.id
    choice = callback.data.split('_')[1]

    if user_id not in games:
        await callback.answer("Ты не в игре!")
        return

    opponent_id = games[user_id]
    if opponent_id is None:
        await callback.answer("Ждём соперника...")
        return

    game_data[user_id] = choice

    if opponent_id in game_data:
        p1 = game_data.pop(user_id)
        p2 = game_data.pop(opponent_id)
        result = determine_winner(p1, p2)

        await bot.send_message(user_id, f"Твой ход: {p1}\nХод соперника: {p2}\nРезультат: {result}")
        await bot.send_message(opponent_id, f"Твой ход: {p2}\nХод соперника: {p1}\nРезультат: {result}")

        del games[user_id]
        del games[opponent_id]
    else:
        await callback.message.edit_text("Ожидаем хода соперника...")

def determine_winner(p1, p2):
    if p1 == p2:
        return "Ничья!"
    wins = {
        'rock': 'scissors',
        'scissors': 'paper',
        'paper': 'rock'
    }
    if wins[p1] == p2:
        return "Ты победил!"
    else:
        return "Соперник победил!"

# === Запуск бота ===
if __name__ == '__main__':
    logging.info("Запуск Telegram-бота...")
    dp.run_polling(bot)
