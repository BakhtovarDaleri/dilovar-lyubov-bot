"""
Telegram Bot for Дилoвар (склад) и Любовь (бухгалтерия) — узкий бот, пишет в ту же базу,
что и основной "Учеты Производство".
"""
import os
import logging
from datetime import datetime, timedelta
import zoneinfo
from dotenv import load_dotenv

load_dotenv()

from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, MessageHandler,
    ConversationHandler, ContextTypes, filters,
)
from supabase import create_client, Client

# --- CONFIGURATION ---
BOT_TOKEN = os.getenv("BOT_TOKEN", "")
SUPABASE_URL = os.getenv("SUPABASE_URL", "")
SUPABASE_KEY = os.getenv("SUPABASE_KEY", "")
DILOVAR_ID = int(os.getenv("DILOVAR_ID", "0"))
LYUBOV_ID = int(os.getenv("LYUBOV_ID", "0"))
ADMIN_NOTIFY_ID = int(os.getenv("ADMIN_NOTIFY_ID", "5450770011"))  # Бахтовар — по умолчанию, чтобы точно был доступ

TZ_MSK = zoneinfo.ZoneInfo("Europe/Moscow")
logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)
logger = logging.getLogger(__name__)

# Расчёт зарплаты Диловара за упаковку
DILOVAR_RATE = 4550
DILOVAR_MIN_UNITS = 550


class SupabaseService:
    def __init__(self):
        self.client: Client = create_client(SUPABASE_URL, SUPABASE_KEY)
        self._category_cache = {}
        self._ip_cache = {}
        self._account_cache = {}
        self._load_caches()

    def _load_caches(self):
        cats = self.client.table("categories").select("id,name").execute().data or []
        self._category_cache = {c["name"]: c["id"] for c in cats}
        ips = self.client.table("ip").select("id,name").execute().data or []
        self._ip_cache = {i["name"]: i["id"] for i in ips}
        accs = self.client.table("accounts").select("id,code").execute().data or []
        self._account_cache = {a["code"]: a["id"] for a in accs}

    def get_category_id(self, name): return self._category_cache.get(name)
    def get_ip_id(self, name): return self._ip_cache.get(name)
    def get_account_id(self, code): return self._account_cache.get(code)

    def post_journal_entry(self, operation_id, entry_date, debit_code, credit_code, amount, comment=""):
        self.client.table("journal_entries").insert({
            "operation_id": operation_id, "entry_date": entry_date,
            "debit_account_id": self.get_account_id(debit_code),
            "credit_account_id": self.get_account_id(credit_code),
            "amount": amount, "comment": comment,
        }).execute()

    def get_suppliers_list(self, type_filter=None):
        q = self.client.table("counterparties").select("id,name,phone,type")
        if type_filter:
            q = q.eq("type", type_filter)
        return q.order("name").execute().data or []

    def get_consumables_list(self):
        res = self.client.table("consumables_catalog").select("id,name,size").order("name").execute()
        items = []
        for r in (res.data or []):
            items.append(f"{r['name']} {r['size']}" if r.get("size") else r["name"])
        return items

    def get_counterparty_by_name(self, name):
        data = self.client.table("counterparties").select("*").eq("name", name).limit(1).execute().data or []
        return data[0] if data else None

    def add_operation_returning_id(self, **fields):
        res = self.client.table("operations").insert(fields).execute()
        return res.data[0]["id"]

    def add_warehouse_movement(self, **fields):
        self.client.table("warehouse_movements").insert(fields).execute()

    def get_hourly_employees(self):
        res = self.client.table("counterparties").select("id,name,hourly_rate").eq("type", "сотрудник").not_.is_("hourly_rate", "null").order("name").execute()
        return res.data or []

    def get_last_operations_for_period(self, date_from, date_to, limit=40):
        res = (
            self.client.table("operations").select("*, counterparties(name)")
            .gte("operation_date", date_from).lte("operation_date", date_to)
            .order("operation_date", desc=True).limit(limit)
            .execute()
        )
        return res.data or []


RULES_TEXT = "🛑 Числа вводим без букв (просто `500`).\nКнопка «❌ Главное меню» отменяет любой шаг."


def get_menu_keyboard(user_id):
    if user_id == ADMIN_NOTIFY_ID:
        return ReplyKeyboardMarkup([["📦 Расходники", "👤 Сотрудники"], ["📊 Отчёт за период", "💵 Мой оклад"], ["❓ Помощь"]], resize_keyboard=True)
    if user_id == DILOVAR_ID:
        return ReplyKeyboardMarkup([["📦 Расходники", "👤 Сотрудники"], ["❓ Помощь"]], resize_keyboard=True)
    if user_id == LYUBOV_ID:
        return ReplyKeyboardMarkup([["📊 Отчёт за период", "💵 Мой оклад"], ["❓ Помощь"]], resize_keyboard=True)
    return None


def get_step_keyboard():
    return ReplyKeyboardMarkup([["🔙 Назад", "❌ Главное меню"]], resize_keyboard=True)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    kb = get_menu_keyboard(uid)
    if not kb:
        await update.message.reply_text(
            f"🔒 У вас нет доступа к этому боту.\n\n"
            f"Ваш ID: `{uid}`\n\n"
            f"Перешлите этот номер администратору — он добавит вас.",
            parse_mode="Markdown",
        )
        return
    await update.message.reply_text(f"👋 Добро пожаловать!\n\n{RULES_TEXT}", reply_markup=kb, parse_mode="Markdown")


async def cancel_to_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Возврат в главное меню.", reply_markup=get_menu_keyboard(update.effective_user.id))
    return ConversationHandler.END


async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(RULES_TEXT, reply_markup=get_menu_keyboard(update.effective_user.id), parse_mode="Markdown")


# --- STATES ---
SUPPLY_SUPPLIER = "SUPPLY_SUPPLIER"
SUPPLY_PRODUCT = "SUPPLY_PRODUCT"
SUPPLY_QTY = "SUPPLY_QTY"
SUPPLY_PRICE = "SUPPLY_PRICE"
SUPPLY_ADD_MORE = "SUPPLY_ADD_MORE"
SUPPLY_COMMENT = "SUPPLY_COMMENT"
SUPPLY_CONFIRM = "SUPPLY_CONFIRM"
EMP_MENU = "EMP_MENU"
SHIFT_EMPLOYEE = "SHIFT_EMPLOYEE"
SHIFT_DATE = "SHIFT_DATE"
SHIFT_START = "SHIFT_START"
SHIFT_END = "SHIFT_END"
SHIFT_CONFIRM = "SHIFT_CONFIRM"
ACCRUAL_EMPLOYEE = "ACCRUAL_EMPLOYEE"
ACCRUAL_AMOUNT = "ACCRUAL_AMOUNT"
ACCRUAL_CONFIRM = "ACCRUAL_CONFIRM"
PACK_EMPLOYEE = "PACK_EMPLOYEE"
PACK_DATE = "PACK_DATE"
PACK_UNITS = "PACK_UNITS"
PACK_CONFIRM = "PACK_CONFIRM"
REPORT_PERIOD = "REPORT_PERIOD"
SALARY_CONFIRM = "SALARY_CONFIRM"


def build_grid(items, columns=2):
    grid, row = [], []
    for it in items:
        row.append(str(it))
        if len(row) == columns:
            grid.append(row); row = []
    if row: grid.append(row)
    grid.append(["🔙 Назад", "❌ Главное меню"])
    return ReplyKeyboardMarkup(grid, resize_keyboard=True)


def parse_flexible_date(text, tz):
    raw = text.strip().lower()
    now = datetime.now(tz)
    if raw == "сегодня":
        d = now.date(); return d.isoformat(), d.strftime("%d.%m.%Y")
    if raw == "вчера":
        d = (now - timedelta(days=1)).date(); return d.isoformat(), d.strftime("%d.%m.%Y")
    parts = text.strip().replace(" ", "").split(".")
    try:
        if len(parts) == 1 and parts[0].isdigit():
            d = now.date().replace(day=int(parts[0]))
        elif len(parts) == 2 and all(p.isdigit() for p in parts):
            d = now.date().replace(day=int(parts[0]), month=int(parts[1]))
        elif len(parts) == 3 and all(p.isdigit() for p in parts):
            year = int(parts[2])
            if year < 100: year += 2000
            d = now.date().replace(day=int(parts[0]), month=int(parts[1]), year=year)
        else:
            return None
    except ValueError:
        return None
    return d.isoformat(), d.strftime("%d.%m.%Y")


# --- РАСХОДНИКИ (корзина, как в основном боте) ---
async def supply_start(update, context):
    context.user_data["s_items"] = []
    db = context.bot_data["db"]
    sups = [s["name"] for s in db.get_suppliers_list(type_filter="поставщик расходников")]
    await update.message.reply_text("📦 Шаг 1: Выберите поставщика:", reply_markup=build_grid(sups))
    return SUPPLY_SUPPLIER


SHAHBOZ_ITEMS = ["Банановая коробка", "Европаддон"]


async def supply_supplier(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад": return await supply_start(update, context)
    context.user_data["s_supplier"] = t
    db = context.bot_data["db"]
    all_items = db.get_consumables_list()

    if "Шахбоз" in t:
        items = [i for i in all_items if any(s in i for s in SHAHBOZ_ITEMS)]
    else:
        items = [i for i in all_items if not any(s in i for s in SHAHBOZ_ITEMS)]

    await update.message.reply_text("Шаг 2: Выберите товар:", reply_markup=build_grid(items))
    return SUPPLY_PRODUCT


async def supply_product(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад": return await supply_start(update, context)
    context.user_data["s_cur_product"] = t
    await update.message.reply_text(f"«{t}» — количество (шт):", reply_markup=get_step_keyboard())
    return SUPPLY_QTY


async def supply_qty(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад":
        db = context.bot_data["db"]
        supplier = context.user_data.get("s_supplier", "")
        all_items = db.get_consumables_list()
        if "Шахбоз" in supplier:
            items = [i for i in all_items if any(s in i for s in SHAHBOZ_ITEMS)]
        else:
            items = [i for i in all_items if not any(s in i for s in SHAHBOZ_ITEMS)]
        await update.message.reply_text("Шаг 2: Выберите товар:", reply_markup=build_grid(items))
        return SUPPLY_PRODUCT
    try: context.user_data["s_cur_qty"] = float(t.replace(",", "."))
    except ValueError:
        await update.message.reply_text("⚠️ Нужно число. Попробуйте ещё раз:", reply_markup=get_step_keyboard())
        return SUPPLY_QTY
    await update.message.reply_text("Цена за 1 шт (₽):", reply_markup=get_step_keyboard())
    return SUPPLY_PRICE


async def supply_price(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад":
        await update.message.reply_text(f"«{context.user_data['s_cur_product']}» — количество (шт):", reply_markup=get_step_keyboard())
        return SUPPLY_QTY
    try: price = float(t.replace(",", "."))
    except ValueError:
        await update.message.reply_text("⚠️ Нужно число. Попробуйте ещё раз:", reply_markup=get_step_keyboard())
        return SUPPLY_PRICE
    qty = context.user_data["s_cur_qty"]
    context.user_data.setdefault("s_items", []).append({
        "product": context.user_data["s_cur_product"], "qty": qty, "price": price, "total": round(qty * price, 2),
    })
    items_text = "\n".join(f"{i+1}. {it['product']} — {it['qty']} шт × {it['price']}₽ = {it['total']}₽" for i, it in enumerate(context.user_data["s_items"]))
    total = round(sum(it["total"] for it in context.user_data["s_items"]), 2)
    kb = [["➕ Добавить ещё", "✅ Это всё"], ["🔙 Назад", "❌ Главное меню"]]
    await update.message.reply_text(f"🧺 Корзина:\n{items_text}\n\nИтого: {total}₽\n\nЕщё позицию?", reply_markup=ReplyKeyboardMarkup(kb, resize_keyboard=True))
    return SUPPLY_ADD_MORE


async def supply_add_more(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад":
        items = context.user_data.get("s_items", [])
        if items: items.pop()
        await update.message.reply_text("Цена за 1 шт (₽):", reply_markup=get_step_keyboard())
        return SUPPLY_PRICE
    if t == "➕ Добавить ещё":
        db = context.bot_data["db"]
        supplier = context.user_data.get("s_supplier", "")
        all_items = db.get_consumables_list()
        if "Шахбоз" in supplier:
            items = [i for i in all_items if any(s in i for s in SHAHBOZ_ITEMS)]
        else:
            items = [i for i in all_items if not any(s in i for s in SHAHBOZ_ITEMS)]
        await update.message.reply_text("Выберите товар:", reply_markup=build_grid(items))
        return SUPPLY_PRODUCT
    if t == "✅ Это всё":
        await update.message.reply_text("Комментарий (или '-'):", reply_markup=get_step_keyboard())
        return SUPPLY_COMMENT
    return SUPPLY_ADD_MORE


async def supply_comment(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    context.user_data["s_comment"] = t if t != "-" else ""
    d = context.user_data
    items_text = "\n".join(f"{i+1}. {it['product']} — {it['qty']} шт × {it['price']}₽" for i, it in enumerate(d["s_items"]))
    total = round(sum(it["total"] for it in d["s_items"]), 2)
    await update.message.reply_text(f"📋 Проверка:\n🏢 {d['s_supplier']}\n{items_text}\n\nИтого: {total}₽", reply_markup=ReplyKeyboardMarkup([["✅ Подтвердить"], ["🔙 Назад", "❌ Главное меню"]], resize_keyboard=True))
    return SUPPLY_CONFIRM


async def supply_confirm(update, context):
    t = update.message.text.strip()
    if t != "✅ Подтвердить": return await cancel_to_menu(update, context)
    db, d = context.bot_data["db"], context.user_data
    cp = db.get_counterparty_by_name(d["s_supplier"])
    category_id = db.get_category_id("Расходники (упаковка, скотч, этикетки)")
    op_date = datetime.now(TZ_MSK).date().isoformat()

    for item in d["s_items"]:
        op_id = db.add_operation_returning_id(
            operation_date=op_date, ip_id=None, counterparty_id=(cp or {}).get("id"),
            category_id=category_id, operation_type="покупка", amount=item["total"],
            quantity=item["qty"], price=item["price"], item_name=item["product"],
            entered_by=f"dilovar_{update.effective_user.id}", status="confirmed",
            payment_method=None, comment=d["s_comment"],
        )
        db.add_warehouse_movement(
            direction="приход", flow_type="расходники_от_поставщика", product_name=item["product"],
            quantity=item["qty"], unit="шт", unit_price=item["price"], movement_date=op_date,
            counterparty_id=(cp or {}).get("id"), ip_id=None, marketplace=None, note=d["s_comment"],
        )
        db.post_journal_entry(operation_id=op_id, entry_date=op_date, debit_code="1200", credit_code="2000",
                               amount=item["total"], comment=f"Закупка (Диловар): {item['product']}")

    total = round(sum(it["total"] for it in d["s_items"]), 2)
    await update.message.reply_text(f"✅ Занесено: {len(d['s_items'])} поз. на {total}₽.", reply_markup=get_menu_keyboard(update.effective_user.id))
    if ADMIN_NOTIFY_ID:
        await context.bot.send_message(chat_id=ADMIN_NOTIFY_ID, text=f"📦 Диловар внёс закупку расходников: {total}₽ ({d['s_supplier']})")
    return ConversationHandler.END


# --- СОТРУДНИКИ (меню-развилка) ---
async def employee_menu_start(update, context):
    kb = [["🕐 Внести смену", "📦 Упаковка (норма)"], ["💵 Начислить оклад"], ["❌ Главное меню"]]
    await update.message.reply_text("👤 Сотрудники — что делаем?", reply_markup=ReplyKeyboardMarkup(kb, resize_keyboard=True))
    return EMP_MENU


async def employee_menu_router(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    db = context.bot_data["db"]

    if t == "🕐 Внести смену":
        emps = db.get_hourly_employees()
        if not emps:
            await update.message.reply_text("Нет сотрудников с почасовой ставкой.", reply_markup=get_menu_keyboard(update.effective_user.id))
            return ConversationHandler.END
        context.user_data["shift_employees"] = {e["name"]: e for e in emps}
        await update.message.reply_text("Выберите сотрудника:", reply_markup=build_grid(list(context.user_data["shift_employees"].keys())))
        return SHIFT_EMPLOYEE

    elif t == "📦 Упаковка (норма)":
        return await pack_start(update, context)

    elif t == "💵 Начислить оклад":
        emps = db.get_suppliers_list(type_filter="сотрудник")
        context.user_data["accrual_employees"] = {e["name"]: e["id"] for e in emps}
        await update.message.reply_text("Выберите сотрудника:", reply_markup=build_grid(list(context.user_data["accrual_employees"].keys())))
        return ACCRUAL_EMPLOYEE

    return EMP_MENU


# --- Внести смену (гибкое время, как в основном боте) ---
def parse_flexible_time(text):
    raw = text.strip().replace(" ", "")
    if ":" in raw:
        try:
            datetime.strptime(raw, "%H:%M"); return raw
        except ValueError:
            return None
    if not raw.isdigit(): return None
    if len(raw) <= 2: h, m = int(raw), 0
    elif len(raw) == 3: h, m = int(raw[0]), int(raw[1:3])
    elif len(raw) == 4: h, m = int(raw[0:2]), int(raw[2:4])
    else: return None
    if not (0 <= h <= 23 and 0 <= m <= 59): return None
    return f"{h:02d}:{m:02d}"


MEAL_COMP_HOURS_THRESHOLD = 8
MEAL_COMP_AMOUNT = 350


async def shift_employee(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    employees = context.user_data.get("shift_employees", {})
    if t not in employees: return SHIFT_EMPLOYEE
    emp = employees[t]
    context.user_data["shift_emp_name"] = t
    context.user_data["shift_emp_id"] = emp["id"]
    context.user_data["shift_emp_rate"] = float(emp["hourly_rate"])
    kb = [["Сегодня", "Вчера"], ["🔙 Назад", "❌ Главное меню"]]
    await update.message.reply_text("Дата смены (Сегодня/Вчера или число):", reply_markup=ReplyKeyboardMarkup(kb, resize_keyboard=True))
    return SHIFT_DATE


async def shift_date(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад": return await employee_menu_start(update, context)
    parsed = parse_flexible_date(t, TZ_MSK)
    if not parsed:
        await update.message.reply_text("⚠️ Не понял дату. Попробуйте ещё раз:")
        return SHIFT_DATE
    context.user_data["shift_date_iso"], context.user_data["shift_date_display"] = parsed
    await update.message.reply_text("Время начала смены (например 9 или 900):", reply_markup=get_step_keyboard())
    return SHIFT_START


async def shift_start(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    parsed = parse_flexible_time(t)
    if not parsed:
        await update.message.reply_text("⚠️ Не понял время. Попробуйте ещё раз (например 9 или 900):")
        return SHIFT_START
    context.user_data["shift_start_time"] = parsed
    await update.message.reply_text("Время окончания смены (например 21 или 2100):", reply_markup=get_step_keyboard())
    return SHIFT_END


async def shift_end(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    parsed = parse_flexible_time(t)
    if not parsed:
        await update.message.reply_text("⚠️ Не понял время. Попробуйте ещё раз (например 21 или 2100):")
        return SHIFT_END

    d = context.user_data
    t_start = datetime.strptime(d["shift_start_time"], "%H:%M")
    t_end = datetime.strptime(parsed, "%H:%M")
    hours = (t_end - t_start).total_seconds() / 3600
    if hours <= 0: hours += 24
    rate = d["shift_emp_rate"]
    meal = MEAL_COMP_AMOUNT if hours >= MEAL_COMP_HOURS_THRESHOLD else 0
    total = round(hours * rate + meal, 2)
    d["shift_end_time"] = parsed
    d["shift_hours"] = round(hours, 2)
    d["shift_total"] = total

    meal_line = f"\n🍽 Обед: +{meal}₽" if meal else ""
    text = f"🕐 {d['shift_emp_name']}: {d['shift_date_display']} {d['shift_start_time']}–{parsed} ({d['shift_hours']}ч){meal_line}\n💰 Итого: {total}₽"
    await update.message.reply_text(text, reply_markup=ReplyKeyboardMarkup([["✅ Подтвердить"], ["❌ Главное меню"]], resize_keyboard=True))
    return SHIFT_CONFIRM


async def shift_confirm(update, context):
    t = update.message.text.strip()
    if t != "✅ Подтвердить": return await cancel_to_menu(update, context)
    db, d = context.bot_data["db"], context.user_data
    category_id = db.get_category_id("Зарплата")
    comment = f"Смена {d['shift_date_display']} {d['shift_start_time']}–{d['shift_end_time']} ({d['shift_hours']}ч), внёс Диловар"
    op_id = db.add_operation_returning_id(
        operation_date=d["shift_date_iso"], ip_id=None, counterparty_id=d["shift_emp_id"],
        category_id=category_id, operation_type="начисление", amount=d["shift_total"],
        entered_by=f"dilovar_{update.effective_user.id}", status="confirmed", payment_method=None, comment=comment,
    )
    db.post_journal_entry(operation_id=op_id, entry_date=d["shift_date_iso"], debit_code="5100", credit_code="2100",
                           amount=d["shift_total"], comment=f"Начисление {d['shift_emp_name']} (смена)")
    await update.message.reply_text(f"✅ Начислено {d['shift_total']}₽ для {d['shift_emp_name']}.", reply_markup=get_menu_keyboard(update.effective_user.id))
    if ADMIN_NOTIFY_ID:
        await context.bot.send_message(chat_id=ADMIN_NOTIFY_ID, text=f"🕐 Диловар внёс смену {d['shift_emp_name']}: {d['shift_date_display']} {d['shift_start_time']}–{d['shift_end_time']} → {d['shift_total']}₽")
    return ConversationHandler.END


# --- Начислить оклад (фикс. сумма любому сотруднику) ---
async def accrual_employee(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    employees = context.user_data.get("accrual_employees", {})
    if t not in employees: return ACCRUAL_EMPLOYEE
    context.user_data["accrual_emp_name"] = t
    context.user_data["accrual_emp_id"] = employees[t]
    await update.message.reply_text("Сумма (₽):", reply_markup=get_step_keyboard())
    return ACCRUAL_AMOUNT


async def accrual_amount(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    try: amount = float(t.replace(",", ".").replace(" ", ""))
    except ValueError:
        await update.message.reply_text("⚠️ Нужно число. Попробуйте ещё раз:", reply_markup=get_step_keyboard())
        return ACCRUAL_AMOUNT
    context.user_data["accrual_amount"] = amount
    d = context.user_data
    await update.message.reply_text(f"💵 Начислить {d['accrual_emp_name']}: {amount}₽?", reply_markup=ReplyKeyboardMarkup([["✅ Подтвердить"], ["❌ Главное меню"]], resize_keyboard=True))
    return ACCRUAL_CONFIRM


async def accrual_confirm(update, context):
    t = update.message.text.strip()
    if t != "✅ Подтвердить": return await cancel_to_menu(update, context)
    db, d = context.bot_data["db"], context.user_data
    category_id = db.get_category_id("Зарплата")
    op_date = datetime.now(TZ_MSK).date().isoformat()
    op_id = db.add_operation_returning_id(
        operation_date=op_date, ip_id=None, counterparty_id=d["accrual_emp_id"],
        category_id=category_id, operation_type="начисление", amount=d["accrual_amount"],
        entered_by=f"dilovar_{update.effective_user.id}", status="confirmed", payment_method=None,
        comment=f"Начисление оклада (внёс Диловар)",
    )
    db.post_journal_entry(operation_id=op_id, entry_date=op_date, debit_code="5100", credit_code="2100",
                           amount=d["accrual_amount"], comment=f"Начисление {d['accrual_emp_name']}")
    await update.message.reply_text(f"✅ Начислено {d['accrual_amount']}₽ для {d['accrual_emp_name']}.", reply_markup=get_menu_keyboard(update.effective_user.id))
    if ADMIN_NOTIFY_ID:
        await context.bot.send_message(chat_id=ADMIN_NOTIFY_ID, text=f"💵 Диловар начислил {d['accrual_emp_name']}: {d['accrual_amount']}₽")
    return ConversationHandler.END


# --- УПАКОВКА (расчёт зарплаты Диловара) ---
async def pack_start(update, context):
    db = context.bot_data["db"]
    emps = db.get_hourly_employees()
    if not emps:
        await update.message.reply_text("Нет сотрудников с почасовой ставкой в базе.", reply_markup=get_menu_keyboard(update.effective_user.id))
        return ConversationHandler.END
    context.user_data["pack_employees"] = {e["name"]: e["id"] for e in emps}
    kb = build_grid(list(context.user_data["pack_employees"].keys()))
    await update.message.reply_text("За кого вносим (кто упаковывал)?", reply_markup=kb)
    return PACK_EMPLOYEE


async def pack_employee(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад": return await pack_start(update, context)
    employees = context.user_data.get("pack_employees", {})
    if t not in employees:
        return PACK_EMPLOYEE
    context.user_data["pack_employee_name"] = t
    context.user_data["pack_employee_id"] = employees[t]
    kb = [["Сегодня", "Вчера"], ["🔙 Назад", "❌ Главное меню"]]
    await update.message.reply_text("Шаг 1: За какой день? (Сегодня/Вчера или число, напр. 15)", reply_markup=ReplyKeyboardMarkup(kb, resize_keyboard=True))
    return PACK_DATE


async def pack_date(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад": return await pack_start(update, context)
    parsed = parse_flexible_date(t, TZ_MSK)
    if not parsed:
        await update.message.reply_text("⚠️ Не понял дату. Попробуйте ещё раз:")
        return PACK_DATE
    context.user_data["pack_date_iso"], context.user_data["pack_date_display"] = parsed
    await update.message.reply_text("Шаг 2: Сколько штук упаковано за этот день (всего)?", reply_markup=get_step_keyboard())
    return PACK_UNITS


async def pack_units(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад": return await pack_start(update, context)
    try: units = float(t.replace(",", "."))
    except ValueError:
        await update.message.reply_text("⚠️ Нужно число. Попробуйте ещё раз:", reply_markup=get_step_keyboard())
        return PACK_UNITS

    context.user_data["pack_units"] = units
    # Зарплата за отработанный день — фиксированная, не уменьшается из-за меньшего кол-ва штук.
    # А вот себестоимость труда на 1 шт в этот конкретный день — реальная, пересчитывается по факту.
    suggested = DILOVAR_RATE
    per_unit_cost = round(DILOVAR_RATE / units, 2) if units > 0 else 0
    context.user_data["pack_suggested"] = suggested
    context.user_data["pack_per_unit_cost"] = per_unit_cost

    note = ""
    if units < DILOVAR_MIN_UNITS:
        note = f"\n⚠️ Меньше обычной нормы ({DILOVAR_MIN_UNITS} шт) — зарплата за день всё равно полная, но себестоимость труда на 1 шт сегодня выше обычного."

    d = context.user_data
    text = (
        f"📦 {d['pack_date_display']}: {units} шт упаковано.{note}\n\n"
        f"💰 Зарплата за день: {suggested}₽\n"
        f"📊 Себестоимость труда на 1 шт сегодня: {per_unit_cost}₽/шт (вместо обычных ~8₽)"
    )
    await update.message.reply_text(text, reply_markup=ReplyKeyboardMarkup([[f"✅ {suggested}₽"], ["Ввести другую сумму"], ["🔙 Назад", "❌ Главное меню"]], resize_keyboard=True))
    return PACK_CONFIRM


async def pack_confirm(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    if t == "🔙 Назад":
        await update.message.reply_text("Шаг 2: Сколько штук упаковано за этот день (всего)?", reply_markup=get_step_keyboard())
        return PACK_UNITS
    if t == "Ввести другую сумму":
        await update.message.reply_text("Введите сумму цифрами:", reply_markup=get_step_keyboard())
        return PACK_CONFIRM

    db, d = context.bot_data["db"], context.user_data
    if t.startswith("✅"):
        amount = d["pack_suggested"]
    else:
        try: amount = float(t.replace(",", ".").replace("₽", ""))
        except ValueError:
            await update.message.reply_text("⚠️ Нужно число. Попробуйте ещё раз:", reply_markup=get_step_keyboard())
            return PACK_CONFIRM

    category_id = db.get_category_id("Зарплата")
    emp_name = d["pack_employee_name"]
    op_id = db.add_operation_returning_id(
        operation_date=d["pack_date_iso"], ip_id=None, counterparty_id=d["pack_employee_id"],
        category_id=category_id, operation_type="начисление", amount=amount,
        quantity=d["pack_units"], entered_by=f"dilovar_{update.effective_user.id}",
        status="confirmed", payment_method=None,
        comment=f"Упаковка {d['pack_date_display']}: {d['pack_units']} шт ({d.get('pack_per_unit_cost','?')}₽/шт труд), внёс Диловар",
    )
    db.post_journal_entry(operation_id=op_id, entry_date=d["pack_date_iso"], debit_code="5100", credit_code="2100",
                           amount=amount, comment=f"Начисление {emp_name} (упаковка {d['pack_date_display']})")
    await update.message.reply_text(f"✅ Начислено {amount}₽ для {emp_name} за {d['pack_date_display']}.", reply_markup=get_menu_keyboard(update.effective_user.id))
    if ADMIN_NOTIFY_ID:
        await context.bot.send_message(chat_id=ADMIN_NOTIFY_ID, text=f"🕐 Диловар внёс упаковку {emp_name}: {d['pack_units']} шт за {d['pack_date_display']} → начислено {amount}₽")
    return ConversationHandler.END


# --- ОТЧЁТ ДЛЯ ЛЮБОВИ ---
async def report_start(update, context):
    t_now = datetime.now(TZ_MSK)
    kb = [[f"Этот месяц (с {t_now.replace(day=1).strftime('%d.%m')})"], ["Свой период (ДД.ММ-ДД.ММ)"], ["❌ Главное меню"]]
    await update.message.reply_text("За какой период отчёт?", reply_markup=ReplyKeyboardMarkup(kb, resize_keyboard=True))
    return REPORT_PERIOD


async def report_period(update, context):
    t = update.message.text.strip()
    if t == "❌ Главное меню": return await cancel_to_menu(update, context)
    db = context.bot_data["db"]
    t_now = datetime.now(TZ_MSK)

    if "Этот месяц" in t:
        date_from = t_now.replace(day=1).date().isoformat()
        date_to = t_now.date().isoformat()
    else:
        try:
            start_raw, end_raw = t.split("-")
            date_from = parse_flexible_date(start_raw.strip(), TZ_MSK)[0]
            date_to = parse_flexible_date(end_raw.strip(), TZ_MSK)[0]
        except Exception:
            await update.message.reply_text("⚠️ Формат: ДД.ММ-ДД.ММ. Попробуйте ещё раз:", reply_markup=get_step_keyboard())
            return REPORT_PERIOD

    rows = db.get_last_operations_for_period(date_from, date_to)
    if not rows:
        await update.message.reply_text("За этот период записей нет.", reply_markup=get_menu_keyboard(update.effective_user.id))
        return ConversationHandler.END
    lines = [f"📊 Операции {date_from} — {date_to}:\n"]
    for r in rows:
        cp_name = (r.get("counterparties") or {}).get("name", "—")
        lines.append(f"▫️ {r.get('operation_date')} | {r.get('operation_type')} | {cp_name} | {r.get('amount')}₽")
    await update.message.reply_text("\n".join(lines), reply_markup=get_menu_keyboard(update.effective_user.id))
    return ConversationHandler.END


# --- ОКЛАД ЛЮБОВИ ---
async def salary_start(update, context):
    await update.message.reply_text("💵 Начислить оклад 6 000₽?", reply_markup=ReplyKeyboardMarkup([["✅ Подтвердить"], ["❌ Главное меню"]], resize_keyboard=True))
    return SALARY_CONFIRM


async def salary_confirm(update, context):
    t = update.message.text.strip()
    if t != "✅ Подтвердить": return await cancel_to_menu(update, context)
    db = context.bot_data["db"]
    cp = db.get_counterparty_by_name("Любовь")
    category_id = db.get_category_id("Зарплата")
    op_date = datetime.now(TZ_MSK).date().isoformat()
    op_id = db.add_operation_returning_id(
        operation_date=op_date, ip_id=None, counterparty_id=(cp or {}).get("id"),
        category_id=category_id, operation_type="начисление", amount=6000,
        entered_by=f"lyubov_{update.effective_user.id}", status="confirmed",
        payment_method=None, comment="Оклад Любовь",
    )
    db.post_journal_entry(operation_id=op_id, entry_date=op_date, debit_code="5100", credit_code="2100", amount=6000, comment="Начисление оклада Любовь")
    await update.message.reply_text("✅ Начислено 6 000₽.", reply_markup=get_menu_keyboard(update.effective_user.id))
    if ADMIN_NOTIFY_ID:
        await context.bot.send_message(chat_id=ADMIN_NOTIFY_ID, text="💵 Любовь начислила себе оклад 6 000₽.")
    return ConversationHandler.END


def main():
    db_service = SupabaseService()
    application = Application.builder().token(BOT_TOKEN).build()
    application.bot_data["db"] = db_service

    supply_conv = ConversationHandler(
        entry_points=[MessageHandler(filters.Regex("^📦 Расходники$"), supply_start)],
        states={
            SUPPLY_SUPPLIER: [MessageHandler(filters.TEXT & ~filters.COMMAND, supply_supplier)],
            SUPPLY_PRODUCT: [MessageHandler(filters.TEXT & ~filters.COMMAND, supply_product)],
            SUPPLY_QTY: [MessageHandler(filters.TEXT & ~filters.COMMAND, supply_qty)],
            SUPPLY_PRICE: [MessageHandler(filters.TEXT & ~filters.COMMAND, supply_price)],
            SUPPLY_ADD_MORE: [MessageHandler(filters.TEXT & ~filters.COMMAND, supply_add_more)],
            SUPPLY_COMMENT: [MessageHandler(filters.TEXT & ~filters.COMMAND, supply_comment)],
            SUPPLY_CONFIRM: [MessageHandler(filters.TEXT & ~filters.COMMAND, supply_confirm)],
        }, fallbacks=[MessageHandler(filters.Regex(r"^(❌ Главное меню|📦 Расходники|👤 Сотрудники|📊 Отчёт за период|💵 Мой оклад)$"), cancel_to_menu)]
    )

    pack_conv = ConversationHandler(
        entry_points=[MessageHandler(filters.Regex("^👤 Сотрудники$"), employee_menu_start)],
        states={
            EMP_MENU: [MessageHandler(filters.TEXT & ~filters.COMMAND, employee_menu_router)],
            PACK_EMPLOYEE: [MessageHandler(filters.TEXT & ~filters.COMMAND, pack_employee)],
            PACK_DATE: [MessageHandler(filters.TEXT & ~filters.COMMAND, pack_date)],
            PACK_UNITS: [MessageHandler(filters.TEXT & ~filters.COMMAND, pack_units)],
            PACK_CONFIRM: [MessageHandler(filters.TEXT & ~filters.COMMAND, pack_confirm)],
            SHIFT_EMPLOYEE: [MessageHandler(filters.TEXT & ~filters.COMMAND, shift_employee)],
            SHIFT_DATE: [MessageHandler(filters.TEXT & ~filters.COMMAND, shift_date)],
            SHIFT_START: [MessageHandler(filters.TEXT & ~filters.COMMAND, shift_start)],
            SHIFT_END: [MessageHandler(filters.TEXT & ~filters.COMMAND, shift_end)],
            SHIFT_CONFIRM: [MessageHandler(filters.TEXT & ~filters.COMMAND, shift_confirm)],
            ACCRUAL_EMPLOYEE: [MessageHandler(filters.TEXT & ~filters.COMMAND, accrual_employee)],
            ACCRUAL_AMOUNT: [MessageHandler(filters.TEXT & ~filters.COMMAND, accrual_amount)],
            ACCRUAL_CONFIRM: [MessageHandler(filters.TEXT & ~filters.COMMAND, accrual_confirm)],
        }, fallbacks=[MessageHandler(filters.Regex(r"^(❌ Главное меню|📦 Расходники|👤 Сотрудники|📊 Отчёт за период|💵 Мой оклад)$"), cancel_to_menu)]
    )

    report_conv = ConversationHandler(
        entry_points=[MessageHandler(filters.Regex("^📊 Отчёт за период$"), report_start)],
        states={REPORT_PERIOD: [MessageHandler(filters.TEXT & ~filters.COMMAND, report_period)]},
        fallbacks=[MessageHandler(filters.Regex(r"^(❌ Главное меню|📦 Расходники|👤 Сотрудники|📊 Отчёт за период|💵 Мой оклад)$"), cancel_to_menu)]
    )

    salary_conv = ConversationHandler(
        entry_points=[MessageHandler(filters.Regex("^💵 Мой оклад$"), salary_start)],
        states={SALARY_CONFIRM: [MessageHandler(filters.TEXT & ~filters.COMMAND, salary_confirm)]},
        fallbacks=[MessageHandler(filters.Regex(r"^(❌ Главное меню|📦 Расходники|👤 Сотрудники|📊 Отчёт за период|💵 Мой оклад)$"), cancel_to_menu)]
    )

    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("help", help_cmd))
    application.add_handler(supply_conv)
    application.add_handler(pack_conv)
    application.add_handler(report_conv)
    application.add_handler(salary_conv)
    application.add_handler(MessageHandler(filters.Regex("^❓ Помощь$"), help_cmd))

    application.run_polling()


if __name__ == "__main__":
    main()
