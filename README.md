import os
import telebot
import pyrebase
from datetime import datetime
import random

# إعداد Firebase
firebase_config = {
    "apiKey": "AlzaSyCU76mpgyBpAjWsTah8jSsvLL2X0DaKt1g",
    "authDomain": "bashman-5d67c.firebaseapp.com",
    "databaseURL": "https://bashman-5d67c-default-rtdb.firebaseio.com/",
    "storageBucket": "bashman-5d67c.appspot.com",
}
firebase = pyrebase.initialize_app(firebase_config)
db = firebase.database()

# إعداد التوكن للبوت
API_TOKEN = '8183208968:AAFCV-lsLrhGZ091sRoF7mvZLTi-pbXkPmc'  # توكنك
bot = telebot.TeleBot(API_TOKEN)

# إعداد المتغيرات
online_users = set()
selected_lib = None
key_verified = False  # للتأكد من أن المفتاح تم التحقق منه
libraries = {
    "libUE4.so": "تحل باند عشر سنوات",
    "libTDataMaster.so": "تحل يومًا وغيابيًا ومراقبة",
    "libanogs.so": "تحل شهرًا",
    "libgcloud.so": "تحل أسبوعًا",
    "libCrashSight.so": "تحل لكراش بديل libanort.so"
}

# مجموعة لتخزين الأوفستات المولدة
generated_offsets = set()

# توليد أوفست عشوائي
def generate_random_offset():
    while True:
        # توليد أوفست عشوائي يبدأ بـ "0x" ويتبعه رقم عشوائي
        offset = f"0x{random.randint(0x000000, 0xFFFFFF):X}"
        
        # التحقق إذا كان الأوفست موجود بالفعل
        if offset not in generated_offsets:
            generated_offsets.add(offset)
            return offset

# التعامل مع أمر /start
@bot.message_handler(commands=["start"])
def start_handler(message):
    global online_users
    if message.chat.id not in online_users:
        online_users.add(message.chat.id)
    bot.send_message(
        message.chat.id,
        f"مرحبًا بك في بوت BASHA! 👋\n"
        f"هناك {len(online_users)} مستخدمين متصلين الآن.\n"
        "يرجى الاشتراك في قناتنا على Telegram عبر الرابط التالي: "
        "@MODBASHA\n"
        "ثم أرسل لي مفتاح التفعيل الخاص بك لكي أبدأ مساعدتك."
    )

# التعامل مع ملفات key.txt
@bot.message_handler(content_types=['document'])
def document_handler(message):
    global selected_lib, key_verified

    file_info = bot.get_file(message.document.file_id)
    downloaded_file = bot.download_file(file_info.file_path)
    file_name = message.document.file_name

    # حفظ الملف
    with open(file_name, "wb") as f:
        f.write(downloaded_file)

    if file_name == "key.txt":
        with open(file_name, "r") as f:
            key_data = f.read().strip()

        # التحقق من المفتاح
        if all(keyword in key_data for keyword in ["MOD BASHA VIP", "key", "Over", "NOPRS FOUN"]):
            try:
                key_parts = key_data.split("\n")
                expiration_date_str = key_parts[2].split("Over ")[1]
                expiration_date = datetime.strptime(expiration_date_str, "%Y-%m-%d %H:%M:%S")
                current_time = datetime.now()

                if expiration_date > current_time:
                    key_verified = True
                    bot.reply_to(
                        message,
                        "تم التحقق من المفتاح بنجاح! اختر المكتبة:",
                    )
                    show_library_options(message)
                else:
                    bot.reply_to(message, "تم انتهاء مدة صلاحية المفتاح. تواصل مع المطور لتجديد الاشتراك @MODS_BASHA")
            except Exception as e:
                bot.reply_to(message, f"حدث خطأ أثناء معالجة المفتاح: {str(e)}")
        else:
            bot.reply_to(message, "المفتاح غير صحيح أو التنسيق غير متطابق. تأكد من الملف المرسل.")
    elif file_name == "main.cpp":
        if key_verified and selected_lib:
            result = add_offsets_to_main_cpp(file_name, selected_lib)
            if result.startswith("تم"):
                modified_file_name = f"BASHAVIP_{file_name}"
                with open(modified_file_name, "rb") as modified_file:
                    bot.send_document(message.chat.id, modified_file, caption="📂 النسخة المعدلة من ملف main.cpp")
                if selected_lib == "وضع جميع المكتبات":
                    report = "✅ تقرير الحماية:\n" + "\n".join(
                        [f"- المكتبة: {lib}\n  التأثير: {effect}" for lib, effect in libraries.items()]
                    )
                else:
                    report = f"✅ تقرير الحماية:\n- المكتبة: {selected_lib}\n- التأثير: {libraries[selected_lib]}"
                bot.send_message(message.chat.id, report)
            else:
                bot.reply_to(message, result)
        else:
            bot.reply_to(message, "يرجى التحقق من المفتاح واختيار المكتبة أولاً.")
    else:
        bot.reply_to(message, "يرجى إرسال ملف باسم 'key.txt' أو 'main.cpp'.")

# عرض المكتبات للاختيار
def show_library_options(message):
    markup = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    for lib in libraries.keys():
        markup.add(telebot.types.KeyboardButton(lib))
    markup.add(telebot.types.KeyboardButton("وضع جميع المكتبات"))
    bot.send_message(message.chat.id, "اختر المكتبة:", reply_markup=markup)

# التعامل مع اختيار المكتبة وطلب ملف main.cpp
@bot.message_handler(func=lambda message: message.text in libraries.keys() or message.text == "وضع جميع المكتبات")
def handle_lib_selection(message):
    global selected_lib
    selected_lib = message.text
    bot.send_message(message.chat.id, f"تم اختيار {selected_lib}. يرجى إرسال ملف main.cpp للتعديل عليه.")

# توليد أوفست وهيكسات عشوائية
def generate_random_hex():
    return "/".join(f"{random.randint(0, 255):02X}" for _ in range(4))

# تعديل ملف main.cpp
def add_offsets_to_main_cpp(cpp_file, lib_name):
    try:
        with open(cpp_file, "r") as f:
            lines = f.readlines()

        if lib_name == "وضع جميع المكتبات":
            offset_code = "".join(
                f'MemoryPatch::createWithHex("{lib}", {generate_random_offset()}, "{generate_random_hex()}");/*⟿✆♔MOD BASHA♔〄*/\n'
                for lib in libraries.keys() for _ in range(10)
            )
        else:
            offset_code = "".join(
                f'MemoryPatch::createWithHex("{lib_name}", {generate_random_offset()}, "{generate_random_hex()}");/*⟿✆♔MOD BASHA♔〄*/\n'
                for _ in range(50)
            )

        found_location = False
        for idx, line in enumerate(lines):
            if "void DrawEsp(ImDrawList *draw) {" in line:
                if "if (Config.Bypass)" in lines[idx + 1]:
                    lines.insert(idx + 2, offset_code)
                else:
                    lines.insert(idx + 1, offset_code)
                found_location = True
                break

        if not found_location:
            return "النص المطلوب غير موجود في ملف main.cpp."

        # تعديل اسم الملف المعدل هنا
        modified_file_name = f"BASHAVIP_{cpp_file}"
        with open(modified_file_name, "w") as f:
            f.writelines(lines)

        return "تم إضافة الحماية بنجاح إلى main.cpp."
    except Exception as e:
        return f"خطأ أثناء معالجة الملف: {str(e)}"

# تشغيل البوت
if __name__ == "__main__":
    bot.polling(none_stop=True)
