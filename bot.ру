import telebot
from telebot import types
import sqlite3
import json

TOKEN = '7667180876:AAFC0oivcoBQGToA36gyTO6aS48Z0Wjzbuw'  # твой токен
ADMIN_CHAT_ID = 696091147  # твой Telegram ID (цифрами)

bot = telebot.TeleBot(TOKEN)

conn = sqlite3.connect('orders.db', check_same_thread=False)
cursor = conn.cursor()

cursor.execute('''
CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    username TEXT,
    full_name TEXT,
    phone TEXT,
    address TEXT,
    cart TEXT,
    total INTEGER,
    status TEXT DEFAULT 'Новый'
)
''')
conn.commit()

products = {
    "Вейп Starter Kit": 2500,
    "Жидкость 30 мл": 700,
    "Зарядное устройство": 500,
    "Кейс для вейпа": 600,
    "Запасные испарители (5 шт.)": 800,
}

carts = {}
user_states = {}

def get_cart_text(user_id):
    cart = carts.get(user_id, {})
    if not cart:
        return "Ваша корзина пуста."
    text = "Ваша корзина:\n"
    total = 0
    for product, qty in cart.items():
        price = products[product]
        text += f"{product} — {qty} шт. x {price}₽ = {qty * price}₽\n"
        total += qty * price
    text += f"\nИтого: {total}₽"
    return text

def calculate_total(user_id):
    cart = carts.get(user_id, {})
    total = 0
    for product, qty in cart.items():
        total += products[product] * qty
    return total

def save_order(user_id, username, full_name, phone, address):
    cart = carts.get(user_id, {})
    if not cart:
        return False
    cart_json = json.dumps(cart, ensure_ascii=False)
    total = calculate_total(user_id)
    cursor.execute('''
        INSERT INTO orders (user_id, username, full_name, phone, address, cart, total)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    ''', (user_id, username, full_name, phone, address, cart_json, total))
    conn.commit()
    carts[user_id] = {}
    return True

@bot.message_handler(commands=['start'])
def start(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    markup.add("Мне есть 18", "Мне нет 18")
    bot.send_message(message.chat.id, 
                     "Привет! Перед покупкой вейпов нужно подтвердить, что вам есть 18 лет.", 
                     reply_markup=markup)

@bot.message_handler(func=lambda m: m.text == "Мне нет 18")
def underage(message):
    bot.send_message(message.chat.id, "Извините, продажа запрещена лицам младше 18 лет.")

@bot.message_handler(func=lambda m: m.text == "Мне есть 18")
def show_products(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    for product in products.keys():
        markup.add(product)
    markup.add("Корзина", "Оформить заказ", "Очистить корзину")
    bot.send_message(message.chat.id, "Выберите товар для добавления в корзину:", reply_markup=markup)
    carts.setdefault(message.chat.id, {})

@bot.message_handler(func=lambda m: m.text in products.keys())
def add_product(message):
    user_id = message.chat.id
    cart = carts.setdefault(user_id, {})
    product = message.text
    cart[product] = cart.get(product, 0) + 1
    bot.send_message(user_id, f"Добавлено: {product} (в корзине {cart[product]} шт.)")

@bot.message_handler(func=lambda m: m.text == "Корзина")
def show_cart(message):
    text = get_cart_text(message.chat.id)
    bot.send_message(message.chat.id, text)

@bot.message_handler(func=lambda m: m.text == "Очистить корзину")
def clear_cart(message):
    carts[message.chat.id] = {}
    bot.send_message(message.chat.id, "Корзина очищена.")

@bot.message_handler(func=lambda m: m.text == "Оформить заказ")
def start_order(message):
    user_id = message.chat.id
    cart = carts.get(user_id, {})
    if not cart:
        bot.send_message(user_id, "Ваша корзина пуста. Добавьте товары перед оформлением.")
        return
    user_states[user_id] = 'awaiting_phone'
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    markup.add(types.KeyboardButton("Отправить номер", request_contact=True))
    bot.send_message(user_id, "Пожалуйста, отправьте ваш контактный номер телефона.", reply_markup=markup)

@bot.message_handler(content_types=['contact'])
def receive_contact(message):
    user_id = message.chat.id
    if user_states.get(user_id) != 'awaiting_phone':
        return
    if not message.contact or message.contact.user_id != user_id:
        bot.send_message(user_id, "Пожалуйста, используйте кнопку для отправки вашего номера.")
        return
    user_states[user_id] = {'phone': message.contact.phone_number}
    bot.send_message(user_id, "Отлично! Теперь отправьте ваш полный адрес доставки.")

@bot.message_handler(func=lambda m: user_states.get(m.chat.id) and isinstance(user_states[m.chat.id], dict) and 'phone' in user_states[m.chat.id])
def receive_address(message):
    user_id = message.chat.id
    state_data = user_states[user_id]
    address = message.text.strip()
    if len(address) < 5:
        bot.send_message(user_id, "Пожалуйста, введите более точный адрес.")
        return
    phone = state_data['phone']
    username = message.from_user.username or ''
    full_name = f"{message.from_user.first_name or ''} {message.from_user.last_name or ''}".strip()
    saved = save_order(user_id, username, full_name, phone, address)
    if not saved:
        bot.send_message(user_id, "Ошибка при сохранении заказа. Попробуйте снова.")
        user_states.pop(user_id, None)
        return
    
    cart = carts.get(user_id, {})
    total = calculate_total(user_id)
    order_text = f"Новый заказ!\n\nПользователь: @{username} ({full_name})\nТелефон: {phone}\nАдрес: {address}\n\nТовары:\n"
    for product, qty in cart.items():
        order_text += f"- {product} x{qty} = {products[product]*qty}₽\n"
    order_text += f"\nИтого: {total}₽"
    bot.send_message(ADMIN_CHAT_ID, order_text)

    payment_link = f"https://yourpaymentlink.com/pay?amount={total}"
    bot.send_message(user_id, f"Спасибо за заказ! Для оплаты перейдите по ссылке:\n{payment_link}")

    bot.send_message(user_id, "Ваш заказ принят. В ближайшее время с вами свяжутся для подтверждения.")
    user_states.pop(user_id, None)

@bot.message_handler(func=lambda m: True)
def fallback(message):
    bot.send_message(message.chat.id, "Пожалуйста, используйте меню для взаимодействия с ботом.")

if __name__ == '__main__':
    print("Бот запущен...")
    bot.infinity_polling()
