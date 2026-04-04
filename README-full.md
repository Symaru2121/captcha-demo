# Кастомные CAPTCHA для Sandbox v3

Набор из пяти кастомных капч с поддержкой **комбинатора**, **поведенческого анализа** и **Proof-of-Work (PoW)** модуля. Реализовано для интеграции с платформой Sandbox v3 через SDK (`Captcher` интерфейс).

---

## Содержание

1. [Общая архитектура](#общая-архитектура)
2. [Капча 1: «Найди лишнее»](#капча-1-найди-лишнее-odd-one-out)
3. [Капча 2: «Что невозможно»](#капча-2-что-невозможно-impossible-reality)
4. [Капча 3: «Нерусские буквы»](#капча-3-нерусские-буквы-alphabet-filter)
5. [Капча 4: «Поверни правильно»](#капча-4-поверни-правильно-rotation-match)
6. [Капча 5: «Собери кости»](#капча-5-собери-кости-dice-sum)
7. [Комбинатор капч](#комбинатор-капч)
8. [Поведенческий анализ](#поведенческий-анализ)
9. [Proof-of-Work](#proof-of-work)
10. [Демо-режим](#демо-режим)
11. [Интеграция — пошаговое руководство](#интеграция--пошаговое-руководство)
12. [Рекомендации по настройке и расширению](#рекомендации-по-настройке-и-расширению)
13. [Общие меры защиты](#общие-меры-защиты)
14. [Observability](#observability)
15. [Файловая структура](#файловая-структура)
15. [Требования к окружению](#требования-к-окружению)
16. [Объём кода](#объём-кода)
17. [v6 — Changelog](#v6--changelog)

---

## Общая архитектура

Каждая капча состоит из:

- **Go-бэкенд** — реализует интерфейс `server.Captcher` из SDK
- **TypeScript-фронтенд** — Canvas-based рендеринг, собирается Vite в single-file HTML
- **Бинарный протокол** — компактная передача данных между фронтом и бэком через `Uint8Array`

### Схема взаимодействия

```
Песочница (Sandbox)                          Капча (Go + HTML)
     │                                            │
     │──── gRPC: NewChallenge(complexity) ──────►│ генерирует задание + PoW challenge
     │◄─── ChallengeResponse(id, html) ──────────│ возвращает HTML (с инлайн JS)
     │                                            │
     │   [HTML загружается в iframe]              │
     │                                            │
     │ Frontend ──postMessage──► Sandbox ──gRPC──► Backend
     │           (Uint8Array)             (HandleFrontendEvent)
     │                                            │
     │ Backend ──gRPC──► Sandbox ──postMessage──► Frontend
     │        (SendClientData)    (captcha:serverData)
     │                                            │
     │                                            │  ┌───────────────────────────┐
     │                                            │  │ Lifecycle для v3:         │
     │                                            │  │ 1. Frontend: MsgClientReady│
     │                                            │  │ 2. Backend: payload + PoW │
     │                                            │  │ 3. Frontend: solve PoW    │
     │                                            │  │ 4. Frontend: показать UI  │
     │                                            │  │ 5. Пользователь: ответ    │
     │                                            │  │ 6. Frontend: submit       │
     │                                            │  │    (PoW solution + ответ  │
     │                                            │  │     + behavior trailer)   │
     │                                            │  │ 7. Backend: verify PoW    │
     │                                            │  │ 8. Backend: verify ответ  │
     │                                            │  │ 9. Backend: behavior score│
     │                                            │  │10. SendChallengeResult    │
     │                                            │  └───────────────────────────┘
     │                                            │
     │ Backend ──SendChallengeResult(confidence)──► Sandbox
     │                                            │
```

### Комбинатор (опционально)

```
Sandbox ──► captcha-combinator ──► random pick ──┬── captcha-odd-one-out
                                                  ├── captcha-impossible
                                                  ├── captcha-alphabet
                                                  ├── captcha-rotation
                                                  └── captcha-dice
```

Комбинатор выглядит снаружи как один `Captcher`, но при каждом `NewChallenge` случайно делегирует вызов одному из пяти модулей.

---

## Капча 1: «Найди лишнее» (Odd One Out)

**Папка:** `captcha-odd-one-out/`

### Механика

Пользователь видит сетку 2×3 из 6 эмодзи/символов. 5 из них принадлежат одной категории (например, фрукты), 1 — из другой (например, инструменты). Нужно кликнуть на лишний.

### Основные характеристики

- **24 категории**, 10 семейств, 288 символов
- Рендеринг через Canvas (анти-бот защита)
- Визуальная рандомизация: размер, поворот, смещение, цвета фона
- **complexity 1–100:** от очевидных различий (фрукты vs инструменты) до тонких (похожие подкатегории)

### Бинарный протокол

| Направление | Сообщение | Размер | Описание |
|---|---|---|---|
| Фронт → Бэк | `CLIENT_READY` (0x10) | 2 байта | Готовность frontend |
| Бэк → Фронт | `CHALLENGE_DATA` (0x81) | 24+ байта | Категории, предметы, seed + PoW блок |
| Фронт → Бэк | `CLICK` (0x20) | 8–16 байт | Индекс ячейки + тайминг + PoW solution + behavior trailer |

### Оценка результата (confidence)

- Правильный ответ: базовые 80, корректировка по таймингу и метаданным (итого 0–95)
- Неправильный ответ: 0
- Финальный confidence корректируется поведенческим анализом

---

## Капча 2: «Что невозможно» (Impossible Reality)

**Папка:** `captcha-impossible/`

### Механика

Пользователь видит 4 карточки с утверждениями на русском языке. 3 описывают реальные ситуации, 1 — абсурдную. Нужно кликнуть на невозможную.

### Основные характеристики

- **96 наборов заданий** (32 лёгких, 32 средних, 20 сложных, 12 экспертных)
- Весь текст рендерится на Canvas (не в DOM — защита от OCR)
- Визуальная рандомизация шрифтов, цветов, шума
- **complexity 1–100:** от очевидного абсурда до тонких нелепостей

### Примеры заданий

- **Лёгкий:** «Кот спит на подушке» / «Рыба плавает в воде» / «Птица поёт на ветке» / **«Камень летает по небу»**
- **Сложный:** более тонкая абсурдность, требующая культурного контекста

### Бинарный протокол

| Направление | Сообщение | Размер | Описание |
|---|---|---|---|
| Фронт → Бэк | `READY` | 8 байт | Готовность + метаданные |
| Бэк → Фронт | `CHALLENGE_PAYLOAD` | 17+ байт заголовок + строки | Набор из 4 утверждений + PoW блок |
| Фронт → Бэк | `CLICK` | 12+ байт | Индекс карточки + тайминг + PoW solution + behavior trailer |

### Оценка результата (confidence)

- Правильный ответ: 55–80 в зависимости от времени ответа (оптимум 800–5000 мс)
- Неправильный ответ: 0

---

## Капча 3: «Нерусские буквы» (Alphabet Filter)

**Папка:** `captcha-alphabet/`

### Механика

Пользователь видит сетку из 12–16 символов. Большинство — буквы русского алфавита, 2–4 — иностранные (греческие, латинские). Нужно отметить все нерусские буквы и нажать «Подтвердить».

### Основные характеристики

- **3 пула иностранных символов** по уровню сложности
- Canvas-рендеринг с рандомизацией шрифтов, поворотов, размеров
- Мультивыбор: клик для выделения, кнопка «Подтвердить» для отправки

### Маппинг сложности

| complexity | Сетка | Иностранных | Пул |
|---|---|---|---|
| 1–20 | 3×4 (12) | 2 | Лёгкий (греческие: Ω, Ψ, Φ, Σ…) |
| 21–40 | 3×4 (12) | 3 | Лёгкий (греческие) |
| 41–60 | 3×4 (12) | 3 | Средний (латинские: G, F, J, L…) |
| 61–75 | 4×4 (16) | 3 | Средний (латинские) |
| 76–90 | 4×4 (16) | 4 | Сложный (двойники: A/А, B/В, C/С…) |
| 91–100 | 4×4 (16) | 4 | Сложный (двойники) |

### Бинарный протокол

| Направление | Сообщение | Размер | Описание |
|---|---|---|---|
| Фронт → Бэк | `INIT_REQUEST` | 6 байт | Запрос инициализации |
| Бэк → Фронт | `CHALLENGE_DATA` | 22+ байт | Заголовок + данные ячеек + PoW блок |
| Фронт → Бэк | `SUBMIT_SELECTION` | 12+ байт | Битмаска выбранных + nonce + тайминг + PoW solution + behavior trailer |

### Оценка результата (confidence)

- Все правильно: 80 (с возможным штрафом за тайминг, минимум 55)
- Частично: `80 × (0.65 × recall + 0.35 × precision)` минус штрафы за ошибки
- Пустой выбор: 0

---

## Капча 4: «Поверни правильно» (Rotation Match)

**Папка:** `captcha-rotation/`

### Механика

Пользователь видит процедурный силуэт (SVG-path), отрендеренный на Canvas и повёрнутый на неправильный угол. Необходимо кликать стрелки (↺ / ↻), чтобы повернуть силуэт в правильное вертикальное положение относительно стрелки-указателя на север. Задание состоит из 1–3 раундов в зависимости от сложности.

### Основные характеристики

- **80 процедурных силуэтов** (SVG-path) в **8 категориях** (по 10 в каждой):
  1. Животные (кот, собака, кролик, лиса, птица, рыба, черепаха, слон, жираф, улитка)
  2. Инструменты (молоток, гаечный ключ, отвёртка, плоскогубцы, кисть, лопата, топор, пила, ключ, ножницы)
  3. Транспорт (машина, автобус, велосипед, самолёт, парусник, ракета, трактор, вертолёт, поезд, самокат)
  4. Быт (чайник, стул, лампа, кружка, бутылка, диван, часы, лейка, утюг, замочная скважина)
  5. Природа (дерево, ёлка, гора, лист, цветок, облако, снежинка, кактус, гриб, жёлудь)
  6. Еда (мороженое, кекс, пицца, банан, морковь, груша, рыбный скелет, хлеб, яблоко, хот-дог)
  7. Спорт (кубок, ракетка, бита, гантель, медаль, боксёрская перчатка, кегля, скейтборд, свисток, ворота)
  8. Музыка (гитара, скрипка, труба, барабан, микрофон, пианино, нота, наушники, арфа, колонка)
- Каждый силуэт: ID, категория, slug, название (RU), сложность (1–3), viewBox 100×100, 1–3 SVG path
- **Многораундовая механика** (1–3 раунда)
- Canvas-рендеринг с визуальным шумом (фоновые точки/линии, ложные направляющие метки, стрелка севера)
- Per-challenge визуальная рандомизация: цвет заливки, стиль обводки, тени — через seeded PRNG (`renderSeed + roundRenderSeed`)
- Дискретный шаг поворота: 90° (лёгкий) → 45° → 60° → 30° (сложный)
- Плавная анимация поворота (`easeOutCubic`, 120–320 мс на шаг)

### Маппинг сложности

| complexity | Раунды | Шаг угла | Сложность силуэта | Вариантов поворота |
|---|---:|---:|---|---:|
| 1–10 | 1 | 90° | лёгкий | 4 |
| 11–20 | 1 | 90° | лёгкий | 4 |
| 21–30 | 1 | 60° | лёгкий | 6 |
| 31–40 | 1 | 60° | лёгкий/средний | 6 |
| 41–50 | 2 | 60° | средний | 6 |
| 51–60 | 2 | 45° | средний | 8 |
| 61–70 | 2 | 45° | средний/сложный | 8 |
| 71–80 | 3 | 45° | средний/сложный | 8 |
| 81–90 | 3 | 45° | сложный | 8 |
| 91–100 | 3 | 30° | сложный | 12 |

### Бинарный протокол

| Направление | Сообщение | Размер | Описание |
|---|---|---|---|
| Фронт → Бэк | `CLIENT_READY` (0x10) | 4 байта | `[0x10, 0x01, clientFlags, 0x00]` |
| Бэк → Фронт | `CHALLENGE_DATA` (0x81) | 16 + N×16 + 30 байт | Заголовок + раунд-блоки + PoW блок |
| Фронт → Бэк | `SUBMIT` (0x20) | 24 байта + behavior trailer | Финальные углы, счётчики кликов, PoW solution, тайминг |

**CHALLENGE_DATA** (бэк → фронт):
- Заголовок (16 байт): `[0x81, 0x01, rounds, angleStep, rotOpts, noiseLevel, uiFlags, complexity, renderSeed(4), payloadFlags(2), roundBlockSize(2)]`
- Раунд-блок (16 байт × N): `[silhouetteId(2), categoryId, difficulty, initialAngle(2), roundSeed(4), animDurationMs(2), reserved(4)]`
- PoW-блок: 30 байт (`pow.EncodeChallengeBinary`)

**SUBMIT** (фронт → бэк):
- `[0x20, 0x01, rounds, flags, elapsedDiv10(2), powSolution(8), pointerKind, 0x00, angle0(2), angle1(2), angle2(2), clickCounts01, clickCounts2]`

### Оценка результата (confidence)

- Все раунды неправильно: 0
- Пропорциональная база: `80 × (correctRounds / totalRounds)`
- Штраф за частичные ошибки: −6 за каждый неправильный раунд
- Слишком быстро (<500 мс/раунд): до −18
- Слишком медленно (>30 с итого): до −12
- Ноль кликов в раунде: −25 за раунд
- Избыточные клики (≥12): −3 за раунд
- Финальное значение clamped [0, 95], затем применяется поведенческий множитель

### Меры защиты

- Canvas-only рендеринг силуэтов (нет SVG DOM узлов, нет текстовых меток)
- Правильный угол никогда не передаётся на фронтенд
- Per-challenge seeded визуальная рандомизация (цвет, обводка, тень, шум)
- Дискретный шаг поворота (нет произвольных значений углов)
- Задержка анимации предотвращает мгновенное решение (120–320 мс на шаг)
- Гарант раунда: минимум один поворот требуется, когда начальный угол неправильный
- PoW-gate перед показом интерактивного UI
- Анализ поведенческого трейлера (траектория, скорость, тайминг, точность кликов, присутствие)
- Анализ паттерна кликов на бэкенде

---

## Капча 5: «Собери кости» (Dice Sum)

**Папка:** `captcha-dice/`

### Механика

Пользователь видит **целевое число** на левой панели и **сетку 2×3 из 6 карточек** справа. На каждой карточке — от 1 до 3 процедурно отрендеренных 3D-кубиков со стандартными казино-пипами (грани 1–6). Нужно найти и кликнуть карточку, сумма кубиков которой равна целевому числу.

Многораундовая механика: в зависимости от сложности играется 1–3 раунда последовательно. После каждого клика — зелёный/красный фидбек, следующий раунд появляется автоматически.

### Основные характеристики

- **Процедурный 3D-рендеринг** кубиков на Canvas
- **8 цветовых схем** (ivory, ruby, cobalt, emerald, amber, violet, slate, coral)
- **4 стиля точек** (circle, rounded square, diamond, ring-dot hybrid)
- 3D псевдо-изометрический рендеринг: передняя грань + верхняя грань (трапеция) + боковая грань (трапеция)
- Стандартные казино-позиции пипов на сетке 3×3
- Per-die seeded вариации: поворот (0–15°), масштаб (0.92–1.08), jitter позиции, цветовая схема, направление тени
- Поверхностные дефекты: крапинки, царапины, градиентный шум, износ граней (плотность масштабируется по complexity)
- **Интеллектуальная генерация дистракторов** (суммы ±1 от целевого на сложных уровнях)
- **Многораундовая механика** (1–3 раунда)

### Маппинг сложности

| complexity | Раунды | Кубиков/карта | Диапазон цели | Разброс дистракторов | Шум | Перекрытие |
|---|---:|---|---|---|---|---|
| 1–10 | 1 | 1 | 3–5 | ≥2 | 0 | Нет |
| 11–20 | 1 | 1 | 3–6 | ≥2 | 0 | Нет |
| 21–30 | 1 | 1–2 | 3–7 | ≥2 | 1 | Нет |
| 31–40 | 2 | 1–2 | 5–8 | ≥1 | 1 | Нет |
| 41–50 | 2 | 1–2 | 5–9 | ≥1 | 1 | Нет |
| 51–60 | 2 | 1–2 | 6–10 | ≥1 | 2 | Нет |
| 61–70 | 3 | 2–3 | 7–11 | ±1 ok | 2 | Да |
| 71–80 | 3 | 2–3 | 8–14 | ±1 ok | 3 | Да |
| 81–90 | 3 | 2–3 | 8–16 | ±1 кластер | 3 | Да |
| 91–100 | 3 | 2–3 | 9–18 | ±1 кластер | 4 | Да |

### Бинарный протокол

| Направление | Сообщение | Размер | Описание |
|---|---|---|---|
| Фронт → Бэк | `CLIENT_READY` (0x10) | 3 байта | Готовность frontend |
| Бэк → Фронт | `CHALLENGE_DATA` (0x81) | 42 + N×63 байт | Фиксированный заголовок + раунд-блоки + render hint sidecar |
| Фронт → Бэк | `CLICK` (0x20) | 16 байт + behavior trailer | Индекс раунда, индекс карточки, тайминг, PoW solution |
| Бэк → Фронт | `ROUND_FEEDBACK` (0x82) | 6 байт | Результат раунда (correct/incorrect, final flag) |

**CHALLENGE_DATA** (бэк → фронт):
- Фиксированный заголовок (42 байта): `msgType, version, roundCount, gridCols=3, gridRows=2, complexity, challengeFlags, reserved, challengeSeed(u32 LE), powBlock(30 bytes)`
- Раунд-блоки (39 байт × N): `targetNumber, roundFlags, roundIndex, reserved, roundSeed(u32 LE), 6×5-byte card records + 1 pad byte`
- Render hint sidecar (24 байта × N): `6×4-byte hint records (rotationQ, colorScheme, dotStyle|shadowDir, sizeVariant)`

**CLICK** (фронт → бэк):
- `msgType(1) | version(1) | roundIndex(1) | cardIndex(1) | elapsedMsDiv10(u16 LE) | clickFlags(1) | pointerKind(1) | powSolution(u64 LE)`
- Behavior trailer (начинается с magic `0xBE`) после байта 15

**ROUND_FEEDBACK** (бэк → фронт):
- `msgType=0x82 | version | roundIndex | resultFlags(bit0=correct, bit1=final) | serverDelayMsDiv10(u16 LE)`

### Оценка результата (confidence)

- **Все раунды правильно:** базовые 80, минус штрафы за тайминг по раундам (clamped 55–80)
- **Частично правильно:** `80 × (correct / total) × 0.7`, clamped 1–56
- **Все неправильно:** 0
- **Штрафы за тайминг:** <350 мс: 12, 350–699 мс: 8, 700–999 мс: 4, 1000–8000 мс: 0, 8001–15000 мс: 2, 15001–30000 мс: 4, >30000 мс: 6
- **Поведенческий множитель (V3):** `final = base × (0.5 + 0.5 × behaviorScore / 100)`

### Меры защиты

- Canvas-only рендеринг: никакие DOM-элементы не раскрывают значения или суммы кубиков
- Правильный индекс карточки никогда не передаётся на фронтенд — проверка на сервере
- PoW привязан ко всему challenge (одно решение для всех раундов)
- Визуальный шум детерминирован из `challengeSeed + roundSeed + cardIndex + dieIndex`
- `ROUND_FEEDBACK` (0x82) — лёгкий ACK с per-round фидбеком до финального результата

---

## Комбинатор капч

### Что это

`captcha-combinator` — тонкий wrapper, который снаружи выглядит как единый `Captcher`, а внутри при каждом `NewChallenge(complexity)` случайно выбирает один из пяти типов капч:

| Kind | Модуль |
|---|---|
| 1 | captcha-odd-one-out |
| 2 | captcha-impossible |
| 3 | captcha-alphabet |
| 4 | captcha-rotation |
| 5 | captcha-dice |

### Зачем

- Непредсказуемость для ботов: нельзя заранее знать, какая капча будет показана
- Единая точка входа для платформы: sandbox видит один `Captcher`, не пять
- Прозрачная маршрутизация: все вызовы SDK (`HandleFrontendEvent`, `HandleConnectionClosed` и т.д.) автоматически делегируются нужному модулю по `challengeId`

### Как работает

1. `NewChallenge(complexity)` → случайный выбор `kind` (равномерный 1/5)
2. Делегирование в соответствующий sub-captcha
3. Сохранение маппинга `challengeId → kind` для последующих вызовов
4. `HandleFrontendEvent` / `HandleConnectionClosed` / `HandleCloseChallenge` — маршрутизация по маппингу
5. `Stream()` — fan-out во все пять sub-captcha (нужно для async очередей)
6. Очистка маппинга при закрытии challenge

### Как подключить

```go
// main.go для комбинатора
package main

import (
    "captcha-combinator/internal/captcha"
    "sdk/server"
)

func main() {
    server.SetCaptcherBuilder(func(ct server.ChallengeId, id server.InstanceId) (server.Captcher, error) {
        return captcha.NewCaptcha(), nil
    })
    server.Run()
}
```

---

## Поведенческий анализ

### Что собирает

Frontend через `BehaviorCollector` накапливает поведенческий след пользователя:

| Данные | Максимум | Размер | Описание |
|---|---|---|---|
| Trajectory samples | 12 | 72 байт | Координаты + время с шагом 50 мс |
| Click events | 3 | 18 байт | Координаты + время кликов |
| Focus/blur events | 4 | 12 байт | Фокус/потеря фокуса canvas |
| Motion summary | — | 6 байт | meanVelocity, CV, acceleration |
| Header + CRC | — | 13 байт | Версия, флаги, контрольная сумма |

**Полный размер extension:** 73–115 байт (типичный ~73 байт).

Данные сериализуются в бинарный trailer с магическим байтом `0xBE` и дописываются в конец существующего click/submit пакета. Старый frontend без behavioral extension совместим: backend обнаружит отсутствие trailer и не штрафует.

### Как оценивает

Backend вычисляет `behaviorScore` (0–100) как взвешенное среднее пяти подсчётов:

| Подсчёт | Вес | Что оценивает |
|---|---|---|
| **LinearityScore** | 0.30 | Слишком прямолинейная траектория → бот |
| **VelocityScore** | 0.20 | Слишком равномерная скорость → бот |
| **TimingScore** | 0.20 | Слишком регулярные интервалы → бот |
| **ClickPrecisionScore** | 0.15 | Попадание в идеальный центр → бот |
| **PresenceScore** | 0.15 | Наличие движения перед кликом |

### Формулы скоринга

**Linearity** — подозрительность прямой траектории:

```
S_R = clamp((R - 0.94) / 0.06, 0, 1)     где R = displacement / pathLength
S_A = clamp((0.18 - A) / 0.18, 0, 1)     где A = сумма углов поворотов (рад)
LinearityScore = round(100 × (1 - (0.6 × S_R + 0.4 × S_A)))
```

**Velocity** — подозрительность равномерной скорости:

```
S_v = clamp((0.08 - CV_v) / 0.08, 0, 1)  где CV_v = stddev(v) / mean(v)
VelocityScore = round(100 × (1 - S_v))
```

**Timing** — подозрительность регулярных интервалов:

```
S_t = clamp((0.10 - CV_t) / 0.10, 0, 1)  где CV_t = stddev(dt) / mean(dt)
TimingScore = round(100 × (1 - S_t))
```

**Итоговая формула confidence:**

```
finalConfidence = baseConfidence × (0.5 + 0.5 × behaviorScore / 100)
```

Примеры:

| baseConfidence | behaviorScore | finalConfidence |
|---|---|---|
| 80 | 100 (человек) | 80 |
| 80 | 50 (подозрительный) | 60 |
| 80 | 0 (явный бот) | 40 |

---

## Proof-of-Work

### Что это

PoW-модуль (`pkg/pow/`) — переиспользуемый серверно-клиентский компонент, который удорожает массовое получение капч. Перед показом основного задания браузер должен вычислить nonce, чтобы SHA-256 хэш имел требуемое количество ведущих нулевых бит.

### Зачем

- Удорожить массовое решение капч ботами
- Асимметричная защита: проверка на сервере — 1 SHA-256, поиск на клиенте — тысячи итераций
- Не заменяет капчу, а является pre-gate перед показом challenge

### Как работает

1. Сервер генерирует `Challenge{Prefix, Difficulty, Timestamp, Nonce}`
2. Challenge передаётся клиенту в составе бинарного payload (30 байт)
3. Клиент показывает «Подготовка... X%» и перебирает nonce от 0
4. Для каждого кандидата: `SHA-256(prefix[16] || timestamp_LE[4] || nonce[8] || solution_LE[8])`
5. Если хэш начинается с ≥ `difficulty` нулевых бит → решение найдено
6. Клиент сохраняет решение и показывает UI капчи
7. При submit клиент отправляет `powSolution + captchaAnswer` одним пакетом
8. Сервер сначала проверяет PoW (дёшево), затем проверяет ответ капчи

### Уровни сложности

| complexity (1–100) | difficulty (бит) | Ожидаемые итерации | Целевая задержка |
|---|---|---|---|
| 1–30 | 14 | ~16 384 | ~0.2 с |
| 31–60 | 16 | ~65 536 | ~0.5 с |
| 61–80 | 18 | ~262 144 | ~1.0 с |
| 81–100 | 20 | ~1 048 576 | ~2.0 с |

### Публичный Go API

```go
package pow

// Challenge — серверный PoW challenge.
type Challenge struct {
    Prefix     string // 16-byte random hex prefix (32 символа)
    Difficulty int    // число ведущих нулевых бит
    Timestamp  int64  // unix seconds
    Nonce      string // 8-byte server nonce hex (16 символов)
}

func GenerateChallenge(difficulty int) *Challenge
func VerifyProof(c *Challenge, solution uint64) bool
func VerifyProofFull(c *Challenge, solution uint64, ttl time.Duration) VerifyResult
func DifficultyForComplexity(complexity int32) int
func IsExpired(c *Challenge, ttlSeconds int64) bool
func CountLeadingZeroBits(hash [32]byte) uint8
func EncodeChallengeBinary(ch *Challenge) ([30]byte, error)
func DecodeChallengeBinary(data []byte) (*Challenge, bool)
```

### Frontend JS API

```js
// pow_solver.js
async function solvePow(prefixBytes, difficulty, issuedAtUnix, nonceBytes, onProgress)
function decodePowChallenge(data, powOffset)

// pow_widget.js
function mountPowWidget({ container, challenge, onDone, onError })
```

### Безопасность PoW

- **TTL:** challenge действителен 60 секунд, затем сервер отвергает proof
- **Одноразовость:** state удаляется после первого submit
- **Replay-защита:** stateful challenge lifecycle + timestamp validation
- **Clock skew:** отклонение > 5 с в будущее → reject
- **Prefix через crypto/rand:** исключает предсказуемость
- **Difficulty trust boundary:** сервер использует только сохранённый difficulty, не клиентское значение

---

## Демо-режим

**Папка:** `demo/`

Автономные HTML-файлы для просмотра и тестирования капч без Go-бэкенда и песочницы. Вся логика генерации и валидации реализована на клиенте.

### Запуск

Откройте `demo/index.html` в браузере — он загрузит все 5 капч.

Или откройте любую капчу отдельно:

| Файл | Капча |
|---|---|
| `demo/captcha1_demo.html` | Найди лишнее |
| `demo/captcha2_demo.html` | Что невозможно? |
| `demo/captcha3_demo.html` | Нерусские буквы |
| `demo/captcha4_demo.html` | Поверни правильно |
| `demo/captcha5_demo.html` | Собери кости |

Работает как `file://`, сервер не нужен.

### Что включено в демо

- Полная база данных категорий/заданий/символов/силуэтов из продакшн-кода
- Canvas-рендеринг, идентичный продакшн-версии
- Интерактивность: клики, выбор, подтверждение, поворот
- Визуальный фидбек: правильно (зелёный), неправильно (красный), пропущено (оранжевый)
- Подсчёт confidence
- Слайдер сложности (1–100) для тестирования разных уровней
- Кнопка «Новое задание» для перегенерации
- Поддержка `postMessage`-интеграции с лаунчером (все 5 капч)
- Двусторонняя коммуникация: `setComplexity`, `reset` → iframe; `mouseMove`, `click`, `status`, `result` → parent

> **Примечание:** В демо-режиме ответ проверяется на клиенте. В продакшн-версии ответ ВСЕГДА проверяется на сервере — клиент никогда не знает правильный ответ.

---

## Интеграция — пошаговое руководство

### Как подключить одну капчу

1. Соберите фронтенд:

```bash
cd captcha-NAME/frontend
yarn install
yarn build    # single-file HTML → ../internal/captcha/html/index.html
```

2. Запустите капчу:

```bash
cd captcha-NAME
go run main.go
```

3. Капча подключится к песочнице по gRPC на порт 38001 и станет доступна в интерфейсе.

### Как подключить комбинатор

1. Вместо запуска отдельной капчи, запустите `captcha-combinator`:

```bash
cd captcha-combinator
go run main.go
```

2. Комбинатор автоматически инициализирует все пять капч и при каждом запросе случайно выбирает одну.

3. В `main.go` комбинатора:

```go
server.SetCaptcherBuilder(func(ct server.ChallengeId, id server.InstanceId) (server.Captcher, error) {
    return captcha.NewCaptcha(), nil // возвращает комбинатор
})
```

### Как включить PoW

В `NewChallenge` вашей капчи:

```go
import "pkg/pow"

func (c *Captcha) NewChallenge(ctx context.Context, complexity int32) (*server.Challenge, error) {
    // 1. Определить difficulty из complexity.
    powDifficulty := pow.DifficultyForComplexity(complexity)

    // 2. Сгенерировать PoW challenge.
    powChallenge := pow.GenerateChallenge(powDifficulty)

    // 3. Сохранить в state.
    state := generateChallenge(complexity)
    state.Pow = *powChallenge

    // 4. Встроить PoW в payload (encodeChallengePayload).
    // ...
}
```

В `handleClick` / `handleSubmit`:

```go
// 5. Извлечь PoW solution из submit.
powSolution := binary.LittleEndian.Uint64(data[2:10])

// 6. Проверить PoW ДО проверки ответа капчи.
if !pow.VerifyProof(&state.Pow, powSolution) {
    delete(states, state.ChallengeID)
    return stream.SendChallengeResult(id, &event.ServerEventChallengeResult{ConfidencePercent: 0})
}

// 7. Только после успешной проверки PoW — обычная валидация.
```

На фронтенде — добавить `pow_solver.js` и `pow_widget.js`:

```js
// После получения payload от сервера:
const powChallenge = decodePowChallenge(payloadBytes, powOffset);

mountPowWidget({
  container: document.getElementById('app'),
  challenge: powChallenge,
  onDone: function(solution) {
    window.__powSolution = solution;
    document.getElementById('captchaRoot').hidden = false;
  }
});

// При submit — включить solution в пакет:
writeUint64LE(view, 2, window.__powSolution);
```

### Как включить поведенческий анализ

На фронтенде — инициализировать `BehaviorCollector`:

```ts
const behavior = new BehaviorCollector(canvasElement);
behavior.start();

// При submit — дописать trailer:
const packet = behavior.appendToMessage(basePacket);
window.top.postMessage({ type: 'captcha:sendData', data: packet }, '*');
```

На бэкенде — распарсить extension и применить score:

```go
import "pkg/behavior"

// 1. Парсить legacy payload как обычно.
click, ok := decodeClickPayload(data[:legacyLen])

// 2. Парсить behavior extension.
behaviorPayload, _ := behavior.ParseBehaviorExtension(data, legacyLen)

// 3. Вычислить behavior score.
analyzer := behavior.NewBehaviorAnalyzer()
result := analyzer.Score(behaviorPayload, targetCenterX, targetCenterY, targetRadius)

// 4. Применить формулу.
baseConfidence := computeConfidence(state, click, isCorrect)
finalConfidence := behavior.ApplyBehaviorConfidence(baseConfidence, result.Score)
```

### Пример минимального кода интеграции

Полный пример интеграции PoW + поведенческого анализа доступен в файле:

```
pkg/pow/integration_example.go
```

---

## Рекомендации по настройке и расширению

### Как добавить новую капчу

1. Скопировать структуру любой из существующих капч
2. Реализовать интерфейс `Captcher` из SDK:

```go
type Captcher interface {
    NewChallenge(context.Context, int32) (*Challenge, error)
    HandleFrontendEvent(context.Context, Stream, string, []byte) error
    HandleConnectionClosed(context.Context, string, []byte) error
    HandleBalancerEvent(context.Context, Stream, string, []byte) error
    HandleCloseChallenge(context.Context, Stream, string, []byte) error
    OnStreamStarted(context.Context, Stream) error
    OnStreamClosed(context.Context, Stream)
    Stream(context.Context, Stream)
}
```

3. Создать фронтенд (Canvas-рендеринг + бинарный протокол)
4. Встроить `BehaviorCollector` и `pow_solver.js` / `pow_widget.js`
5. Собрать через Vite в single-file HTML
6. Запустить — песочница подхватит автоматически
7. При желании зарегистрировать в комбинаторе (добавить kind + delegate)

### Как настроить сложность

Параметр `complexity` (1–100) приходит от платформы в `NewChallenge`. Каждая капча интерпретирует его по-своему:

- **Найди лишнее:** сходство категорий (1 = фрукты vs инструменты, 100 = фрукты vs овощи)
- **Что невозможно:** tier набора заданий (1 = очевидный абсурд, 100 = тонкая нелепость)
- **Нерусские буквы:** размер сетки + тип символов (1 = греческие, 100 = латинские двойники)
- **Поверни правильно:** раунды + шаг угла + сложность силуэтов (1 = 1 раунд, 90°, лёгкие; 100 = 3 раунда, 30°, сложные)
- **Собери кости:** раунды + количество кубиков + разброс дистракторов (1 = 1 раунд, 1 кубик; 100 = 3 раунда, 2–3 кубика, ±1 кластер)

### Как настроить PoW difficulty

PoW difficulty привязан к `complexity` через `DifficultyForComplexity()`. Маппинг:

```go
complexity 1-30   → 14 бит (~0.2 с)
complexity 31-60  → 16 бит (~0.5 с)
complexity 61-80  → 18 бит (~1.0 с)
complexity 81-100 → 20 бит (~2.0 с)
```

Не рекомендуется устанавливать difficulty выше 20 для интерактивного пользовательского потока. На слабых мобильных устройствах difficulty 20 может ощущаться как 3–5 секунд.

Для изменения маппинга отредактируйте `DifficultyForComplexity()` в `pkg/pow/pow.go`.

### Rate limiting (рекомендация для платформы)

PoW и капча не заменяют rate limiting. Рекомендуется на уровне платформы:

- Ограничить количество `NewChallenge` запросов на IP/сессию
- Ввести exponential backoff при множественных неудачных попытках
- Мониторить ratio success/failure по IP

### Fingerprinting (рекомендация для платформы)

На уровне платформы рекомендуется собирать:

- User-Agent, Accept-Language
- Canvas fingerprint (уже используется captcha UI)
- WebGL renderer
- Screen resolution / timezone
- Доступные шрифты

Это позволит кластеризовать подозрительные сессии и повышать `complexity` для аномальных клиентов.

### Мониторинг и алерты

Рекомендуемые метрики:

| Метрика | Порог для алерта |
|---|---|
| PoW reject rate | > 5% — возможна проблема с клиентом |
| Средний behaviorScore | < 50 — массовая бот-активность |
| Среднее время решения капчи | < 200 мс — подозрительно быстро |
| Challenge TTL timeout rate | > 10% — возможно, difficulty слишком высокий |
| Confidence = 0 rate | > 30% — аномальный трафик |

---

## Общие меры защиты

Все капчи используют следующие механизмы защиты:

1. **Canvas-only рендеринг** — символы и тексты рисуются только на Canvas, не размещаются в DOM
2. **Серверная валидация** — правильный ответ никогда не передаётся на фронт
3. **Рандомизация визуала** — шрифты, повороты, размеры, цвета, шум — разные для каждого задания (через seeded PRNG)
4. **Тайминг-анализ** — слишком быстрые и слишком медленные ответы снижают confidence
5. **Одноразовые задания** — после ответа (любого) задание удаляется из памяти
6. **Фоновая очистка** — горутина каждые 30 секунд удаляет просроченные задания (TTL 5 мин)
7. **Компактный бинарный протокол** — минимальный трафик, подходит для медленных соединений
8. **Proof-of-Work** — computational pre-gate, удорожает массовые запросы
9. **Поведенческий анализ** — trajectory, velocity, timing, click precision — multiplier на confidence
10. **Комбинатор** — рандомный выбор типа капчи (1/5), непредсказуем для ботов
11. **TTL challenge** — PoW challenge действителен 60 секунд, captcha state — 5 минут
12. **Replay-защита** — stateful challenge lifecycle, одноразовый submit
13. **Clock skew protection** — отклонение > 5 с в будущее → reject
14. **One-shot semantics** — после любого финального submit state удаляется

---

## Observability

All captcha backends emit structured JSON log lines via Go's `log/slog` package. These can be scraped by Prometheus (via `loki`), Datadog, or any structured-log pipeline.

### Key log events

| Event name | Level | When emitted | Key fields |
|---|---|---|---|
| `captcha.new_challenge` | INFO | A new challenge is created | `type`, `complexity`, `version` |
| `captcha.result` | INFO | A challenge is scored | `type`, `challenge_id`, `correct`, `confidence`, `elapsed_ms`, `behavior_score`, `pow_valid`, `complexity` |
| `pow.rejected` | WARN | PoW verification fails | `challenge_id`, `reason` ("invalid_proof", "expired", "clock_skew") |
| `combinator.dispatch` | INFO | Combinator selects a sub-captcha | `kind`, `challenge_id`, `complexity` |
| `*.expired challenge cleaned up` | INFO | TTL cleanup removes a stale challenge | `challenge_id` |

### Recommended dashboards

Filter on `captcha.result` to build:

- **Success rate** — `count(correct=true) / count(*)` grouped by `type`
- **Confidence distribution** — histogram of `confidence` per `type`
- **Solve latency** — p50/p95/p99 of `elapsed_ms`
- **Behavior score distribution** — histogram of `behavior_score`
- **PoW reject rate** — `count(pow.rejected) / count(captcha.result)`

### Rate limiter in the combinator

The combinator (`captcha-combinator`) includes a per-instance token-bucket rate limiter on `NewChallenge` requests. Default: 50 burst, 1 token per 100ms refill (≈10 rps sustained). When the limit is exceeded, the combinator returns an error and logs `combinator: rate limit exceeded for NewChallenge`.

For production, replace with a distributed rate limiter keyed by IP/session (e.g., Redis-backed sliding window).

---

## Файловая структура

```
captcha-delivery/
├── README.md                                    # Этот файл
│
├── pkg/                                         # Общие переиспользуемые пакеты
│   ├── pow/                                     # Proof-of-Work модуль
│   │   ├── pow.go                               # Публичный API: генерация, верификация, маппинг
│   │   ├── pow_test.go                          # Unit-тесты (13 тестов)
│   │   ├── pow_solver.js                        # Frontend PoW solver (SubtleCrypto)
│   │   ├── pow_widget.js                        # Встраиваемый виджет прогресса
│   │   ├── integration_example.go               # Пример интеграции с captcha
│   │   └── go.mod
│   └── behavior/                                # Поведенческий анализ
│       └── behavior.go                          # Парсер, анализатор, scoring
│
├── captcha-odd-one-out/                         # Капча «Найди лишнее»
│   ├── main.go                                  # Точка входа
│   ├── go.mod
│   ├── internal/captcha/
│   │   ├── captcha.go                           # Реализация Captcher
│   │   ├── captcha_v3.go                        # V3 с поведенческим анализом
│   │   ├── categories.go                        # База 24 категорий
│   │   ├── generator.go                         # Генерация + бинарный протокол
│   │   └── html/
│   │       └── index.html                       # Собранный фронтенд (single-file)
│   └── frontend/
│       ├── src/index.ts                         # Canvas-рендеринг (533 строки)
│       └── src/styles.scss
│
├── captcha-impossible/                          # Капча «Что невозможно»
│   ├── main.go
│   ├── go.mod
│   ├── internal/captcha/
│   │   ├── captcha.go                           # Реализация Captcher
│   │   ├── captcha_v3.go                        # V3 с поведенческим анализом
│   │   ├── sets.go                              # 96 наборов заданий
│   │   ├── generator.go                         # Генерация + протокол
│   │   └── html/
│   │       └── index.html                       # Собранный фронтенд (single-file)
│   └── frontend/
│       ├── src/index.ts                         # Canvas-рендеринг текста (622 строки)
│       └── src/styles.scss
│
├── captcha-alphabet/                            # Капча «Нерусские буквы»
│   ├── main.go
│   ├── go.mod
│   ├── internal/captcha/
│   │   ├── captcha.go                           # Реализация Captcher
│   │   ├── charsets.go                          # Пулы символов
│   │   ├── generator.go                         # Генерация + протокол
│   │   └── html/
│   │       └── index.html                       # Собранный фронтенд (single-file)
│   └── frontend/
│       ├── src/index.ts                         # Canvas-рендеринг (612 строк)
│       └── src/styles.scss
│
├── captcha-rotation/                            # Капча «Поверни правильно»
│   ├── main.go                                  # Точка входа
│   ├── go.mod
│   ├── SUMMARY.md
│   ├── internal/captcha/
│   │   ├── captcha.go                           # Реализация Captcher
│   │   ├── captcha_v3.go                        # V3 с поведенческим анализом
│   │   ├── silhouettes.go                       # База 80 силуэтов (8 категорий)
│   │   ├── generator.go                         # Генерация + бинарный протокол + скоринг
│   │   └── html/
│   │       └── index.html                       # Собранный фронтенд (single-file)
│   └── frontend/
│       ├── src/index.ts                         # Canvas-рендеринг + поворот (791 строка)
│       └── src/styles.scss
│
├── captcha-dice/                                # Капча «Собери кости»
│   ├── main.go                                  # Точка входа
│   ├── go.mod
│   ├── SUMMARY.md
│   ├── internal/captcha/
│   │   ├── captcha.go                           # Реализация Captcher
│   │   ├── captcha_v3.go                        # V3 с поведенческим анализом
│   │   ├── generator.go                         # Генерация + протокол + скоринг
│   │   └── html/
│   │       └── index.html                       # Собранный фронтенд (single-file)
│   └── frontend/
│       ├── src/index.ts                         # 3D-рендеринг кубиков (1190 строк)
│       └── src/styles.scss
│
├── captcha-combinator/                          # Комбинатор (5 капч)
│   ├── main.go
│   ├── go.mod
│   └── internal/combinator/
│       └── combinator.go                        # Роутер по kind 1–5
│
└── demo/                                        # Автономные демо без сервера
    ├── index.html                               # Лаунчер — все 5 капч + навигация
    ├── captcha1_demo.html                       # Найди лишнее (автономный)
    ├── captcha2_demo.html                       # Что невозможно (автономный)
    ├── captcha3_demo.html                       # Нерусские буквы (автономный)
    ├── captcha4_demo.html                       # Поверни правильно (автономный)
    └── captcha5_demo.html                       # Собери кости (автономный)
```

---

## Требования к окружению

| Компонент | Версия |
|---|---|
| Go | 1.21+ |
| Node.js | 18+ (для сборки фронтенда) |
| SDK | `sandbox-v3/pkg/sdk` |
| Браузер | Chrome 70+ / Firefox 63+ / Safari 12+ (Web Crypto API) |

### Сборка фронтенда (для каждой капчи)

```bash
cd captcha-NAME/frontend
yarn install
yarn build    # собирает single-file HTML в ../internal/captcha/html/index.html
```

### Запуск капчи

```bash
cd captcha-NAME
go run main.go
```

Капча подключится к песочнице по gRPC на порт 38001.

### Запуск песочницы

```bash
docker-compose up    # или ./bin/sandbox
```

---

## Объём кода

### По модулям

| Модуль | Go (строк) | TypeScript/JS (строк) | Всего |
|---|---:|---:|---:|
| Найди лишнее | 1 095 | 533 | ~1 630 |
| Что невозможно | 1 379 | 622 | ~2 000 |
| Нерусские буквы | 909 | 612 | ~1 520 |
| Поверни правильно | 1 476 | 791 | ~2 270 |
| Собери кости | 1 199 | 1 190 | ~2 390 |
| Комбинатор | 253 | — | ~253 |
| Поведенческий анализ (`pkg/behavior`) | 620 | — | ~620 |
| Proof-of-Work (`pkg/pow`) | 925 | 334 | ~1 260 |
| **Итого (без демо)** | **7 856** | **4 082** | **~11 943** |

### Демо

| Файл | Строк |
|---|---:|
| `demo/index.html` | 2 367 |
| `demo/captcha1_demo.html` | 1 242 |
| `demo/captcha2_demo.html` | 1 186 |
| `demo/captcha3_demo.html` | 1 309 |
| `demo/captcha4_demo.html` | 1 251 |
| `demo/captcha5_demo.html` | 1 169 |
| **Итого демо** | **8 524** |

### Общий итог

| Категория | Строк |
|---|---:|
| Go (бэкенд + тесты + пакеты) | ~7 860 |
| TypeScript/JS (фронтенд + PoW) | ~4 080 |
| Демо HTML | ~8 520 |
| **Всего** | **~20 460** |

---

## v6 — Changelog

### Новые капчи

**Капча 4: «Поверни правильно» (Rotation Match)**
- 80 процедурных силуэтов (SVG-path) в 8 категориях
- Многораундовая механика (1–3 раунда)
- Шаг поворота: 90° (лёгкий) → 45° → 60° → 30° (сложный)
- Canvas-рендеринг с визуальным шумом и рандомизацией

**Капча 5: «Собери кости» (Dice Sum)**
- Процедурный 3D-рендеринг костей на Canvas
- 8 цветовых схем, 4 стиля точек
- Интеллектуальная генерация дистракторов (суммы ±1 от целевого на сложных уровнях)
- Многораундовая механика (1–3 раунда)

### Обновлённый комбинатор
- 5 типов капч (вместо 3)
- Равномерный случайный выбор 1/5

### Переработанная демо-страница
- Новый дизайн в стиле cyberpunk/terminal: тёмная тема, неоновый зелёный акцент
- Все 5 капч в едином интерфейсе с вкладками
- Панель телеметрии: confidence, время ответа, раунд, статус
- Поведенческий анализ с визуализацией траектории мыши
- Адаптивный дизайн (desktop/tablet/mobile)

### Интеграция демо-файлов
- Все 5 `captchaX_demo.html` поддерживают `postMessage`-интеграцию с лаунчером
- Двусторонняя коммуникация: `setComplexity`, `reset` → iframe; `mouseMove`, `click`, `status`, `result` → parent

### Архитектурные улучшения

**Graceful shutdown:**
- All captcha backends now support graceful shutdown via `Close()` method
- Background cleanup goroutines use `select` with `stopCh` channel instead of bare `for range ticker.C`
- Prevents race conditions during deploy when cleanup goroutines outlive the main process

**Observability:**
- Consistent structured logging across all backends using `log/slog`
- Standardized log event names: `captcha.new_challenge`, `captcha.result`, `pow.rejected`, `combinator.dispatch`
- All log entries include `challenge_id`, `type`, `confidence`, `elapsed_ms`, `behavior_score` where applicable

**Rate limiting:**
- Per-instance token-bucket rate limiter added to the combinator's `NewChallenge` path
- Prevents burst abuse at the captcha layer before platform-level rate limiting

**Behavior scoring tests:**
- Comprehensive unit tests for the behavioral analysis engine (`pkg/behavior/behavior_test.go`)
- Covers trailer parsing, CRC validation, all five sub-scores, overall scoring, and `ApplyBehaviorConfidence`
