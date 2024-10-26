import openai
from telegram import Update
from telegram.ext import CallbackContext, CommandHandler, Updater, MessageHandler, Filters
import sys
import os
import json
from copy import deepcopy

# Pastikan Anda menambahkan token API OpenAI dan token bot Telegram sebagai argumen saat menjalankan
openai.api_key, TOKEN = sys.argv[1:3]

# Path untuk menyimpan data
DATA_path = "data"
IDS_path = f"{DATA_path}/IDS.json"
MEMORY_path = f"{DATA_path}/MEMORY.json"
if not os.path.exists(DATA_path):
    os.makedirs(DATA_path)

# Inisialisasi file JSON jika belum ada
if not os.path.exists(IDS_path):
    with open(IDS_path, 'w+') as f:
        json.dump({}, f)

if not os.path.exists(MEMORY_path):
    with open(MEMORY_path, 'w+') as f:
        json.dump({}, f)

with open(IDS_path, 'r') as f:
    IDS = json.load(f)

with open(MEMORY_path, 'r') as f:
    MEMORY = json.load(f)

# Konstanta
DEFAULT_DICT = {'state': 'chat', 'a': [], 'q': []}
MYID = 283460642
GODS = {MYID: "おはよう　おもさま"}

# Fungsi untuk membuat permintaan ke GPT
def GPTchat(prompt):
    return openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": prompt},
        ]
    )['choices'][0]['message']['content']

# Fungsi untuk membuat gambar
def GPTimg(prompt):
    return openai.Image.create(
        prompt=prompt,
        n=1,
        size="1024x1024"
    )['data'][0]['url']

# Mengisi data pengguna ke dalam memori
def _fill(chat_id, username):
    if username not in IDS:
        IDS[username] = chat_id
        IDS[str(chat_id)] = username
        with open(IDS_path, 'w+') as fp:
            json.dump(IDS, fp)
    if chat_id not in MEMORY:
        MEMORY[chat_id] = deepcopy(DEFAULT_DICT)

# Command /start
def start_command(update: Update, context: CallbackContext) -> None:
    chat_id = update.message.chat_id
    _fill(chat_id, update.message.from_user.username)
    update.message.reply_text('Selamat Datang. Aku dalah AI Mario, ada yang bisa Aku bantu kali ini? | gunakan /chat untuk memulai sesi chat! | Gunakan sebijaknya API nya Bayar hiks hiks') 

# Command /img
def img(update: Update, context: CallbackContext) -> None:
    chat_id = update.message.chat_id
    _fill(chat_id, update.message.from_user.username)
    MEMORY[chat_id]['state'] = 'img'
    update.message.reply_text('Mode gambar diaktifkan. Kirimkan deskripsi gambar.')

# Command /chat
def chat(update: Update, context: CallbackContext) -> None:
    chat_id = update.message.chat_id
    _fill(chat_id, update.message.from_user.username)
    MEMORY[chat_id]['state'] = 'chat'
    update.message.reply_text('Mode chat diaktifkan.')

# Handler untuk teks yang masuk
def handleGPT(update: Update, context: CallbackContext):
    chat_id = update.message.chat_id
    msg = update.message.text.lower()
    _fill(chat_id, update.message.from_user.username)

    if MEMORY[chat_id]['state'] == 'img':
        # Menggunakan API gambar
        update.message.reply_text(GPTimg(msg))
    else:
        # Menggunakan API chat
        update.message.reply_text(GPTchat(msg))

# Menyiapkan handler
handlers = [
    CommandHandler('start', start_command),
    CommandHandler('img', img),
    CommandHandler('chat', chat),
    MessageHandler(Filters.text, handleGPT),
]

updater = Updater(TOKEN, workers=4)  # Menyesuaikan jumlah worker
for handler in handlers:
    updater.dispatcher.add_handler(handler)

updater.start_polling()
print('Bot dimulai')
updater.idle()

