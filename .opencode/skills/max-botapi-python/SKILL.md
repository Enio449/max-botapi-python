---
name: max-botapi-python
description: Помогает проектировать, писать и ревьюить Python-ботов для мессенджера MAX на maxapi/max-botapi-python и obabot: polling, webhook, события, фильтры, клавиатуры, callback, MemoryContext/FSM, медиа и миграция aiogram-кода.
license: MIT
compatibility: opencode
metadata:
  language: ru
  ecosystem: python
  platform: max-messenger
---

# MAX Bot API Python Skill

## Использованные знания

Этот skill основан на локальной wiki репозитория `max-botapi-python` (`wiki/bot_methods.md`, `wiki/events.md`, `wiki/handlers.md`, `wiki/keyboard.md`, `wiki/memory_context.md`, `wiki/webhook.md`), актуальном формате OpenCode `SKILL.md`, общих паттернах MAX Bot API и публичной информации о библиотеке `obabot`.

## Назначение

Используй этот skill, когда нужно написать, изменить, проверить или объяснить Python-бота для мессенджера MAX. Skill покрывает два основных пути:

1. **`maxapi` / `max-botapi-python`** — нативный стиль библиотеки MAX: `Bot`, `Dispatcher`, события `message_created`, `message_callback`, `bot_started`, методы `bot.send_message(...)`, `event.message.answer(...)`, клавиатуры через `InlineKeyboardBuilder` или `ButtonsPayload`.
2. **`obabot`** — универсальная async-библиотека для Telegram и MAX с aiogram-совместимым API: `create_bot(...)`, `router.message(...)`, `Command`, `F`, `FSMContext`, `message.answer(...)` и единый код для Telegram/MAX.

Всегда пиши пояснения, сообщения об ошибках в примерах и комментарии в коде на русском языке, если пользователь явно не попросил другой язык.

## Быстрый выбор библиотеки

### Когда выбирать `maxapi`

Выбирай `maxapi`, если:

- бот предназначен только для MAX;
- нужны нативные события MAX (`bot_added`, `dialog_muted`, `message_chat_created`, `user_removed` и т.д.);
- важны нативные типы MAX, вложения и методы управления чатами;
- проект уже использует импорты вида `from maxapi import Bot, Dispatcher`.

### Когда выбирать `obabot`

Выбирай `obabot`, если:

- нужно один раз написать бота и запускать его в Telegram, MAX или сразу на обеих платформах;
- пользователь мигрирует существующий aiogram 3.x код;
- нужны привычные aiogram-декораторы, фильтры, callback query и FSM;
- проект уже использует `from obabot import create_bot`.

### Что уточнить перед реализацией

Если задача неоднозначна, сначала уточни:

- бот только для MAX или также для Telegram;
- нужен polling или webhook;
- где хранится токен (`MAX_TOKEN`, `.env`, секреты CI/CD);
- нужны ли FSM/память, callback-кнопки, медиа, админские методы;
- есть ли ограничения хостинга для webhook: домен, HTTPS, порт, reverse proxy.

## Базовый шаблон `maxapi` с polling

Используй этот шаблон как основу для MAX-only бота:

```python
import asyncio
import logging
import os

from maxapi import Bot, Dispatcher
from maxapi.filters import Command, F
from maxapi.types import BotStarted, MessageCreated

logging.basicConfig(level=logging.INFO)

bot = Bot(os.environ["MAX_TOKEN"])
dp = Dispatcher()


@dp.bot_started()
async def on_bot_started(event: BotStarted):
    await event.bot.send_message(
        chat_id=event.chat_id,
        text="Привет! Отправь /start, чтобы начать работу.",
    )


@dp.message_created(Command("start"))
async def on_start(event: MessageCreated):
    await event.message.answer("Привет! Я бот для MAX.")


@dp.message_created(F.message.body.text)
async def echo(event: MessageCreated):
    text = event.message.body.text
    await event.message.answer(f"Вы написали: {text}")


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
```

Правила для `maxapi`:

- хендлеры всегда делай `async def`;
- для ответа на входящее сообщение предпочитай `await event.message.answer(...)`;
- для произвольной отправки используй `await bot.send_message(chat_id=..., text=...)` или `user_id=...`;
- не храни токен в коде, используй переменные окружения;
- если бот тестируется в групповом чате, напомни пользователю выдать боту права администратора;
- если ранее был настроен webhook, перед polling может понадобиться удалить webhook через метод библиотеки, если он доступен в используемой версии.

## Базовый шаблон `maxapi` с webhook

Высокоуровневый вариант:

```python
import asyncio
import logging
import os

from maxapi import Bot, Dispatcher
from maxapi.types import MessageCreated

logging.basicConfig(level=logging.INFO)

bot = Bot(os.environ["MAX_TOKEN"])
dp = Dispatcher()


@dp.message_created()
async def handle_message(event: MessageCreated):
    await event.message.answer("Бот работает через webhook.")


async def main():
    await dp.handle_webhook(
        bot=bot,
        host="0.0.0.0",
        port=8080,
        log_level="info",
    )


if __name__ == "__main__":
    asyncio.run(main())
```

Для webhook:

- установи зависимости webhook-режима, например `pip install maxapi[webhook]`, если проект использует extra библиотеки;
- используй `handle_webhook(...)`, когда не нужны дополнительные FastAPI routes;
- используй `init_serve(...)` и `@dp.webhook_post(...)`, когда нужен ручной контроль над обработчиком запроса;
- webhook endpoint должен быстро возвращать успешный ответ, а тяжёлые задачи лучше выносить в фоновые очереди;
- не логируй токены, персональные данные и полные payload без необходимости.

## События и хендлеры `maxapi`

Основные события:

- `message_created` — новое сообщение;
- `message_edited` — сообщение изменено;
- `message_removed` — сообщение удалено;
- `message_callback` — пользователь нажал callback-кнопку;
- `bot_started` — пользователь запустил бота;
- `bot_stopped` — пользователь остановил бота;
- `bot_added` / `bot_removed` — бот добавлен или удалён из чата;
- `user_added` / `user_removed` — пользователь добавлен или удалён;
- `chat_title_changed` — изменено название чата;
- `dialog_cleared`, `dialog_muted`, `dialog_unmuted` — изменения диалога;
- `message_chat_created` — пользователь нажал кнопку создания чата; учитывай, что поведение этого события может зависеть от текущего состояния API MAX;
- `on_started` — внутреннее событие запуска библиотеки.

Общий синтаксис:

```python
@dp.message_created(F.message.body.text)
async def handle_text(event: MessageCreated):
    await event.message.answer(f"Повторяю: {event.message.body.text}")
```

Рекомендации:

- регистрируй более специфичные хендлеры выше общих;
- для команд используй `Command("start")`, `Command("help")`;
- для текстовых сообщений проверяй наличие текста через `F.message.body.text`;
- для callback-кнопок используй отдельный `@dp.message_callback(...)`;
- типизируй параметр `event`, чтобы агенту и IDE было проще подсказывать поля.

## Методы бота `maxapi`

Частые методы:

- `send_message(chat_id=..., user_id=..., text=..., attachments=..., link=..., notify=..., parse_mode=...)` — отправить сообщение;
- `edit_message(message_id=..., text=..., attachments=..., link=..., notify=..., parse_mode=...)` — изменить сообщение;
- `delete_message(message_id)` — удалить сообщение;
- `get_messages(chat_id=..., message_ids=..., from_time=..., to_time=..., count=...)` — получить список сообщений;
- `get_message(message_id)` — получить одно сообщение;
- `pin_message(chat_id=..., message_id=..., notify=...)` и `delete_pin_message(chat_id)` — закрепы;
- `get_me()`, `change_info(...)`, `set_my_commands(...)` — профиль и команды бота;
- `get_chats(...)`, `get_chat_by_id(...)`, `get_chat_by_link(...)`, `edit_chat(...)`, `delete_chat(...)` — чаты;
- `get_chat_members(...)`, `get_chat_member(...)`, `add_chat_members(...)`, `kick_chat_member(...)` — участники;
- `get_list_admin_chat(...)`, `add_list_admin_chat(...)`, `remove_admin(...)`, `get_me_from_chat(...)`, `delete_me_from_chat(...)` — администрирование;
- `get_updates()` — получить обновления вручную;
- `send_action(chat_id=..., action=...)` — показать действие, например набор текста;
- `send_callback(callback_id=..., message=..., notification=...)` — ответить на callback;
- `get_upload_url(type=...)`, `get_video(video_token)` — медиа.

При генерации кода проверяй реальные сигнатуры в установленной версии библиотеки, если репозиторий содержит исходники или lock-файл.

## Клавиатуры `maxapi`

### InlineKeyboardBuilder

```python
from maxapi.filters import Command
from maxapi.types import CallbackButton, LinkButton, MessageCreated
from maxapi.utils.inline_keyboard import InlineKeyboardBuilder


@dp.message_created(Command("menu"))
async def show_menu(event: MessageCreated):
    builder = InlineKeyboardBuilder()
    builder.row(
        CallbackButton(text="Профиль", payload="profile"),
        LinkButton(text="Сайт", url="https://example.com"),
    )

    await event.message.answer(
        text="Выберите действие:",
        attachments=[builder.as_markup()],
    )
```

### Pydantic-модели

```python
from maxapi.types import ButtonsPayload, CallbackButton, LinkButton, MessageCreated


@dp.message_created(Command("links"))
async def show_links(event: MessageCreated):
    buttons = [
        [LinkButton(text="Документация", url="https://example.com/docs")],
        [CallbackButton(text="Помощь", payload="help")],
    ]
    payload = ButtonsPayload(buttons=buttons).pack()
    await event.message.answer("Полезные ссылки:", attachments=[payload])
```

Типы кнопок:

- `LinkButton` — открыть ссылку;
- `CallbackButton` — отправить callback payload;
- `ChatButton` — создать чат;
- `RequestGeoLocationButton` — запросить геолокацию, но учитывай возможные ограничения API MAX;
- `MessageButton` — быстро отправить сообщение;
- `RequestContactButton` — запросить контакт;
- `OpenAppButton` — открыть встроенное приложение.

## Callback-кнопки `maxapi`

Пример обработки:

```python
from maxapi.types import MessageCallback


@dp.message_callback()
async def handle_callback(event: MessageCallback):
    payload = event.callback.payload

    if payload == "profile":
        await event.bot.send_callback(
            callback_id=event.callback.id,
            message="Открываю профиль.",
            notification="Готово",
        )
        await event.message.answer("Ваш профиль пока пуст.")
        return

    await event.bot.send_callback(
        callback_id=event.callback.id,
        message="Неизвестное действие.",
        notification="Ошибка",
    )
```

При callback:

- всегда валидируй `payload`;
- не доверяй данным из payload для прав доступа;
- отвечай на callback, чтобы пользователь получил обратную связь;
- для сложных payload используй компактный формат и серверное состояние, а не большие JSON-строки в кнопке.

## MemoryContext и состояние в `maxapi`

`MemoryContext` подходит для простого хранения данных и состояния пользователя в рамках сессии.

```python
from maxapi.filters import Command
from maxapi.middlewares.context import MemoryContext
from maxapi.types import MessageCreated


@dp.message_created(Command("clear"))
async def clear_context(event: MessageCreated, context: MemoryContext):
    await context.clear()
    await event.message.answer("Контекст очищен.")


@dp.message_created(Command("data"))
async def show_data(event: MessageCreated, context: MemoryContext):
    data = await context.get_data()
    await event.message.answer(f"Текущие данные: {data}")
```

Методы:

- `get_data()` — получить словарь данных;
- `set_data(data)` — заменить данные;
- `update_data(**kwargs)` — обновить данные;
- `set_state(state=None)` — установить или сбросить состояние;
- `get_state()` — получить состояние;
- `clear()` — очистить данные и состояние.

Учти особенность `message_chat_created`: в этом событии `chat_id` может указывать на созданный чат, а `user_id` — на бота.

## Медиа и вложения `maxapi`

Общий подход:

1. Получи URL загрузки через `get_upload_url(type=...)`.
2. Загрузи файл по полученному URL и токену способом, который поддерживает текущая версия библиотеки.
3. Сформируй attachment payload нужного типа.
4. Передай вложение в `attachments=[...]` у `send_message(...)` или `event.message.answer(...)`.

Правила:

- проверяй лимиты размера и MIME-типы в актуальной документации API MAX;
- для пользовательских файлов валидируй расширение и размер до загрузки;
- не сохраняй временные файлы дольше необходимого;
- не используй устаревший `download_file(...)`, если в проекте доступен более актуальный upload flow.

## Шаблон `obabot` для MAX

```python
import asyncio
import logging
import os

from obabot import create_bot
from obabot.filters import Command, F

logging.basicConfig(level=logging.INFO)

bot, dp, router = create_bot(max_token=os.environ["MAX_TOKEN"])


@router.message(Command("start"))
async def start(message):
    await message.answer(f"Привет с платформы {message.platform}!")


@router.message(F.text)
async def echo(message):
    await message.answer(f"Вы написали: {message.text}")


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
```

## Шаблон `obabot` для двух платформ

```python
import asyncio
import logging
import os

from obabot import create_bot
from obabot.filters import Command

logging.basicConfig(level=logging.INFO)

bot, dp, router = create_bot(
    tg_token=os.environ.get("TG_TOKEN"),
    max_token=os.environ.get("MAX_TOKEN"),
)


@router.message(Command("start"))
async def start(message):
    await message.answer(f"Привет! Сообщение пришло из {message.platform.upper()}.")


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
```

Правила для `obabot`:

- `create_bot(...)` возвращает `(bot, dp, router)`;
- передавай `max_token` для MAX, `tg_token` для Telegram или оба токена для dual-platform режима;
- для тестов можно использовать `test_mode=True` или переменную `TESTING=1`, если это поддерживает установленная версия;
- используй `router.message(...)` как рекомендуемый стиль, `dp.message(...)` допустим для простых случаев;
- `message.platform` показывает источник сообщения: `telegram` или `max`;
- `message.text`, `message.from_user`, `message.chat`, `message.message_id`, `answer(...)`, `reply(...)`, `delete(...)`, `edit_text(...)` должны восприниматься как aiogram-совместимый слой;
- для MAX-зависимых возможностей проверяй, поддерживает ли адаптер obabot нужный тип события или attachment.

## FSM в `obabot`

```python
from obabot.filters import Command
from obabot.fsm import FSMContext, State, StatesGroup


class Form(StatesGroup):
    name = State()
    age = State()


@router.message(Command("start"))
async def start_form(message, state: FSMContext):
    await state.set_state(Form.name)
    await message.answer("Как вас зовут?")


@router.message(Form.name)
async def process_name(message, state: FSMContext):
    await state.update_data(name=message.text)
    await state.set_state(Form.age)
    await message.answer("Сколько вам лет?")


@router.message(Form.age)
async def process_age(message, state: FSMContext):
    data = await state.update_data(age=message.text)
    await state.clear()
    await message.answer(f"Спасибо! Данные сохранены: {data}")
```

## Клавиатуры в `obabot`

```python
from obabot.types import InlineKeyboardButton, InlineKeyboardMarkup

keyboard = InlineKeyboardMarkup(
    inline_keyboard=[
        [
            InlineKeyboardButton(text="Профиль", callback_data="profile"),
            InlineKeyboardButton(text="Помощь", callback_data="help"),
        ]
    ]
)

await message.answer("Выберите действие:", reply_markup=keyboard)
```

Callback в `obabot`:

```python
from obabot.filters import F


@router.callback_query(F.data == "profile")
async def show_profile(callback):
    await callback.answer("Открываю профиль.")
    await callback.message.edit_text("Профиль пока пуст.")
```

## Миграция aiogram → obabot

Минимальная миграция:

1. Замени `from aiogram import Bot, Dispatcher, Router` на `from obabot import create_bot`.
2. Замени фильтры и FSM на `from obabot.filters import ...` и `from obabot.fsm import ...`.
3. Вместо ручной инициализации `Bot(...)`, `Dispatcher()`, `Router()` используй `bot, dp, router = create_bot(...)`.
4. Удали `dp.include_router(router)`, если `create_bot(...)` уже возвращает подключённый router в текущей версии.
5. Оставь хендлеры и aiogram-подобные методы без изменений, если адаптер их поддерживает.
6. Проверь места, где используются Telegram-only типы: файлы, медиа, callback, deep links, payments, web apps.

## Проверочный список качества

Перед завершением задачи проверь:

- код асинхронный и запускается через `asyncio.run(main())`;
- токены читаются из окружения или секретов, а не захардкожены;
- polling и webhook не запускаются одновременно без явной причины;
- callback payload валидируется;
- ошибки сетевых вызовов не скрываются без логирования;
- сообщения пользователю написаны на русском языке, если проект русскоязычный;
- хендлеры не перекрывают друг друга случайным порядком;
- для webhook указан понятный host/port и есть инструкция по зависимостям;
- есть хотя бы минимальная команда `/start` или `/help`;
- для групповых чатов учтены права администратора бота;
- для тестов не нужны реальные токены или сетевые вызовы, если можно использовать test mode/mocks.

## Частые ошибки

- Использовать `message.text` в `maxapi` вместо `event.message.body.text` без адаптера.
- Использовать `attachments=` в `obabot`, где aiogram-совместимый слой ожидает `reply_markup=` для inline-клавиатуры.
- Путать `CallbackButton(payload=...)` в `maxapi` и `InlineKeyboardButton(callback_data=...)` в `obabot`.
- Не отвечать на callback-запрос.
- Запускать polling при активном webhook.
- Добавлять blocking I/O в хендлер без `asyncio.to_thread(...)` или очереди.
- Логировать токен бота или полный webhook payload с персональными данными.

## Как отвечать пользователю

Когда пользователь просит код:

- укажи выбранную библиотеку и почему;
- дай полный минимальный пример, который можно запустить;
- перечисли команды установки;
- явно укажи переменные окружения;
- если есть риск несовместимости версии, попроси проверить установленную версию или предложи команду проверки.

Когда пользователь просит исправить проект:

- сначала найди текущую библиотеку по импортам;
- не смешивай `maxapi` и `obabot` без причины;
- меняй минимально необходимый набор файлов;
- добавляй тесты или smoke-check там, где это возможно;
- не добавляй реальные токены, URL webhook с секретами или персональные данные.
