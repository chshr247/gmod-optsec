---
name: gmod-optsec
description: Garry's Mod server/addon performance optimization and security hardening. Use whenever the user writes, reviews, or debugs GLua code that touches hot paths (HUDPaint, Think, Tick, render hooks, timers, net messages), complains about FPS drops, server lag, GC hitches, or asks about GMod security — net message validation, exploits, backdoors, rate limiting, DDoS protection, anticheat. Trigger on mentions of "оптимизация", "фпс", "лаги", "худ", "hook", "timer", "net", "безопасность", "эксплойт" in a GMod context, even if the user doesn't say "optimization" explicitly. Complements the glua skill: glua gives exact API signatures, this skill gives the performance/security rules.
---

# GMod Optimization & Security

Правила производительности и безопасности для GLua. Полное руководство с
примерами кода — в `references/optimization_and_security.md` (на русском):
читай нужный раздел при работе над соответствующей темой, не пересказывай
по памяти.

**Точные сигнатуры API, реалмы, хуки, енумы** — это зона glua-скилла
(`/glua:glua`): не угадывай сигнатуры, сверяйся с его `references/`.

## Реалм-модель (минимум, который нужен всегда)

- `SERVER` / `CLIENT` — два раздельных Lua-состояния, память не общая.
  "Shared" — файл, загруженный в оба, не третье состояние.
- Рендер (`surface`, `render`, `cam`, `draw`, `vgui`) — только клиент.
  Спавн, урон, `util.AddNetworkString` — только сервер.
- Сервер решает, что попадёт клиенту: `AddCSLuaFile` + `include`.
- Никогда не доверяй данным от клиента: всё, что пришло через `net`,
  concommand или клиентский конвар — валидируется на сервере.

## Ядро правил производительности

**Частота вызова — главный множитель стоимости.** Прежде чем писать код,
определи, как часто он выполняется:

| Горячий путь | Частота |
|---|---|
| `HUDShouldDraw` | сотни раз/сек (5+ за кадр) — только lookup-таблица |
| `HUDPaint`, `Think` (клиент), `CalcView`, `CreateMove` | каждый кадр |
| `Pre/PostDraw*Renderables` | несколько раз за кадр (depth + skybox) — фильтруй `bDrawingDepth`/`bDrawingSkybox` первой строкой |
| `PrePlayerDraw`/`PostPlayerDraw` | × число видимых игроков за кадр |
| `Tick`, `ENT:Think` (сервер) | каждый тик (66/сек) |

В горячем пути:

1. **Ноль аллокаций за кадр**: `Color()`, `Vector()`, `{}`, конкатенация
   строк, `string.format`, замыкания — всё это мусор для GC → микрофризы.
   Создавай один раз при загрузке файла, переиспользуй (`col.a = x`,
   `vec:SetUnpacked(...)`).
2. **Кэшируй за пределами выполнения**: локализуй глобальные функции в
   upvalue (`local floor = math.floor`), кэшируй `LocalPlayer()` лениво,
   `ScrW()/ScrH()` — с пересчётом по `OnScreenSizeChanged`,
   `surface.GetTextSize` — пока текст не изменился, `Material()` и
   `surface.CreateFont` — никогда внутри рендер-хуков.
3. **Собирай данные таймером, рисуй хуком**: `timer.Create` раз в 0.1с
   обновляет кэш, `HUDPaint` только рисует готовое — даже проверка
   `CurTime()` уходит из кадра.
4. **hook.Add → hook.Remove**: хук существует только пока нужен. Для хуков,
   привязанных к энтити/панели, передавай сам объект вторым аргументом —
   хук автоудалится при невалидности (объект придёт первым аргументом в
   callback). Уникальные имена хуков в цикле = утечка.
5. **Таймеры**: один глобальный таймер на всех игроков вместо N; удаляй
   по-игроковые таймеры в `PlayerDisconnected`; 50+ активных таймеров —
   повод для рефакторинга.
6. **Сеть**: `net.WriteUInt` с минимальными битами, никогда `WriteTable`;
   отправляй при изменении с тротлингом, не каждый тик; `net.Send(близким)`
   вместо `Broadcast`; NetworkVar (DTVar) > NW2 > NW.

## Ядро правил безопасности

Каждый `net.Receive` — вход для атакующего. Минимум: `IsValid(ply)`,
типы/диапазоны значений, права, дистанция, rate limit, лимит длины строк
и размера таблиц. Полный чеклист и готовый rate limiter — в разделах 4–5
руководства. SQL — только через `sql.SQLStr`; пути файлов — санитизация;
`RunString`/`CompileString`/`http.Fetch` в чужих аддонах — маркеры бэкдора.

## Разделы руководства (references/optimization_and_security.md)

| Тема | Раздел |
|---|---|
| 3D2D, дистанция, LOD, depth/skybox-проходы | 1.1 |
| HUD: кэширование, аллокации, таблица частот хуков, стоимость HUDPaint | 1.2 |
| Render targets для сложного UI | 1.3 |
| Коллизии, звуки, партиклы, материалы | 1.4–1.7 |
| Кэширование функций/переменных (upvalue, LocalPlayer, переиспользование объектов) | 1.8 |
| Жизненный цикл хуков, автоочистка по entity, аудит hook.GetTable | 1.9 |
| Серверные Think/таймеры, батчинг, мониторинг таймеров | 2.1–2.2 |
| Таблицы, NW/NW2/DTVar, ents.Find*, лимиты, tickrate | 2.3–2.7 |
| Оптимизация net-сообщений (размер, частота, RecipientFilter) | 3 |
| Безопасность net: валидация, rate limiting, чеклист | 4 |
| Бэкдоры, эксплойты, SQL/file injection | 5 |
| DDoS, connection flood, firewall | 6 |
| RCON, аддоны, privilege escalation | 7 |
| Античит (спидхак, телепорт, проп-спам, крашеры) | 8 |
| Структура аддона, профайлер хуков, чеклисты перед запуском | 9 |
