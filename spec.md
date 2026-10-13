# Техническая спецификация (Technical Specification)
## Автономное веб-приложение «Техасский Холдем» (Texas Hold'em Poker)

---

### 1. Общие сведения и Архитектурные Ограничения

#### 1.1. Назначение документа
Настоящая техническая спецификация определяет архитектуру, модели данных, алгоритмы и требования к реализации single-page/single-file приложения Texas Hold'em Poker. Документ случит эталонным руководством для инженеров-разработчиков и систем автоматической генерации/рефакторинга кода (включая Antigravity CLI).

#### 1.2. Ограничения среды (Technical Constraints)
1. **Single-File Architecture:** Весь исходный код (HTML, CSS-стили, JavaScript-логика, SVG-ресурсы, звуковые синтезаторы) должен находиться строго в одном файле `index.html`.
2. **Zero Dependencies:** Запрещено использование внешних фреймворков (React, Vue, jQuery), CSS-препроцессоров (Tailwind, Bootstrap) и сторонних библиотек (Lodash, Tone.js, Howler.js).
3. **Zero Network Calls:** Приложение работает полностью оффлайн (ограничение CORS/Fetch/XHR на стороне пользователя отсутствует, внешние CDN и изображения не загружаются).
4. **Target Standard:** HTML5, CSS3 (CSS Variables, Flexbox, Grid), Vanilla JS (ES6+ Strict Mode).

---

### 2. Модульная Структура Кода (Фрагментация JS)

Внутри тега `<script>` структура кода разделена на следующие изолированные пространства имен / модули (ES6 Pattern / Object Literals):

```
┌─────────────────────────────────────────────────────────────┐
│                       UI Controller                         │
│   (DOM Event Listeners, Modal Handler, Render Engine)       │
└──────────────┬──────────────────────────────┬───────────────┘
               │                              │
┌──────────────▼──────────────┐┌──────────────▼───────────────┐
│        Poker Engine         ││         Audio Engine         │
│ (Game State, Round Control, ││ (Web Audio API Synthesizer,  │
│  Pot Manager, Side Pots)    ││  Sound Effect Generators)    │
└──────────────┬──────────────┘└──────────────────────────────┘
               │
┌──────────────▼──────────────┐
│     Evaluator & AI Module   │
│ (Combination Ranker, Kicker │
│  Calculator, Bot Decision)  │
└─────────────────────────────┘
```

---

### 3. Модели Данных (Data Structures)

#### 3.1. Структура Карты (`Card`)
```javascript
/**
 * @typedef {Object} Card
 * @property {string} suit - Масть: 'H' (Hearts), 'D' (Diamonds), 'C' (Clubs), 'S' (Spades)
 * @property {number} value - Достоинство: от 2 до 14 (11=J, 12=Q, 13=K, 14=A)
 */
```

#### 3.2. Структура Игрока (`Player`)
```javascript
/**
 * @typedef {Object} Player
 * @property {number} id - Уникальный идентификатор (0: Игрок-Человек, 1-3: Боты)
 * @property {string} name - Отображаемое имя
 * @property {string} archetype - Стиль ИИ ('tight', 'aggressive', 'loose', 'human')
 * @property {number} chips - Текущий баланс фишек
 * @property {number} currentBet - Ставка в текущем раунде торгов
 * @property {number} totalBetInHand - Суммарный взнос в банк за всю раздачу
 * @property {Card[]} cards - Карты на руках (2 карты)
 * @property {boolean} folded - Флаг сброса руки
 * @property {boolean} isAllIn - Флаг статуса Ва-банк
 * @property {boolean} isDealer - Флаг позиционной кнопки (Button)
 */
```

#### 3.3. Состояние Игровой Сессии (`GameState`)
```javascript
/**
 * @typedef {Object} GameState
 * @property {'PREFLOP'|'FLOP'|'TURN'|'RIVER'|'SHOWDOWN'|'IDLE'} stage - Текущая фаза
 * @property {Card[]} deck - Колода карт
 * @property {Card[]} communityCards - Общие карты на столе (0-5)
 * @property {Player[]} players - Массив из 4 игроков
 * @property {number} pot - Главный банк (Main Pot)
 * @property {Array<{amount: number, eligiblePlayerIds: number[]}>} sidePots - Побочные банки
 * @property {number} currentBet - Текущая максимальная ставка для колы
 * @property {number} minRaise - Минимальный шаг рейза
 * @property {number} activePlayerIdx - Индекс ходящего игрока (0..3)
 * @property {number} dealerIdx - Индекс баттона дилера
 * @property {number} smallBlind - Размер малого блайнда ($10)
 * @property {number} bigBlind - Размер большого блайнда ($20)
 * @property {number} gameSpeed - Множитель скорости задержек (1 или 2)
 */
```

---

### 4. Детализация Алгоритмов и Математики

#### 4.1. Генерация и перемешивание колоды (Fisher-Yates)
Используется алгоритм Дурштенфельда / Фишера — Йетса с вызовом `crypto.getRandomValues()` для достижения криптографической случайности раздачи.

```javascript
function createAndShuffleDeck() {
  const suits = ['H', 'D', 'C', 'S'];
  const deck = [];
  for (let s of suits) {
    for (let v = 2; v <= 14; v++) {
      deck.push({ suit: s, value: v });
    }
  }
  
  // Crypto-safe Fisher-Yates Shuffle
  const array = new Uint32Array(deck.length);
  window.crypto.getRandomValues(array);
  for (let i = deck.length - 1; i > 0; i--) {
    const j = array[i] % (i + 1);
    [deck[i], deck[j]] = [deck[j], deck[i]];
  }
  return deck;
}
```

#### 4.2. Оценка комбинаций (Evaluator Engine)
Из 7 карт (2 на руках + 5 общих) генерируются все комбинации из 5 карт (всего $\binom{7}{5} = 21$ вариант). Каждая комбинация из 5 карт оценивается функцией `evaluate5CardHand(hand)`.

##### Ранжирование Категорий (Category Rank):
9. **Royal Flush** (Стрит-флеш до туза)
8. **Straight Flush** (5 карт одной масти подряд)
7. **Four of a Kind** (Каре)
6. **Full House** (Тройка + Пара)
5. **Flush** (5 карт одной масти)
4. **Straight** (5 карт подряд, учтен туз-низ: A-2-3-4-5)
3. **Three of a Kind** (Сет / Тройка)
2. **Two Pair** (Две пары)
1. **One Pair** (Одна пара)
0. **High Card** (Старшая карта)

##### Формат результата оценки:
Возвращается объект вида `{ rank: number, score: number, name: string, best5: Card[] }`, где `score` — уникальное сравнимое значение (или вектор `[rank, kicker1, kicker2, ...]` для покомпонентного сравнения).

#### 4.3. Алгоритм Расчета Побочных Банков (Side Pots)
При формировании побочных банков после завершения торгов:
1. Вычисляются уникальные уровни ставок игроков, пошедших `All-In`, а также общие взносы остальных участников (`totalBetInHand`).
2. Взнос каждого игрока распределяется снизу вверх по уровням (Pots).
3. Неуравненные остатки ставок (Uncalled bets) возвращаются игроку, перебившему всех по сумме, до раздачи побочных банков.

---

### 5. Логика Искусственного Интеллекта (Bot Decision Matrix)

ИИ принимает решение на основе силы стартовой руки (Pre-flop) или абсолютной/потенциальной силы комбинации (Post-flop) с добавлением рандомизации (вероятность блефа).

#### 5.1. Пре-флоп оценка рук (Hand Equity Tier):
* **Tier 1 (Топовые):** AA, KK, QQ, JJ, AKs, AKo.
* **Tier 2 (Сильные):** TT-77, AQ, AJ, ATs, KQ.
* **Tier 3 (Маргинальные / Коннекторы):** 66-22, одномастные коннекторы (89s, 9Ts, JTs).
* **Tier 4 (Мусор):** Все остальные сочетания.

#### 5.2. Модель поведения по архетипам:

| Показатель / Поведение | Виктор «Скала» (Tight-Passive) | Елена «Акула» (Aggressive) | Макс «Джокер» (Loose/Wildcard) |
| :--- | :--- | :--- | :--- |
| **Диапазон входа (VPIP)** | ~15% (Только Tier 1-2) | ~35% (Tier 1-3) | ~65% (Tier 1-4) |
| **Агрессия на флопе (C-Bet)** | Низкая (Только при попадании в сет/пару) | Высокая (80% при чеке игрока) | Средняя (Рандомный All-In на дро) |
| **Вероятность блефа** | < 2% | ~25% | ~40% |
| **Реакция на рейз** | Сброс при отсутствии топ-пары+ | Перерейз / Колл | Колл / All-In |

---

### 6. Синтезатор Звуков (Web Audio API Engine)

Звуки создаются через динамический `AudioContext` без обращения к файловой системе.

```javascript
class SoundEngine {
  constructor() {
    this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    this.muted = localStorage.getItem('poker_muted') === 'true';
  }

  // Звук раздачи карты (белый шум + спад громкости)
  playCardSlide() {
    if (this.muted) return;
    const bufferSize = this.ctx.sampleRate * 0.08;
    const buffer = this.ctx.createBuffer(1, bufferSize, this.ctx.sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
    
    const noise = this.ctx.createBufferSource();
    noise.buffer = buffer;
    const filter = this.ctx.createBiquadFilter();
    filter.type = 'bandpass';
    filter.frequency.value = 1200;
    
    noise.connect(filter);
    filter.connect(this.ctx.destination);
    noise.start();
  }

  // Звук фишек (Синусоидальный импульс высоты)
  playChipClack() {
    if (this.muted) return;
    const osc = this.ctx.createOscillator();
    const gain = this.ctx.createGain();
    osc.type = 'sine';
    osc.frequency.setValueAtTime(2200, this.ctx.currentTime);
    osc.frequency.exponentialRampToValueAtTime(400, this.ctx.currentTime + 0.05);
    gain.gain.setValueAtTime(0.3, this.ctx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, this.ctx.currentTime + 0.05);
    osc.connect(gain);
    gain.connect(this.ctx.destination);
    osc.start();
    osc.stop(this.ctx.currentTime + 0.05);
  }
  
  // Дополнительно: playCheck(), playFold(), playWin()
}
```

---

### 7. Интерфейс и Верстка (UI/UX Specification)

#### 7.1. CSS Иерархия Layout:
1. **Header / Bar Controls:** Кнопка Mute, Лог раздачи, Шпаргалка комбинаций, Переключатель скорости (1x/2x).
2. **Poker Table Container (`.poker-table`):**
   * CSS Grid / Absolute positioning для размещения 4 местоположений (Bottom: Вы, Left: Скала, Top: Акула, Right: Джокер).
   * Овальное сукно с `border-radius: 150px`, радиальным градиентом `#0f5228` -> `#072b14` и внешней рамой с эффектом кожи (`box-shadow`, `#2b1704`).
3. **Community Cards Area:** Центр стола. Фишки Пота, общие карты с анимацией CSS Flip/Slide.
4. **Action Panel (`.action-panel`):**
   * Фиксирована снизу экрана на мобильных устройствах.
   * Элементы управления: `Fold`, `Check/Call`, `Bet/Raise`, слайдер суммы ставки + кнопки быстрых пресетов (`Min`, `1/2 Pot`, `Pot`, `All-In`).

#### 7.2. Адаптивные контрольные точки (Breakpoints):
* **Desktop (≥ 1025px):** Полноразмерный стол $1000 \times 600\text{px}$.
* **Tablet (641px - 1024px):** Масштабирование через `transform: scale()` или пропорциональный Flex/Grid.
* **Mobile Portrait (≤ 640px):** Вертикальная ориентация стола (высота больше ширины), кнопки панелей занимают 100% ширины нижней зоны (Thumb Zone).

---

### 8. Интерактивные Модальные Окна и Состояния

1. **Cheat Sheet Modal (`#cheatSheetModal`):**
   * Иерархический список от Роял-флеша до Старшей карты с визуальными примерами карт (мини-SVG / текстовые глифы `♠ ♥ ♦ ♣`).
2. **Game Log Modal (`#logModal`):**
   * Хронологический вывод всех событий (`[Пре-флоп] Бот 1 ставит $20`, `Игрок сбрасывает`).
3. **Rebuy Modal (`#rebuyModal`):**
   * Автоматически срабатывает, когда `players[0].chips === 0` и текущая раздача завершена. Предоставляет кнопку «Получить $1,000».

---

### 9. Критерии Приемки (Acceptance Criteria for Antigravity CLI)

1. **Сборка:** Весь проект генерируется в одном файле `index.html` и успешно открывается локально по протоколу `file:///`.
2. **Валидация правил:**
   * Очередность ходов корректна, блайнды списываются автоматически.
   * При фолде 3 игроков банк передается оставшемуся без вскрытия карт.
   * На шоудауне правильно подсвечивается победитель и корректно определяется сильнейший вектор из 5 карт.
3. **Отсутствие консольных ошибок:** При любых действиях пользователя (быстрое кликанье, All-In, изменение размеров окна) браузерная консоль не содержит `Uncaught TypeError` или `NaN` значений в банках/стеках.