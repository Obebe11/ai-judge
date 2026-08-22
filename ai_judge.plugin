from typing import Any, List
import re
import json
import threading
import time

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Input, Text, Switch, Divider, Selector

__id__ = "ai_judge"
__name__ = "ИИ Судья"
__description__ = "Команда `.суд @user1 @user2 50` — вызывает ИИ-судью. Берёт N сообщений ПОСЛЕ реплая, анонимизирует участников и выносит вердикт кто прав. Стелс-мод + тесты."
__author__ = "@you"
__version__ = "1.0.2"
__icon__ = "exteraPlugins/7"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.3.0"
__requirements__ = ["requests"]

DEFAULT_BASE_URL = "https://api.openai.com/v1"
DEFAULT_MODEL = "gpt-4o-mini"
DEFAULT_LIMIT = 50
DEFAULT_PROMPT = """Ты — беспристрастный ИИ-Судья для чатов Telegram.

Твоя задача: разобраться кто прав в споре, опираясь ТОЛЬКО на предоставленный транскрипт и общеизвестные факты. Ты НЕ знаешь реальные имена участников — они анонимизированы как Сторона A, Сторона B и т.д. Суди строго по аргументам, логике и фактам, а не по личностям.

Если в споре есть проверяемые утверждения (даты, факты, научные тезисы, юридические нормы), укажи какие источники/тип источников подтверждают позицию.

Верни СТРОГО JSON без markdown:
{
  "winner": "Сторона A" | "Сторона B" | "Ничья" | "Недостаточно данных",
  "confidence": 0-100,
  "verdict": "1-2 предложения: кто прав и почему, кратко",
  "reasoning": "подробный разбор по пунктам: аргументы каждой стороны, логические ошибки, фактические ошибки",
  "facts": ["список проверяемых фактов с указанием кто что утверждал и верно ли это"],
  "sources": ["список источников/типов источников для проверки, если применимо"],
  "advice": "совет как закрыть спор"
}
"""

# ------------ helpers outside class ------------

def extract_mentions(text: str):
    return re.findall(r"@([A-Za-z0-9_]{4,32})", text)

def parse_sud_command(text: str):
    """
    Форматы:
      .суд @user1 @user2 50
      .суд 50
      .суд @user1 20
      .суд
    Возвращает (mentions: list[str], limit: int, raw: str)
    """
    t = text.strip()
    # убираем префикс .суд / .court / /суд
    m = re.match(r"^[\.\/!](суд|court|judge)\b\s*(.*)$", t, re.IGNORECASE | re.DOTALL)
    if not m:
        return None
    rest = m.group(2).strip()
    # limit — число в конце (5-200)
    limit = DEFAULT_LIMIT
    lm = re.search(r"\b(\d{1,3})\b\s*$", rest)
    if lm:
        try:
            v = int(lm.group(1))
            if 3 <= v <= 200:
                limit = v
                rest = rest[:lm.start()].strip()
        except:
            pass
    mentions = extract_mentions(rest)
    return mentions, limit, rest


class AIJudgePlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()
        self.log("ИИ Судья загружен. Команда: .суд @user1 @user2 50 (ответом на сообщение)")

    def _on_test_fake_click(self, view=None):
        try:
            from ui.bulletin import BulletinHelper
            BulletinHelper.show_info("🧪 Запускаю тестовый суд...")
        except:
            pass
        self.log("Тест: фейковый срач запущен из настроек")
        try:
            from client_utils import run_on_queue, PLUGINS_QUEUE, get_selected_account
            # пробуем взять текущий аккаунт
            try:
                acc = get_selected_account()
            except:
                acc = 0
            run_on_queue(lambda: self._run_fake_test(acc), PLUGINS_QUEUE)
        except Exception:
            from client_utils import run_on_queue
            run_on_queue(lambda: self._run_fake_test(0))

    def _on_test_llm_click(self, view=None):
        try:
            from ui.bulletin import BulletinHelper
            BulletinHelper.show_info("🔍 Проверяю LLM...")
        except:
            pass
        self.log("Тест: проверка LLM запущена из настроек")
        try:
            from client_utils import run_on_queue, PLUGINS_QUEUE, get_selected_account
            try:
                acc = get_selected_account()
            except:
                acc = 0
            run_on_queue(lambda: self._run_llm_ping(acc), PLUGINS_QUEUE)
        except Exception:
            from client_utils import run_on_queue
            run_on_queue(lambda: self._run_llm_ping(0))

    def create_settings(self) -> List[Any]:
        return [
            Header(text="ИИ Судья — настройки"),
            Text(text="Команда: ответь на сообщение в чате `.суд @юзер1 @юзер2 50` — бот возьмёт 50 сообщений ПОСЛЕ реплая, анонимизирует и вынесет вердикт.", subtext="Работает только когда ты отправляешь команду", icon="msg_info"),
            Text(text="🔒 Анонимизация включена всегда", subtext="Судья НЕ видит реальные ники (Сторона A/B). Отключить нельзя — для честности", icon="msg_secret", red=False),
            Divider(),
            Header(text="LLM API (OpenAI-совместимый)"),
            Input(key="api_key", text="API ключ", default="", subtext="Ключ от OpenAI / OpenRouter / др. Хранится локально", icon="msg_key"),
            Input(key="base_url", text="Base URL", default=DEFAULT_BASE_URL, subtext="Например https://api.openai.com/v1 или https://openrouter.ai/api/v1", icon="msg_link"),
            Input(key="model", text="Модель", default=DEFAULT_MODEL, subtext="gpt-4o-mini, gpt-4o, deepseek-chat, claude-3.5 и т.д.", icon="msg_bot"),
            Input(key="default_limit", text="Лимит по умолчанию", default=str(DEFAULT_LIMIT), subtext="Сколько сообщений брать если число не указано (3-200)", icon="msg_edit"),
            Divider(),
            Header(text="Промпт судьи"),
            Input(key="system_prompt", text="Системный промпт (оставь пустым = дефолт)", default="", subtext="Если заполнишь — заменит дефолтный. Используй {transcript} не нужно — он подставится автоматически.", icon="msg_text"),
            Switch(key="show_transcript", text="Прикладывать анонимизированный транскрипт к вердикту", default=False, subtext="Полезно для отладки", icon="msg_view_file"),
            Switch(key="stealth_mode", text="Стелс-мод (в Избранное)", default=False, subtext="Вердикт уходит в Избранное, а не в чат. Для теста без палева", icon="msg_secret"),
            Input(key="stealth_peer", text="Куда слать в стелсе (оставь пусто = Избранное)", default="", subtext="ID диалога или @username. Пусто = Saved Messages", icon="msg_forward"),
            Divider(),
            Header(text="Тестирование (без чата)"),
            Text(text="🧪 Тестовый суд — фейковый срач", subtext="Запустит суд на синтетических данных, проверит LLM", icon="msg_info", on_click=self._on_test_fake_click),
            Text(text="🔍 Проверить LLM (пинг)", subtext="Отправит тестовый запрос к API и покажет ответ в Избранном", icon="msg_settings", on_click=self._on_test_llm_click),
            Text(text="Команды в чате", subtext=".суд тест — фейковый срач в чате\n.суд пинг — проверка LLM\n.суд 5 (ответом) — реальный суд", icon="msg_text"),
        ]

    # ---------- hooks ----------
    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        try:
            msg = getattr(params, "message", None)
            if not isinstance(msg, str):
                return HookResult()
            raw = msg.strip()
            if not re.match(r"^[\.\/!](суд|court|judge)\b", raw, re.IGNORECASE):
                return HookResult()

            # --- тестовые команды (работают без реплая) ---
            low = raw.lower().strip()
            # .суд тест / .судтест / .суд test / .court test / .суд пинг / .суд ping
            if low in (".суд тест", ".судтест", ".суд test", ".court test", ".judge test", "/суд тест", "!суд тест"):
                self.log(f"Тестовая команда: {raw} account={account}")
                # peer для ответа — текущий чат если есть, иначе Избранное
                test_peer = getattr(params, "peer", None) or getattr(params, "dialog_id", None)
                if test_peer is None:
                    try:
                        from client_utils import get_last_fragment
                        frag = get_last_fragment(account)
                        if frag:
                            test_peer = frag.getDialogId() if hasattr(frag, "getDialogId") else None
                    except:
                        pass
                # если не нашли — уйдёт в Избранное внутри _run_fake_test
                try:
                    from client_utils import run_on_queue, PLUGINS_QUEUE
                    run_on_queue(lambda: self._run_fake_test(account, test_peer), PLUGINS_QUEUE)
                except:
                    from client_utils import run_on_queue
                    run_on_queue(lambda: self._run_fake_test(account, test_peer))
                return HookResult(strategy=HookStrategy.CANCEL)

            if low in (".суд пинг", ".суд ping", ".court ping", ".judge ping", "/суд пинг"):
                self.log(f"Пинг LLM: {raw} account={account}")
                ping_peer = getattr(params, "peer", None) or getattr(params, "dialog_id", None)
                if ping_peer is None:
                    try:
                        from client_utils import get_last_fragment
                        frag = get_last_fragment(account)
                        if frag:
                            ping_peer = frag.getDialogId() if hasattr(frag, "getDialogId") else None
                    except:
                        pass
                try:
                    from client_utils import run_on_queue, PLUGINS_QUEUE
                    run_on_queue(lambda: self._run_llm_ping(account, ping_peer), PLUGINS_QUEUE)
                except:
                    from client_utils import run_on_queue
                    run_on_queue(lambda: self._run_llm_ping(account, ping_peer))
                return HookResult(strategy=HookStrategy.CANCEL)

            parsed = parse_sud_command(raw)
            if not parsed:
                return HookResult()
            mentions, limit, _rest = parsed

            # дефолт лимит из настроек
            try:
                dl = int(self.get_setting("default_limit", str(DEFAULT_LIMIT)))
                if 3 <= dl <= 200:
                    # если юзер не указал число явно — используем дефолт
                    # parse уже выставил DEFAULT_LIMIT, но если в тексте не было числа — заменим
                    if not re.search(r"\b\d{1,3}\b\s*$", raw.strip()):
                        limit = dl
            except:
                pass

            # peer и reply id
            peer_id = getattr(params, "peer", None)
            # в некоторых версиях поле называется dialog_id / peerId
            if peer_id is None:
                peer_id = getattr(params, "dialog_id", None)
            if peer_id is None:
                peer_id = getattr(params, "peerId", None)

            reply_id = getattr(params, "replyToMsg", None)
            if reply_id is None:
                reply_id = getattr(params, "reply_to_msg_id", None)
            if reply_id is None:
                # попробуем достать из replyTo
                rt = getattr(params, "replyTo", None)
                if rt is not None:
                    reply_id = getattr(rt, "reply_to_msg_id", None) or getattr(rt, "message_id", None)

            # если peer не смогли взять из params — попробуем из фрагмента
            if peer_id is None:
                try:
                    from client_utils import get_last_fragment
                    frag = get_last_fragment(account)
                    if frag:
                        peer_id = frag.getDialogId() if hasattr(frag, "getDialogId") else None
                        if peer_id is None and hasattr(frag, "getDialog_id"):
                            peer_id = frag.getDialog_id()
                except Exception as e:
                    self.log(f"peer fallback failed: {e}")

            if peer_id is None:
                # не смогли определить чат — модифицируем сообщение с подсказкой
                params.message = "❗️ ИИ Судья: не удалось определить чат. Отправь .суд ответом на сообщение в группе."
                return HookResult(strategy=HookStrategy.MODIFY, params=params)

            if not reply_id or int(reply_id) == 0:
                params.message = "❗️ ИИ Судья: отправь .суд <b>ответом</b> на сообщение, ПОСЛЕ которого начинается срач.\nПример: ответь на сообщение → `.суд @user1 @user2 50`"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)

            api_key = (self.get_setting("api_key", "") or "").strip()
            if not api_key:
                params.message = "❗️ ИИ Судья: укажи API ключ в настройках плагина (Настройки → Плагины → ИИ Судья)"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)

            # Отменяем отправку команды, запускаем суд в фоне
            # Сначала скажем что суд собирается (через очередь)
            self.log(f"Суд вызван: peer={peer_id} reply={reply_id} mentions={mentions} limit={limit} account={account}")

            # Запускаем в фоне, чтобы не морозить UI
            try:
                from client_utils import run_on_queue, PLUGINS_QUEUE
                run_on_queue(lambda: self._run_court(account, peer_id, int(reply_id), mentions, limit), PLUGINS_QUEUE)
            except Exception:
                from client_utils import run_on_queue
                run_on_queue(lambda: self._run_court(account, peer_id, int(reply_id), mentions, limit))

            return HookResult(strategy=HookStrategy.CANCEL)

        except Exception as e:
            self.log(f"on_send_message_hook error: {e}")
            return HookResult()

    def _get_saved_peer(self, account: int):
        """Peer для Избранного = свой user_id"""
        # 1) пробуем через AccountClient
        try:
            c = self.client(account)
            uc = c.get_user_config()
            me = uc.getCurrentUser()
            if me:
                # у TLRPC.User поле id
                uid = getattr(me, "id", None) or getattr(me, "user_id", None)
                if uid:
                    return int(uid)
        except Exception as e:
            self.log(f"get_saved_peer via client failed: {e}")
        # 2) пробуем через client_utils напрямую
        try:
            from client_utils import get_user_config
            uc = get_user_config(account)
            me = uc.getCurrentUser()
            if me:
                uid = getattr(me, "id", None) or getattr(me, "user_id", None)
                if uid:
                    return int(uid)
        except Exception as e:
            self.log(f"get_saved_peer via utils failed: {e}")
        return None

    def _resolve_stealth_target(self, account: int, original_peer: int):
        """Куда слать в стелс-моде: настройка stealth_peer > Избранное > оригинал"""
        raw = (self.get_setting("stealth_peer", "") or "").strip()
        if raw:
            # если @username — пробуем резолвить
            if raw.startswith("@"):
                raw = raw[1:]
            # если число — считаем peer_id
            try:
                if raw.lstrip("-").isdigit():
                    return int(raw)
            except:
                pass
            # пытаемся резолвить username через API (синхронно)
            try:
                from client_utils import send_request
                from org.telegram.tgnet import TLRPC
                ev = threading.Event()
                holder = {}
                def cb(resp, err):
                    holder["resp"] = resp
                    holder["err"] = err
                    ev.set()
                req = TLRPC.TL_contacts_resolveUsername()
                req.username = raw
                try:
                    send_request(req, cb, account=account)
                except TypeError:
                    send_request(req, cb)
                ev.wait(8)
                resp = holder.get("resp")
                if resp:
                    # может быть chat или user
                    chats = getattr(resp, "chats", []) or []
                    users = getattr(resp, "users", []) or []
                    if chats:
                        ch = chats[0]
                        cid = getattr(ch, "id", None)
                        if cid:
                            # channel peer = -100... + cid
                            return -1000000000000 - int(cid)
                    if users:
                        uid = getattr(users[0], "id", None)
                        if uid:
                            return int(uid)
            except Exception as e:
                self.log(f"stealth resolve @{raw} failed: {e}")
        # дефолт — Избранное
        sp = self._get_saved_peer(account)
        return sp if sp is not None else original_peer

    # ---------- тестирование ----------
    FAKE_MESSAGES = [
        {"id": 1001, "from_id": 101, "from_name": "Тестер А", "text": "Земля плоская, я видел видос на ютубе где horizon ровный", "date": 0},
        {"id": 1002, "from_id": 202, "from_name": "Тестер Б", "text": "Земля — почти сфера. Это измерял ещё Эратосфен в 240 году до н.э., подтверждается спутниками и кругосветками", "date": 0},
        {"id": 1003, "from_id": 101, "from_name": "Тестер А", "text": "Но NASA врет, все фото Земли — фотошоп, я не верю", "date": 0},
        {"id": 1004, "from_id": 202, "from_name": "Тестер Б", "text": "Фотошоп не объясняет тень Земли на Луне при лунном затмении — она всегда круглая. И GPS не работал бы на плоской", "date": 0},
        {"id": 1005, "from_id": 101, "from_name": "Тестер А", "text": "А как тогда Boeing летает по прямой если Земля круглая? Должен же падать", "date": 0},
        {"id": 1006, "from_id": 202, "from_name": "Тестер Б", "text": "Самолёт летит по геодезической — кратчайший путь на сфере, это учитывает инерциальная система и притяжение", "date": 0},
    ]

    def _run_fake_test(self, account: int, peer_hint=None):
        """Синтетический суд без реального чата — проверка всего пайплайна"""
        is_stealth = bool(self.get_setting("stealth_mode", False))
        # куда слать: peer_hint = чат где написали .суд тест; если стелс — переводим в Избранное как в _run_court
        if peer_hint is not None:
            if is_stealth:
                target = self._resolve_stealth_target(account, peer_hint)
            else:
                target = peer_hint
        else:
            target = self._get_saved_peer(account)
            is_stealth = True  # без чата (кнопка в настройках) — только в Избранное
        if target is None:
            target = self._get_saved_peer(account) or 0
            if target == 0:
                try:
                    from ui.bulletin import BulletinHelper
                    BulletinHelper.show_info("❌ Тест: не найден peer для отправки. Открой Избранное и попробуй снова")
                except:
                    pass
                self.log("Fake test: no target peer")
                return

        self.log(f"🧪 Fake test start account={account} target={target}")
        self._safe_send(account, target, "🧪 <b>Тестовый суд запущен</b>\nПроверяю анонимизацию + LLM…\n<i>Синтетический срач: плоская vs круглая Земля</i>", None, parse_mode="HTML")

        messages = list(self.FAKE_MESSAGES)
        # анонимизация ВСЕГДА включена (нельзя выключить)
        anon_transcript, mapping, reverse, participants = self._anonymize(messages, [])
        self.log(f"Fake anonymized: {mapping} -> {participants}")

        # проверка что судья не видит имена
        anon_check = "Тестер" not in anon_transcript and "Сторона" in anon_transcript
        self._safe_send(account, target, f"🔒 Анонимизация: <b>{'OK' if anon_check else 'FAIL'}</b>\nСудья видит:\n<blockquote expandable>{anon_transcript.replace('<','&lt;')}</blockquote>\nУчастники: {', '.join(participants)}", None, parse_mode="HTML")
        if not anon_check:
            self._safe_send(account, target, "❌ <b>Критическая ошибка:</b> реальные имена утекли в транскрипт!", None, parse_mode="HTML")
            return

        verdict_json, raw = self._call_llm(anon_transcript, participants)
        if verdict_json is None:
            self._safe_send(account, target, f"❌ <b>LLM не ответил / вернул не JSON</b>\nПроверь API ключ, Base URL, модель в настройках.\n\nОтвет:\n<code>{(raw or 'пусто')[:1200].replace('<','&lt;')}</code>\n\nЛог: смотри adb logcat | grep 'ИИ Суд'", None, parse_mode="HTML")
            return

        # форматируем как реальный вердикт, но с пометкой ТЕСТ
        fake_peer =  -100999999  # фейковый id для шапки
        final = self._format_verdict(verdict_json, raw, mapping, reverse, participants, messages, anon_transcript, len(messages), fake_peer, 999)
        final = "🧪 <b>ТЕСТОВЫЙ ВЕРДИКТ (синтетика)</b> — плагин работает!\n<i>Это фейковый срач для проверки пайплайна. Реальные ники были скрыты.</i>\n\n" + final
        self._safe_send(account, target, final, None, parse_mode="HTML")
        self.log("Fake test done")

    def _run_llm_ping(self, account: int, peer_hint=None):
        is_stealth = bool(self.get_setting("stealth_mode", False))
        if peer_hint is not None and is_stealth:
            target = self._resolve_stealth_target(account, peer_hint)
        elif peer_hint is not None:
            target = peer_hint
        else:
            target = self._get_saved_peer(account)
        if target is None:
            target = self._get_saved_peer(account) or 0
        if target == 0:
            self.log("LLM ping: no target")
            return
        api_key = (self.get_setting("api_key", "") or "").strip()
        base_url = (self.get_setting("base_url", DEFAULT_BASE_URL) or DEFAULT_BASE_URL).strip()
        model = (self.get_setting("model", DEFAULT_MODEL) or DEFAULT_MODEL).strip()
        if not api_key:
            self._safe_send(account, target, "❌ <b>LLM пинг: нет API ключа</b>\nЗаполни в Настройки → Плагины → ИИ Судья", None, parse_mode="HTML")
            return
        self._safe_send(account, target, f"🔍 <b>Пинг LLM</b> → <code>{base_url}</code>\nМодель: <code>{model}</code>\nОтправляю тестовый запрос…", None, parse_mode="HTML")
        # минимальный промпт
        try:
            import requests, json as js
            url = base_url.rstrip("/") + "/chat/completions"
            headers = {"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"}
            payload = {"model": model, "messages": [{"role": "user", "content": "Ответь одним словом: pong"}], "temperature": 0}
            r = requests.post(url, headers=headers, json=payload, timeout=18)
            body = r.text[:1500]
            if r.status_code == 200:
                try:
                    j = r.json()
                    ans = j["choices"][0]["message"]["content"]
                    self._safe_send(account, target, f"✅ <b>LLM OK</b> ({r.status_code})\nМодель ответила: <code>{str(ans)[:500].replace('<','&lt;')}</code>\n\nМожешь запускать <code>.суд тест</code>", None, parse_mode="HTML")
                except Exception as e:
                    self._safe_send(account, target, f"⚠️ LLM вернул 200 но не распарсился:\n<code>{body.replace('<','&lt;')}</code>", None, parse_mode="HTML")
            else:
                self._safe_send(account, target, f"❌ <b>LLM ошибка HTTP {r.status_code}</b>\n<code>{body.replace('<','&lt;')}</code>\nПроверь ключи/URL/модель", None, parse_mode="HTML")
        except Exception as e:
            self._safe_send(account, target, f"❌ <b>LLM пинг упал</b>: {str(e)[:1000].replace('<','&lt;')}", None, parse_mode="HTML")
            self.log(f"LLM ping err {e}")

    # ---------- core logic ----------
    def _run_court(self, account: int, peer_id: int, reply_msg_id: int, mentions: List[str], limit: int):
        stealth = bool(self.get_setting("stealth_mode", False))
        # куда слать ответы
        target_peer = peer_id
        target_reply = reply_msg_id
        stealth_info = ""
        if stealth:
            resolved = self._resolve_stealth_target(account, peer_id)
            if resolved is not None:
                target_peer = resolved
                target_reply = None  # в Избранном нет треда исходного чата
                stealth_info = f" (стелс: из чата {peer_id} #{reply_msg_id})"
            # тихо подтверждаем в логах + буллетень
            try:
                from ui.bulletin import BulletinHelper
                BulletinHelper.show_info(f"ИИ Суд: стелс-мод → вердикт уйдёт в Избранное{stealth_info}")
            except:
                pass
            self.log(f"Стелс-мод активен: original={peer_id} -> target={target_peer}")

        # 1. Сообщим что суд созывается
        self._send_status(account, target_peer, f"🏛 <b>ИИ Суд созывается…</b>\nСобираю {limit} сообщений после #{reply_msg_id}{stealth_info}…", target_reply)

        # 2. Соберём сообщения (всегда из оригинального чата)
        messages = self._fetch_history(account, peer_id, reply_msg_id, limit, mentions)
        if not messages:
            self._send_status(account, target_peer, f"❌ ИИ Суд: не удалось собрать сообщения после #{reply_msg_id} в чате {peer_id}. Возможно нет новых сообщений или нет доступа к истории.{stealth_info}", target_reply)
            return

        # 3. Анонимизация
        anon_transcript, mapping, reverse_mapping, participants_human = self._anonymize(messages, mentions)

        # 4. Вызов LLM
        self._send_status(account, target_peer, f"🔍 Собрано {len(messages)} сообщений. Судья изучает доводы…{stealth_info}", target_reply)

        verdict_json, raw_llm = self._call_llm(anon_transcript, participants_human)

        # 5. Форматируем вердикт с подстановкой реальных ников через макросы
        final_text = self._format_verdict(verdict_json, raw_llm, mapping, reverse_mapping, participants_human, messages, anon_transcript, limit, peer_id, reply_msg_id)
        if stealth:
            # добавляем шапку откуда вердикт
            stealth_header = f"🕵️ <i>Стелс-мод: вердикт из чата <code>{peer_id}</code> реплай #{reply_msg_id} → показан только тебе (в Избранном)</i>\n"
            # если указан кастомный peer — покажем
            custom = (self.get_setting("stealth_peer", "") or "").strip()
            if custom:
                stealth_header += f"Кастомный получатель: <code>{custom}</code> (target {target_peer})\n"
            final_text = stealth_header + "\n" + final_text

        self._safe_send(account, target_peer, final_text, target_reply, parse_mode="HTML")

    def _fetch_history(self, account: int, peer_id: int, reply_msg_id: int, limit: int, mentions: List[str]):
        """
        Берёт сообщения ПОСЛЕ reply_msg_id. Пытается через TL_messages_getHistory с min_id.
        Делает несколько попыток разными способами т.к. TL схема может отличаться.
        Возвращает список dict {id, from_id, from_name, text, date}
        """
        import time
        messages = []
        # пробуем через send_request + TLRPC
        try:
            from client_utils import send_request
            try:
                from org.telegram.tgnet import TLRPC
            except Exception as e:
                self.log(f"TLRPC import failed: {e}")
                TLRPC = None

            if TLRPC is None:
                return self._fetch_history_via_storage(account, peer_id, reply_msg_id, limit)

            # резолвим InputPeer
            input_peer = self._get_input_peer(account, peer_id)
            if input_peer is None:
                self.log("input_peer is None, fallback to storage")
                return self._fetch_history_via_storage(account, peer_id, reply_msg_id, limit)

            # синхронный запрос helper
            def sync_req(req):
                ev = threading.Event()
                holder = {}
                def cb(resp, err):
                    holder["resp"] = resp
                    holder["err"] = err
                    ev.set()
                try:
                    # пробуем с account=
                    try:
                        send_request(req, cb, account=account)
                    except TypeError:
                        # старая сигнатура без account
                        send_request(req, cb)
                except Exception as e:
                    holder["err"] = e
                    ev.set()
                ev.wait(12)
                return holder.get("resp"), holder.get("err")

            # резолв юзеров если упомянуты — чтобы фильтровать
            wanted_ids = set()
            if mentions:
                for uname in mentions:
                    uid = self._resolve_username_sync(account, uname, sync_req, TLRPC)
                    if uid:
                        wanted_ids.add(uid)
                self.log(f"mentions {mentions} -> ids {wanted_ids}")

            # Фетчим — делаем батчами, т.к. нужен фильтр по юзерам и min_id
            fetched = []
            # TL_messages_getHistory параметры: peer, offset_id, offset_date, add_offset, limit, max_id, min_id, hash
            # Хотим сообщения новее reply_msg_id -> min_id = reply_msg_id, offset_id = 0, limit = limit+50
            tries = 0
            offset_id = 0
            max_id = 0
            min_id = reply_msg_id
            need = limit + 20  # запас для фильтра
            while len(fetched) < need and tries < 4:
                tries += 1
                req = TLRPC.TL_messages_getHistory()
                req.peer = input_peer
                req.offset_id = offset_id
                req.offset_date = 0
                req.add_offset = 0
                req.limit = min(100, need - len(fetched) + 10)
                req.max_id = max_id
                req.min_id = min_id
                req.hash = 0
                resp, err = sync_req(req)
                if err:
                    self.log(f"getHistory err: {err}")
                    break
                if not resp:
                    break
                batch = getattr(resp, "messages", None) or []
                users = getattr(resp, "users", None) or []
                chats = getattr(resp, "chats", None) or []
                # мап юзеров для имён
                user_map = {}
                for u in users:
                    try:
                        uid = getattr(u, "id", None) or getattr(u, "user_id", None)
                        fname = getattr(u, "first_name", "") or ""
                        lname = getattr(u, "last_name", "") or ""
                        uname = getattr(u, "username", "") or ""
                        disp = (fname + " " + lname).strip() or uname or f"User{uid}"
                        if uid:
                            user_map[uid] = disp
                    except:
                        pass
                # парсим сообщения
                if not batch:
                    break
                batch_parsed = []
                for m in batch:
                    try:
                        mid = getattr(m, "id", 0)
                        # skip service / empty?
                        if mid <= reply_msg_id:
                            continue
                        # from_id
                        fid = None
                        from_id_obj = getattr(m, "from_id", None)
                        if from_id_obj is not None:
                            fid = getattr(from_id_obj, "user_id", None)
                            if fid is None:
                                fid = getattr(from_id_obj, "userId", None)
                            if fid is None and hasattr(from_id_obj, "chat_id"):
                                fid = -getattr(from_id_obj, "chat_id")
                        if fid is None:
                            fid = getattr(m, "fromId", None)
                        # peer_id может быть channel
                        text = getattr(m, "message", "") or ""
                        if not text:
                            text = getattr(m, "caption", "") or ""
                        # пропускаем пустые/сервисные
                        if not text and getattr(m, "media", None) is None:
                            # allow but mark
                            text = ""
                        # дата
                        date = getattr(m, "date", 0)
                        # имя
                        from_name = user_map.get(fid, f"ID:{fid}" if fid else "Unknown")
                        # если это мы сами — попробуем взять из UserConfig
                        batch_parsed.append({"id": mid, "from_id": fid, "from_name": from_name, "text": text, "date": date, "raw": m})
                    except Exception as e:
                        self.log(f"parse msg err {e}")
                        continue
                # Telegram отдаёт новейшие сначала, нам нужны хронологически после reply -> reverse
                # но batch уже отсортирован от новых к старым, фильтруем и позже отсортируем
                fetched.extend(batch_parsed)
                if len(batch) < req.limit:
                    break
                # для следующей пагинации — двигаем offset_id к самому старому из батча
                try:
                    oldest = min(getattr(x, "id", 0) for x in batch)
                    if oldest == offset_id:
                        break
                    offset_id = oldest
                    # если уже ушли ниже min_id — стоп
                    if oldest <= reply_msg_id:
                        break
                except:
                    break
                time.sleep(0.2)

            # сортировка по id возрастанию (хронология)
            fetched.sort(key=lambda x: x["id"])
            # фильтр по участникам если указаны
            if wanted_ids:
                filtered = [m for m in fetched if m["from_id"] in wanted_ids]
                # если после фильтра меньше limit — берём что есть, но пробуем добрать?
                # если filtered слишком мало — возьмём всё с пометкой
                if len(filtered) >= max(1, limit // 3):
                    fetched = filtered[:limit]
                else:
                    # недостаточно сообщений от указанных — берём все, но предупредим позже
                    fetched = filtered[:limit] if filtered else fetched[:limit]
            else:
                fetched = fetched[:limit]

            # если всё ещё пусто — fallback
            if not fetched:
                self.log("fetched empty, fallback storage")
                return self._fetch_history_via_storage(account, peer_id, reply_msg_id, limit)

            return fetched

        except Exception as e:
            self.log(f"_fetch_history err: {e}")
            try:
                return self._fetch_history_via_storage(account, peer_id, reply_msg_id, limit)
            except Exception as e2:
                self.log(f"fallback also failed: {e2}")
                return []

    def _get_input_peer(self, account: int, peer_id: int):
        try:
            # через client
            c = self.client(account)
            mc = c.get_messages_controller()
            # пробуем разные имена
            for meth in ["getInputPeer", "getInputPeerById", "getPeer"]:
                if hasattr(mc, meth):
                    try:
                        ip = getattr(mc, meth)(peer_id)
                        if ip:
                            return ip
                    except Exception as e:
                        continue
        except Exception as e:
            self.log(f"getInputPeer via client failed: {e}")

        # ручной конструкт через TLRPC
        try:
            from org.telegram.tgnet import TLRPC
            # Определим тип peer_id
            # peer_id >0 = user, peer_id <0 и > -1000000000000 = chat, иначе channel
            # Но для простоты попробуем TL_inputPeerUser / Channel / Chat
            if peer_id > 0:
                ip = TLRPC.TL_inputPeerUser()
                ip.user_id = peer_id
                # access_hash нужен, без него запрос может не пройти — поэтому лучше через controller
                # попробуем поставить 0
                if hasattr(ip, "access_hash"):
                    ip.access_hash = 0
                return ip
            elif peer_id < 0:
                # channel id в dialog_id хранится как -100 + id
                # peer_id = -1000000000000 - channelId? Точная формула: dialog_id = -1000000000000 - channel_id для каналов?
                # Попробуем восстановить
                if peer_id <= -1000000000000:
                    channel_id = -peer_id - 1000000000000
                    ip = TLRPC.TL_inputPeerChannel()
                    ip.channel_id = channel_id
                    if hasattr(ip, "access_hash"):
                        ip.access_hash = 0
                    return ip
                else:
                    chat_id = -peer_id
                    ip = TLRPC.TL_inputPeerChat()
                    ip.chat_id = chat_id
                    return ip
        except Exception as e:
            self.log(f"manual InputPeer failed: {e}")
        return None

    def _resolve_username_sync(self, account, username, sync_req, TLRPC):
        try:
            req = TLRPC.TL_contacts_resolveUsername()
            req.username = username
            resp, err = sync_req(req)
            if err or not resp:
                return None
            # resp.users contains user
            users = getattr(resp, "users", []) or []
            for u in users:
                un = getattr(u, "username", "") or ""
                if un.lower() == username.lower():
                    return getattr(u, "id", None)
            if users:
                return getattr(users[0], "id", None)
        except Exception as e:
            self.log(f"resolve @{username} err {e}")
        return None

    def _fetch_history_via_storage(self, account, peer_id, reply_msg_id, limit):
        """ Fallback через MessagesStorage (локальная БД) """
        try:
            c = self.client(account)
            storage = c.get_messages_storage()
            # storage.getMessages(peer_id, ...) — но сигнатура неизвестна, пробуем
            # Альтернатива: MessagesController.getMessages
            # Попробуем storage.query?
            # Самый простой fallback — вернуть пусто и сообщить юзеру что не получилось через API
            self.log("storage fallback not implemented fully, returning []")
            return []
        except Exception as e:
            self.log(f"storage fallback err {e}")
            return []

    def _anonymize(self, messages, mentions):
        """
        Мапит реальных юзеров -> Сторона A, B, C...
        Возвращает (transcript_text, mapping, reverse, participants_human)
        """
        # соберём уникальных юзеров в порядке появления
        uniq = []
        seen = set()
        for m in messages:
            fid = m.get("from_id")
            if fid is not None and fid not in seen:
                seen.add(fid)
                uniq.append(fid)
        # буквы A,B,C...
        letters = [chr(ord("A")+i) for i in range(26)]
        anon_names = {}
        reverse = {}
        for i, fid in enumerate(uniq):
            name = f"Сторона {letters[i % len(letters)]}"
            # если участников 2 — можно A/B, если больше — нумерация
            anon_names[fid] = name
            reverse[name] = fid

        # если были @mentions — отметим кто есть кто в human списке
        participants_human = []
        for fid in uniq:
            anon = anon_names[fid]
            # найдём имя
            disp = next((m["from_name"] for m in messages if m["from_id"]==fid), str(fid))
            # найдём username из mentions если совпал?
            participants_human.append(f"{anon} ({disp})")

        # строим транскрипт
        lines = []
        for m in messages:
            fid = m.get("from_id")
            anon = anon_names.get(fid, "Наблюдатель")
            text = (m.get("text") or "").strip()
            if not text:
                text = "[медиа без текста]"
            # обрежем слишком длинные
            if len(text) > 800:
                text = text[:800] + "…"
            mid = m.get("id")
            lines.append(f"[{anon}]: {text}")

        transcript = "\n".join(lines)
        # маппинг fid -> anon для подстановки в вердикт
        mapping = anon_names  # fid -> anon
        return transcript, mapping, reverse, participants_human

    def _call_llm(self, transcript, participants_human):
        api_key = (self.get_setting("api_key", "") or "").strip()
        base_url = (self.get_setting("base_url", DEFAULT_BASE_URL) or DEFAULT_BASE_URL).strip().rstrip("/")
        model = (self.get_setting("model", DEFAULT_MODEL) or DEFAULT_MODEL).strip()
        system_prompt = (self.get_setting("system_prompt", "") or "").strip()
        if not system_prompt:
            system_prompt = DEFAULT_PROMPT

        # Добавим участников в системный промпт? Нет, в юзер
        user_content = f"Участники (анонимизированы): {', '.join(participants_human) if participants_human else 'неизвестны'}\n\nТранскрипт срача (хронология, {len(transcript.splitlines())} сообщений ПОСЛЕ указанного):\n{transcript}\n\nВерни JSON как описано. Суди честно. Если недостаточно данных — так и скажи."

        payload = {
            "model": model,
            "messages": [
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_content}
            ],
            "temperature": 0.2,
        }

        url = base_url + "/chat/completions"
        headers = {"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"}

        raw = ""
        verdict = None
        try:
            import requests
            r = requests.post(url, headers=headers, json=payload, timeout=60)
            raw = r.text
            if r.status_code != 200:
                self.log(f"LLM http {r.status_code}: {raw[:500]}")
                return None, f"HTTP {r.status_code}: {raw[:800]}"
            data = r.json()
            content = data["choices"][0]["message"]["content"]
            raw = content
            # извлечь JSON
            verdict = self._extract_json(content)
            if verdict is None:
                # попробуем распарсить весь content как json
                verdict = json.loads(content)
            return verdict, content
        except Exception as e:
            self.log(f"LLM call err: {e} raw={raw[:500]}")
            return None, raw or str(e)

    def _extract_json(self, text):
        # ищем ```json ... ``` или { ... }
        try:
            # strip markdown fences
            m = re.search(r"```(?:json)?\s*(\{.*?\})\s*```", text, re.DOTALL)
            if m:
                return json.loads(m.group(1))
            # найдём первый { и последний }
            start = text.find("{")
            end = text.rfind("}")
            if start != -1 and end != -1 and end > start:
                j = text[start:end+1]
                return json.loads(j)
        except Exception as e:
            self.log(f"extract_json fail: {e}")
        return None

    def _format_verdict(self, verdict, raw_llm, mapping, reverse, participants_human, messages, transcript, limit, peer_id, reply_id):
        # мап anon -> реальное имя для подстановки макросов
        # построим anon -> display
        anon_to_display = {}
        for fid, anon in mapping.items():
            disp = next((m["from_name"] for m in messages if m["from_id"]==fid), str(fid))
            # найдём username если есть
            # display как @? используем имя
            anon_to_display[anon] = disp

        def deanon(text):
            if not isinstance(text, str):
                return text
            for anon, disp in anon_to_display.items():
                text = text.replace(anon, f"{anon} ({disp})")
            return text

        header = "🏛 <b>ИИ СУД — ВЕРДИКТ</b>"
        anon_note = "⚖️ <i>Судья не знал реальные ники — судил анонимно по аргументам</i>"
        meta = f"💬 Сообщений разобрано: <b>{len(messages)}</b> (запрошено {limit}) • реплай #{reply_id}"

        participants_line = ""
        if anon_to_display:
            parts = [f"{anon} → <b>{disp}</b>" for anon, disp in anon_to_display.items()]
            participants_line = "👥 Участники: " + " | ".join(parts)

        if verdict and isinstance(verdict, dict):
            winner_anon = verdict.get("winner", "—")
            winner_human = deanon(str(winner_anon))
            confidence = verdict.get("confidence", "")
            verdict_short = deanon(verdict.get("verdict", ""))
            reasoning = deanon(verdict.get("reasoning", ""))
            facts = verdict.get("facts", [])
            sources = verdict.get("sources", [])
            advice = deanon(verdict.get("advice", ""))

            # winner подсветка
            if "Сторона" in str(winner_anon):
                winner_line = f"🏆 <b>Победитель:</b> {winner_human}"
            elif "ничья" in str(winner_anon).lower():
                winner_line = f"🤝 <b>Вердикт:</b> Ничья"
            elif "недостаточно" in str(winner_anon).lower():
                winner_line = f"❓ <b>Вердикт:</b> Недостаточно данных"
            else:
                winner_line = f"🏆 <b>Вердикт:</b> {winner_human}"

            conf_line = f" (уверенность {confidence}%)" if isinstance(confidence, int) or (isinstance(confidence, str) and confidence.isdigit()) else ""

            body = f"{header}\n{anon_note}\n{meta}\n{participants_line}\n\n{winner_line}{conf_line}\n\n<b>Кратко:</b> {verdict_short}\n\n<b>Разбор:</b>\n{reasoning}"

            if facts:
                facts_s = "\n".join([f"• {deanon(str(x))}" for x in facts[:6]])
                body += f"\n\n<b>Факты:</b>\n{facts_s}"
            if sources:
                src_s = "\n".join([f"• {str(x)}" for x in sources[:6]])
                body += f"\n\n<b>Источники для проверки:</b>\n{src_s}"
            if advice:
                body += f"\n\n<b>Совет:</b> {advice}"

            # макросы для копирования — подсказка
            body += f"\n\n<i>Макросы: {{{{winner}}}}={winner_human}, {{{{count}}}}={len(messages)}</i>"

            if self.get_setting("show_transcript", False):
                # экранируем HTML в транскрипте?
                safe_trans = transcript.replace("<", "&lt;").replace(">", "&gt;")
                body += f"\n\n<blockquote expandable>Анонимизированный транскрипт:\n{safe_trans}</blockquote>"

            return body
        else:
            # fallback — показываем сырой ответ LLM
            safe_raw = (raw_llm or "нет ответа").replace("<", "&lt;").replace(">", "&gt;")
            # обрежем
            if len(safe_raw) > 3500:
                safe_raw = safe_raw[:3500] + "…"
            participants_line = participants_line or ""
            return f"{header}\n{anon_note}\n{meta}\n{participants_line}\n\n⚠️ Судья вернул не-JSON, показываю как есть:\n\n{safe_raw}"

    def _send_status(self, account, peer_id, text, reply_id):
        self._safe_send(account, peer_id, text, reply_id, parse_mode="HTML")

    def _safe_send(self, account, peer_id, text, reply_id=None, parse_mode="HTML"):
        try:
            # SDK >=1.4.5 имеет self.client(account); на старых — сразу fallback
            try:
                c = self.client(account)
            except Exception as e:
                c = None

            if c is not None:
                kwargs = {}
                if reply_id:
                    kwargs["replyToMsg"] = int(reply_id)
                try:
                    c.send_text(peer_id, text, parse_mode=parse_mode, **kwargs)
                    return
                except TypeError:
                    try:
                        c.send_text(peer_id, text, **kwargs)
                        return
                    except Exception:
                        pass
                except Exception as e:
                    self.log(f"client.send_text failed: {e}")

            # fallback через глобальную функцию
            from client_utils import send_text
            try:
                send_text(peer_id, text, parse_mode=parse_mode, account=account, replyToMsg=int(reply_id) if reply_id else None)
                return
            except TypeError:
                send_text(peer_id, text, parse_mode=parse_mode, replyToMsg=int(reply_id) if reply_id else None)
                return
            except Exception as e2:
                self.log(f"send_text fallback err: {e2}")
                # последняя попытка без parse_mode
                send_text(peer_id, text, account=account)

        except Exception as e:
            self.log(f"_safe_send err: {e}")

