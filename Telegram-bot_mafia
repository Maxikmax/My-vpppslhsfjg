"""
Telegram Mafia Bot — single-file implementation
Requires: python-telegram-bot==20.7  (pip install python-telegram-bot==20.7)
Run: BOT_TOKEN=xxx python mafia_bot.py
"""

import asyncio
import logging
import os
import random
from dataclasses import dataclass, field
from enum import Enum
from typing import Dict, List, Optional

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler, ContextTypes,
)

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger("mafia")

# ---------- Модель ----------

class Role(Enum):
    MAFIA = "Мафия"
    DOCTOR = "Доктор"
    SHERIFF = "Шериф"
    CIVILIAN = "Мирный"

class Phase(Enum):
    LOBBY = "lobby"
    NIGHT = "night"
    DAY = "day"
    VOTE = "vote"
    ENDED = "ended"

@dataclass
class Player:
    user_id: int
    name: str
    role: Optional[Role] = None
    alive: bool = True

@dataclass
class Game:
    chat_id: int
    host_id: int
    players: Dict[int, Player] = field(default_factory=dict)
    phase: Phase = Phase.LOBBY
    day_no: int = 0
    # ночные действия
    mafia_target: Optional[int] = None
    doctor_target: Optional[int] = None
    sheriff_target: Optional[int] = None
    last_protected: Optional[int] = None
    # голосование
    votes: Dict[int, int] = field(default_factory=dict)  # voter -> target
    pending_night_actors: set = field(default_factory=set)

GAMES: Dict[int, Game] = {}  # chat_id -> Game

MIN_PLAYERS = 4

# ---------- Утилиты ----------

def alive_players(g: Game) -> List[Player]:
    return [p for p in g.players.values() if p.alive]

def alive_mafia(g: Game) -> List[Player]:
    return [p for p in alive\_players(g) if p.role == Role.MAFIA]

def alive_civ(g: Game) -> List[Player]:
    return [p for p in alive\_players(g) if p.role != Role.MAFIA]

def fmt_list(players: List[Player]) -> str:
    return "\n".join(f"• {p.name}" for p in players) or "—"

def assign_roles(g: Game):
    ids = list(g.players.keys())
    random.shuffle(ids)
    n = len(ids)
    n_maf = max(1, n // 4)
    roles = [Role.MAFIA] * n_maf + [Role.DOCTOR, Role.SHERIFF]
    roles += [Role.CIVILIAN] * (n - len(roles))
    roles = roles[:n]
    random.shuffle(roles)
    for uid, r in zip(ids, roles):
        g.players[uid].role = r

def check_winner(g: Game) -> Optional[str]:
    m = len(alive_mafia(g))
    c = len(alive_civ(g))
    if m == 0:
        return "🏆 Победа мирных жителей!"
    if m >= c:
        return "🔪 Победа мафии!"
    return None

# ---------- Хэндлеры ----------

async def cmd_start(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Привет! Я бот для игры в Мафию.\n\n"
        "Команды в групповом чате:\n"
        "/newgame — создать игру\n"
        "/join — присоединиться\n"
        "/startgame — начать (мин. 4 игрока)\n"
        "/players — список игроков\n"
        "/stop — отменить игру\n\n"
        "⚠️ Сначала напишите мне /start в личку, иначе я не смогу прислать вам роль."
    )

async def cmd_newgame(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    chat = update.effective_chat
    if chat.type == "private":
        await update.message.reply_text("Создавай игру в групповом чате.")
        return
    if chat.id in GAMES and GAMES[chat.id].phase != Phase.ENDED:
        await update.message.reply_text("Игра уже идёт. /stop чтобы отменить.")
        return
    g = Game(chat_id=chat.id, host_id=update.effective_user.id)
    GAMES[chat.id] = g
    await update.message.reply_text(
        "🎲 Новая игра в Мафию!\nЖмите /join чтобы присоединиться.\n"
        f"Минимум игроков: {MIN_PLAYERS}.\n"
        "Когда все собрались — /startgame"
    )

async def cmd_join(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    chat = update.effective_chat
    user = update.effective_user
    g = GAMES.get(chat.id)
    if not g or g.phase != Phase.LOBBY:
        await update.message.reply_text("Сейчас нельзя присоединиться. /newgame")
        return
    if user.id in g.players:
        await update.message.reply_text("Ты уже в игре.")
        return
    # проверим, что бот может писать в личку
    try:
        await ctx.bot.send_message(user.id, "✅ Ты присоединился к игре в Мафию.")
    except Exception:
        await update.message.reply_text(
            f"{user.first_name}, сначала напиши мне в личку /start, потом /join."
        )
        return
    g.players[user.id] = Player(user_id=user.id, name=user.first_name)
    await update.message.reply_text(
        f"➕ {user.first_name} в игре. Игроков: {len(g.players)}"
    )

async def cmd_players(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    g = GAMES.get(update.effective_chat.id)
    if not g:
        await update.message.reply_text("Игры нет.")
        return
    await update.message.reply_text("Игроки:\n" + fmt_list(list(g.players.values())))

async def cmd_stop(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    g = GAMES.get(update.effective_chat.id)
    if not g:
        return
    if update.effective_user.id != g.host_id:
        await update.message.reply_text("Остановить может только создатель игры.")
        return
    g.phase = Phase.ENDED
    GAMES.pop(g.chat_id, None)
    await update.message.reply_text("🛑 Игра остановлена.")

async def cmd_startgame(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    g = GAMES.get(update.effective_chat.id)
    if not g or g.phase != Phase.LOBBY:
        return
    if len(g.players) < MIN_PLAYERS:
        await update.message.reply_text(f"Нужно минимум {MIN_PLAYERS} игроков.")
        return
    assign_roles(g)
    for p in g.players.values():
        try:
            await ctx.bot.send_message(p.user_id, f"🎭 Твоя роль: *{p.role.value}*", parse_mode="Markdown")
        except Exception:
            pass
    mafs = alive_mafia(g)
    if len(mafs) > 1:
        names = ", ".join(m.name for m in mafs)
        for m in mafs:
            try:
                await ctx.bot.send_message(m.user_id, f"🔪 Твои напарники: {names}")
            except Exception:
                pass
    await update.message.reply_text(
        f"🌃 Игра началась! Игроков: {len(g.players)}\n"
        f"Мафии: {len(mafs)}\nНаступает ночь..."
    )
    asyncio.create_task(night_phase(ctx, g))

# ---------- Фазы ----------

def target_keyboard(g: Game, action: str, exclude: List[int] = None) -> InlineKeyboardMarkup:
    exclude = exclude or []
    btns = []
    for p in alive_players(g):
        if p.user_id in exclude:
            continue
        btns.append([InlineKeyboardButton(p.name, callback_data=f"{action}:{g.chat_id}:{p.user_id}")])
    btns.append([InlineKeyboardButton("Пропустить", callback_data=f"{action}:{g.chat_id}:0")])
    return InlineKeyboardMarkup(btns)

async def night_phase(ctx: ContextTypes.DEFAULT_TYPE, g: Game):
    g.phase = Phase.NIGHT
    g.day_no += 1
    g.mafia_target = None
    g.doctor_target = None
    g.sheriff_target = None
    g.pending_night_actors = set()

    await ctx.bot.send_message(g.chat_id, f"🌙 Ночь {g.day_no}. Город засыпает...")

    # Мафия
    for m in alive_mafia(g):
        g.pending_night_actors.add(m.user_id)
        await ctx.bot.send_message(
            m.user_id, "Кого убить этой ночью?",
            reply_markup=target_keyboard(g, "maf", exclude=[mm.user_id for mm in alive_mafia(g)])
        )
    # Доктор
    for d in [p for p in alive\_players(g) if p.role == Role.DOCTOR]:
        g.pending_night_actors.add(d.user_id)
        excl = [g.last_protected] if g.last_protected else []
        await ctx.bot.send_message(
            d.user_id, "Кого спасти? (нельзя того же что и прошлой ночью)",
            reply_markup=target_keyboard(g, "doc", exclude=excl)
        )
    # Шериф
    for s in [p for p in alive\_players(g) if p.role == Role.SHERIFF]:
        g.pending_night_actors.add(s.user_id)
        await ctx.bot.send_message(
            s.user_id, "Кого проверить?",
            reply_markup=target_keyboard(g, "shr", exclude=[s.user_id])
        )

    # таймаут ночи 60 секунд
    for _ in range(60):
        await asyncio.sleep(1)
        if not g.pending_night_actors:
            break

    await resolve_night(ctx, g)

async def resolve_night(ctx: ContextTypes.DEFAULT_TYPE, g: Game):
    killed_msg = ""
    if g.mafia_target and g.mafia_target != g.doctor_target:
        victim = g.players.get(g.mafia_target)
        if victim and victim.alive:
            victim.alive = False
            killed_msg = f"☠️ Этой ночью убит: *{victim.name}* (роль: {victim.role.value})"
    if not killed_msg:
        killed_msg = "🌅 Этой ночью никто не погиб."
    g.last_protected = g.doctor_target

    await ctx.bot.send_message(g.chat_id, f"☀️ Утро {g.day_no}.\n{killed_msg}", parse_mode="Markdown")

    win = check_winner(g)
    if win:
        await end_game(ctx, g, win)
        return

    await day_phase(ctx, g)

async def day_phase(ctx: ContextTypes.DEFAULT_TYPE, g: Game):
    g.phase = Phase.VOTE
    g.votes = {}
    alive = alive_players(g)
    btns = [[InlineKeyboardButton(p.name, callback_data=f"vote:{g.chat_id}:{p.user_id}")] for p in alive]
    btns.append([InlineKeyboardButton("Воздержаться", callback_data=f"vote:{g.chat_id}:0")])
    await ctx.bot.send_message(
        g.chat_id,
        f"🗳 Голосование. Живые игроки:\n{fmt_list(alive)}\n\nУ вас 60 секунд.",
        reply_markup=InlineKeyboardMarkup(btns),
    )
    for _ in range(60):
        await asyncio.sleep(1)
        if len(g.votes) >= len(alive):
            break
    await resolve_vote(ctx, g)

async def resolve_vote(ctx: ContextTypes.DEFAULT_TYPE, g: Game):
    counts: Dict[int, int] = {}
    for t in g.votes.values():
        if t == 0: continue
        counts[t] = counts.get(t, 0) + 1
    if not counts:
        await ctx.bot.send_message(g.chat_id, "Никого не повесили.")
    else:
        mx = max(counts.values())
        leaders = [uid for uid, c in counts.items() if c == mx]
        if len(leaders) > 1:
            await ctx.bot.send_message(g.chat_id, "🤝 Ничья — никого не повесили.")
        else:
            victim = g.players[leaders[0]]
            victim.alive = False
            await ctx.bot.send_message(
                g.chat_id,
                f"⚖️ Повешен: *{victim.name}* (роль: {victim.role.value})",
                parse_mode="Markdown",
            )
    win = check_winner(g)
    if win:
        await end_game(ctx, g, win)
        return
    await asyncio.sleep(2)
    asyncio.create_task(night_phase(ctx, g))

async def end_game(ctx: ContextTypes.DEFAULT_TYPE, g: Game, msg: str):
    g.phase = Phase.ENDED
    roster = "\n".join(f"• {p.name} — {p.role.value} {'💀' if not p.alive else ''}"
                       for p in g.players.values())
    await ctx.bot.send_message(g.chat_id, f"{msg}\n\nИтог:\n{roster}")
    GAMES.pop(g.chat_id, None)

# ---------- Колбэки ----------

async def on_callback(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    try:
        action, chat_id_s, target_s = q.data.split(":")
        chat_id = int(chat_id_s); target = int(target_s)
    except Exception:
        return
    g = GAMES.get(chat_id)
    if not g: return
    uid = q.from_user.id
    player = g.players.get(uid)
    if not player or not player.alive:
        await q.edit_message_text("Ты не можешь действовать.")
        return

    if action == "maf" and player.role == Role.MAFIA and g.phase == Phase.NIGHT:
        g.mafia_target = target or None
        g.pending_night_actors.discard(uid)
        name = g.players[target].name if target else "никого"
        await q.edit_message_text(f"🔪 Цель мафии: {name}")
    elif action == "doc" and player.role == Role.DOCTOR and g.phase == Phase.NIGHT:
        g.doctor_target = target or None
        g.pending_night_actors.discard(uid)
        name = g.players[target].name if target else "никого"
        await q.edit_message_text(f"💉 Спасаешь: {name}")
    elif action == "shr" and player.role == Role.SHERIFF and g.phase == Phase.NIGHT:
        g.sheriff_target = target or None
        g.pending_night_actors.discard(uid)
        if target:
            t = g.players[target]
            verdict = "МАФИЯ 🔪" if t.role == Role.MAFIA else "мирный 😇"
            await q.edit_message_text(f"🔍 {t.name} — {verdict}")
        else:
            await q.edit_message_text("🔍 Пропустил проверку.")
    elif action == "vote" and g.phase == Phase.VOTE:
        g.votes[uid] = target
        name = g.players[target].name if target else "воздержался"
        await q.edit_message_text(f"Твой голос: {name}")

# ---------- main ----------

def main():
    token = os.environ.get("BOT_TOKEN")
    if not token:
        raise SystemExit("Установи BOT_TOKEN в окружении")
    app = Application.builder().token(token).build()
    app.add_handler(CommandHandler("start", cmd_start))
    app.add_handler(CommandHandler("newgame", cmd_newgame))
    app.add_handler(CommandHandler("join", cmd_join))
    app.add_handler(CommandHandler("startgame", cmd_startgame))
    app.add_handler(CommandHandler("players", cmd_players))
    app.add_handler(CommandHandler("stop", cmd_stop))
    app.add_handler(CallbackQueryHandler(on_callback))
    log.info("Bot started")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
