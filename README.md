
from typing import Dict, Any
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup, KeyboardButton, Update, Contact

# Bot token
TOKEN = "7961055236:AAHoGcIwo91sHc12cV0VSCaWobYoRTgOrMc"

# User states and data
user_cart: Dict[int, Dict[str, int]] = {}

# Menu items
MENU_ITEMS = {
    'wings': {
        'name': '🍗 Куриные крылышки',
        'items': {
            'wings_6': {'name': '6 крылышек', 'price': 35000},
            'wings_12': {'name': '12 крылышек', 'price': 65000},
            'wings_24': {'name': '24 крылышка', 'price': 120000}
        }
    },
    'burgers': {
        'name': '🍔 Бургеры',
        'items': {
            'twister': {'name': 'Твистер', 'price': 28000},
            'zinger': {'name': 'Зингер', 'price': 32000},
            'big_box': {'name': 'Биг Бокс', 'price': 45000}
        }
    },
    'drinks': {
        'name': '🥤 Напитки',
        'items': {
            'cola': {'name': 'Кола', 'price': 8000},
            'fanta': {'name': 'Фанта', 'price': 8000},
            'sprite': {'name': 'Спрайт', 'price': 8000}
        }
    }
}

def get_main_menu() -> ReplyKeyboardMarkup:
    """Create the main menu keyboard."""
    keyboard = [
        [KeyboardButton("🍗 Меню"), KeyboardButton("🛒 Корзина")],
        [KeyboardButton("📱 Контакты"), KeyboardButton("⚙️ Настройки")]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Start the bot and show the welcome message."""
    welcome_text = """
🍗 Добро пожаловать в KFC Uzbekistan Delivery!

Мы рады приветствовать вас в нашем боте доставки. Здесь вы можете:
• Заказать любимые блюда KFC
• Отслеживать статус заказа
• Узнать о наших акциях

Выберите действие из меню ниже:
    """
    await update.message.reply_text(welcome_text, reply_markup=get_main_menu())

async def show_menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Display the menu categories."""
    menu_text = "🍗 Наше меню:\n\n"
    keyboard = []

    for category_id, category in MENU_ITEMS.items():
        menu_text += f"{category['name']}\n"
        keyboard.append([InlineKeyboardButton(category['name'], callback_data=f"category_{category_id}")])

    keyboard.append([InlineKeyboardButton("◀️ Назад", callback_data='back')])
    reply_markup = InlineKeyboardMarkup(keyboard)

    if update.callback_query:
        await update.callback_query.edit_message_text(menu_text, reply_markup=reply_markup)
    else:
        await update.message.reply_text(menu_text, reply_markup=reply_markup)

async def show_category(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Show items in the selected category."""
    query = update.callback_query
    category_id = query.data.split('_')[1]

    if category_id not in MENU_ITEMS:
        await query.answer("Ошибка: категория не найдена!")
        return

    category = MENU_ITEMS[category_id]
    menu_text = f"{category['name']}:\n\n"
    keyboard = []

    for item_id, item in category['items'].items():
        menu_text += f"{item['name']} - {item['price']} сум\n"
        keyboard.append([
            InlineKeyboardButton(f"➕ {item['name']}", callback_data=f"add_{item_id}"),
            InlineKeyboardButton(f"➖ {item['name']}", callback_data=f"remove_{item_id}")
        ])

    keyboard.append([InlineKeyboardButton("◀️ Назад", callback_data='menu')])
    reply_markup = InlineKeyboardMarkup(keyboard)
    await query.edit_message_text(menu_text, reply_markup=reply_markup)

async def add_to_cart(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Add an item to the user's cart."""
    query = update.callback_query
    user_id = query.from_user.id
    item_id = query.data.split('_')[1]

    if user_id not in user_cart:
        user_cart[user_id] = {}

    user_cart[user_id][item_id] = user_cart[user_id].get(item_id, 0) + 1
    await query.answer("Товар добавлен в корзину!")


async def remove_from_cart(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Remove an item from the user's cart."""
    query = update.callback_query
    user_id = query.from_user.id
    item_id = query.data.split('_')[1]

    if user_id in user_cart and item_id in user_cart[user_id]:
        user_cart[user_id][item_id] -= 1
        if user_cart[user_id][item_id] <= 0:
            del user_cart[user_id][item_id]
        await query.answer("Товар удален из корзины!")
    else:
        await query.answer("Товар не найден в корзине!")

async def show_cart(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Display the user's cart."""
    user_id = update.effective_user.id
    cart_text = "🛒 Ваша корзина:\n\n"
    total = 0

    if user_id in user_cart and user_cart[user_id]:
        for item_id, quantity in user_cart[user_id].items():
            for category in MENU_ITEMS.values():
                if item_id in category['items']:
                    item = category['items'][item_id]
                    cart_text += f"{item['name']} x{quantity} - {item['price'] * quantity} сум\n"
                    total += item['price'] * quantity
                    break

        cart_text += f"\n💰 Итого: {total} сум"
        keyboard = [
            [InlineKeyboardButton("✅ Оформить заказ", callback_data='checkout')],
            [InlineKeyboardButton("❌ Очистить корзину", callback_data='clear_cart')],
            [InlineKeyboardButton("◀️ Назад", callback_data='back')]
        ]
    else:
        cart_text = "🛒 Ваша корзина пуста"
        keyboard = [[InlineKeyboardButton("◀️ Назад", callback_data='back')]]

    reply_markup = InlineKeyboardMarkup(keyboard)

    if update.callback_query:
        await update.callback_query.edit_message_text(cart_text, reply_markup=reply_markup)
    else:
        await update.message.reply_text(cart_text, reply_markup=reply_markup)

async def contacts(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Display contact information."""
    contacts_text = """
📱 Наши контакты:

☎️ Call-центр: +998 (78) 129-70-00
📍 Адреса ресторанов:
• Ташкент, ул. Навои, 1
• Ташкент, ул. Амира Темура, 15
• Самарканд, ул. Регистан, 5

⏰ Время работы: 10:00 - 23:00
    """
    keyboard = [[InlineKeyboardButton("◀️ Назад", callback_data='back')]]
    reply_markup = InlineKeyboardMarkup(keyboard)

    if update.callback_query:
        await update.callback_query.edit_message_text(contacts_text, reply_markup=reply_markup)
    else:
        await update.message.reply_text(contacts_text, reply_markup=reply_markup)

async def settings(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Display settings options."""
    settings_text = """
⚙️ Настройки:

Выберите действие:
    """
    keyboard = [
        [InlineKeyboardButton("🌐 Язык", callback_data='language')],
        [InlineKeyboardButton("📱 Номер телефона", callback_data='phone')],
        [InlineKeyboardButton("◀️ Назад", callback_data='back')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)

    if update.callback_query:
        await update.callback_query.edit_message_text(settings_text, reply_markup=reply_markup)
    else:
        await update.message.reply_text(settings_text, reply_markup=reply_markup)

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handle incoming messages and route to appropriate functions."""
    text = update.message.text

    if text == "🍗 Меню":
        await show_menu(update, context)
    elif text == "🛒 Корзина":
        await show_cart(update, context)
    elif text == "📱 Контакты":
        await contacts(update, context)
    elif text == "⚙️ Настройки":
        await settings(update, context)
    else:
        await update.message.reply_text("Пожалуйста, используйте кнопки меню.")

async def button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handle button presses."""
    query = update.callback_query
    await query.answer()


# Route based on the callback data
    if query.data == 'menu':
        await show_menu(update, context)
    elif query.data.startswith('category_'):
        await show_category(update, context)
    elif query.data.startswith('add_'):
        await add_to_cart(update, context)
    elif query.data.startswith('remove_'):
        await remove_from_cart(update, context)
    elif query.data == 'cart':
        await show_cart(update, context)
    elif query.data == 'contacts':
        await contacts(update, context)
    elif query.data == 'settings':
        await settings(update, context)
    elif query.data == 'back':
        await start(update, context)
    elif query.data == 'clear_cart':
        user_id = query.from_user.id
        user_cart[user_id] = {}
        await show_cart(update, context)
    elif query.data == 'checkout':
        await query.edit_message_text("Для оформления заказа, пожалуйста, отправьте ваш номер телефона.")
    elif query.data == 'phone':
        await query.message.reply_text(
            "📱 Пожалуйста, отправьте свой номер телефона, используя кнопку ниже.",
            reply_markup=ReplyKeyboardMarkup(
                [[KeyboardButton("📲 Отправить номер", request_contact=True)]],
                resize_keyboard=True,
                one_time_keyboard=True
            )
        )
    elif query.data == 'language':
        keyboard = [
            [InlineKeyboardButton("🇷🇺 Русский", callback_data='lang_ru')],
            [InlineKeyboardButton("🇺🇿 O‘zbekcha", callback_data='lang_uz')],
            [InlineKeyboardButton("◀️ Назад", callback_data='settings')]
        ]
        await query.edit_message_text("🌐 Выберите язык:", reply_markup=InlineKeyboardMarkup(keyboard))
    elif query.data == 'lang_ru':
        await query.answer("Язык установлен: Русский")
        await settings(update, context)
    elif query.data == 'lang_uz':
        await query.answer("Til o‘rnatildi: O‘zbekcha")
        await settings(update, context)

async def handle_contact(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handle user contact information."""
    contact: Contact = update.message.contact
    if contact:
        await update.message.reply_text(
            f"✅ Ваш заказ принят! Ваш номер телефона: {contact.phone_number}",
            reply_markup=get_main_menu()
        )
    else:
        await update.message.reply_text("❗️ Пожалуйста, отправьте контакт с помощью кнопки.")

def main() -> None:
    """Run the bot."""
    application = Application.builder().token(TOKEN).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.add_handler(CallbackQueryHandler(button))
    application.add_handler(MessageHandler(filters.CONTACT, handle_contact))

    print("🤖 Bot is running...")
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if name == 'main':
    main()
