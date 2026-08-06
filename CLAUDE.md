# EVA.UA — Interaction Library

Живий showcase мікровзаємодій для EVA.UA: демо + код Web/iOS/Android для кожного
компонента, вбудований у Notion через `/embed`. Один файл — `index.html`,
задеплоєний через GitHub Pages. Аудиторія: стейкхолдери (дивляться демо) і
розробники (копіюють код).

## Джерело правди — Figma

Кожен компонент береться з `EVA Library WEB` (fileKey `q5jwASDszq6oGJBx5ogoKR`).
Коли даю посилання на конкретний вузол — **завжди** спочатку тягни реальні дані
через Figma MCP (`get_design_context` / `get_metadata`), а не вгадуй path/кольори
з опису. Точні координати path, fill, viewBox — тільки з Figma.

## Структура секції (повторюється на кожен компонент)

```html
<section class="interaction">
  <div class="interaction-head">
    <h2>Назва компонента (англ. термін в дужках, напр. "checkbox")</h2>
    <a class="figma-link" href="{вузол Figma}" target="_blank" rel="noopener">View in Figma →</a>
  </div>

  <div class="compare">
    <div class="compare-col is-bad">
      <span class="compare-tag bad">Зараз</span>
      <div class="compare-demo">...погана поведінка...</div>
    </div>
    <div class="compare-col is-good">
      <span class="compare-tag good">Пропозиція</span>
      <div class="compare-demo">...гарна поведінка...</div>
      <p class="status-line"></p>
      <div class="tokens"><span class="token">...</span></div>
      <label class="slow-toggle">...</label>  <!-- якщо є мережевий запит -->
    </div>
  </div>

  <details class="code-toggle">
    <summary>Показати код (Web / iOS / Android)</summary>
    <div class="code-tabs">...таби Web / iOS · UIKit / Android · Compose...</div>
  </details>
</section>
```

**Важливо:** у колонках "Зараз"/"Пропозиція" НЕМАЄ текстових описів-абзаців —
тільки бейдж + сама демка + токени. Пояснення тексту "не професійно виглядало" —
видалено навмисно, не повертати.

## Мова "Зараз" vs "Пропозиція"

- **Зараз** — навмисно погана поведінка, що імітує поточний прод: без transition
  взагалі (`style="transition:none;"` inline), клік → пауза ~800мс без жодного
  фідбека → раптовий стрибок у кінцевий стан. Без ховера, якщо в реальному
  проді його нема (перевіряти зі скріншотів користувача, не вигадувати).
- **Пропозиція** — press-фідбек одразу (<100мс), контур/елемент реагує на
  ховер, стан завантаження через trace-техніку (див. нижче), success —
  заливка через `clip-path` + `burst()` частинки де доречно.
- Кольори бейджів: `--bad-bg/--bad-text/--bad-line` (червоні), `--good-bg/--good-text/--good-line` (зелені).

## Motion-токени (в `:root`)

```css
--motion-duration-instant: 50ms;
--motion-duration-fast: 100ms;
--motion-duration-quick: 150ms;
--motion-duration-base: 200ms;
--motion-duration-expand: 250ms;
--motion-duration-moderate: 300ms;
--motion-duration-screen: 450ms;

--motion-easing-standard: cubic-bezier(0.2, 0, 0, 1);
--motion-easing-decelerate: cubic-bezier(0, 0, 0, 1);
--motion-easing-accelerate: cubic-bezier(0.3, 0, 1, 1);
--motion-easing-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
```
Кожна анімація посилається на конкретний токен (в `.tokens` бейджах під демкою).

## Техніка "trace" (стан завантаження)

Контур елемента промальовується по колу з нижньої точки, потім **у тому самому
напрямку** стирається (не назад/ping-pong, а продовжуючи рух — draw і erase
одним неперервним рухом). Для кожної форми:

1. Сэмплити точки контуру (`outline.getPointAtLength()`, ~100-160 точок) або,
   якщо форма — правильне коло, генерувати напряму через `sin`/`cos`.
2. Знайти найнижчу точку (`max y`) — це стартова точка.
3. Побудувати новий `<path class="trace-path">` з точок, що починаються там.
   Для heart/checkbox напрямок точок реверсується (clockwise), для нового
   компонента — перевіряй на око, який бік виглядає природно, і опиши в
   коментарі чому.
4. **КРИТИЧНО:** кінцеве значення kejyframe — окрема CSS-змінна `--neg-len`
   (обчислена в JS: `-traceLen`), НЕ `calc(-1 * var(--len))`. Браузери не
   завжди коректно інтерполюють `calc()` з CSS-змінною в animated keyframe —
   контур "стрибає" замість плавного стирання. Це вже раз ламалось, не
   повторювати.
5. Keyframes:
   ```css
   @keyframes X-trace {
     0%   { stroke-dashoffset: var(--len);   animation-timing-function: var(--motion-easing-decelerate); }
     50%  { stroke-dashoffset: 0;             animation-timing-function: var(--motion-easing-accelerate); }
     100% { stroke-dashoffset: var(--neg-len); }
   }
   ```
6. **Не переривати анімацію посеред циклу.** Коли реальна відповідь
   приходить, чекати `animationiteration` (спільна функція `waitForCycle(el)`
   у скрипті) і лише тоді показувати success — інакше виглядає як "різко
   промалювалось і зникло".
7. Для дуже маленьких елементів (~20px, напр. кулька toggle) — trace-контур
   виглядає "брудно" (накладання ліній). Замість нього: спокійне "дихання"
   (`scale` + `opacity` в циклі, без окремого спінера/кільця). НЕ обертовий
   спінер — на такому масштабі він виглядає метушливо.

## Оптимістичний UI (checkbox / radio)

За замовчуванням клік перемикає стан **миттєво, без loading-стану** (щоб не
відчувалось як довге очікування). Повний trace-loading показується, лише
якщо увімкнено тумблер "Повільний інтернет" поруч. Патерн:

```js
if (!slowToggle.checked) { succeed(); return; }  // швидкий шлях
wrap.classList.add('loading');
fakeRequest().then(() => waitForCycle(trace)).then(succeed);  // повільний шлях
```
Обидва шляхи ведуть в одну спільну функцію `succeed()`.

## Burst-частинки (успіх)

Однакова функція для heart / toggle / slider, лише колір і масштаб різні:
```js
function burst() {
  const n = 8, dist = 26; // heart: #212121; toggle/slider: #76BF44, менші (n=6-8, dist~16-26)
  for (let i = 0; i < n; i++) {
    const angle = (i / n) * Math.PI * 2;
    const dot = document.createElement('span');
    dot.style.cssText = 'position:absolute;...transform:translate(-50%,-50%);opacity:1;transition:transform .5s cubic-bezier(.2,.8,.3,1), opacity .5s ease-out;';
    wrap.appendChild(dot);
    void dot.offsetWidth; // ОБОВ'ЯЗКОВО forced reflow — без цього transition іноді
                           // не спрацьовує з першого кліку (браузер не встигає
                           // закомітити початковий стан)
    dot.style.transform = 'translate(calc(-50% + ' + Math.cos(angle)*dist + 'px), calc(-50% + ' + Math.sin(angle)*dist + 'px))';
    dot.style.opacity = '0';
    setTimeout(() => dot.remove(), 550);
  }
}
```

## Ховер

- Розповсюджується тільки на неактивний/невибраний стан (`if (active) return;`
  всередині hover-хендлера) — на вже обраному елементі зазвичай **нема**
  ховер-ефекту (як у heart/radio), якщо явно не попросили інше (як у toggle,
  де ховер додає soft shadow + border darken незалежно від стану).
- `supportsHover = window.matchMedia('(hover: hover)').matches` — гейтить
  ефект на touch-пристроях, де hover не повинен "залипати".
- Для маленьких контролів (checkbox/radio 20px) — темніє контур + легкий
  `scale(1.06)`. Для toggle — темніє рамка + `drop-shadow` на кульці +
  `scale(1.08)`, кольоровий акцент відрізняється для on/off стану.

## Іконки

Усі кастомні SVG-іконки (heart, checkbox, radio) рендеряться **20×20px**
(viewBox збігається з розміром) — не збільшувати штучно. Тап-область кнопки
може бути більшою (`.checkbox-btn` 44px) для зручності на тач, іконка
центрується всередині. Для списків (`.checkbox-row .is-compact`) тап-область
теж стискається до 20px, щоб gap іконка↔лейбл був точним (8px).

## Список із трьох пунктів (checkbox/radio)

Компонент показується не як одинока іконка, а як список 3 рядків
(`.checkbox-list` / `.checkbox-row`, gap 16px між рядками, 8px іконка↔лейбл,
лейбл "List item" — за замовчуванням, як у Figma-компоненті). Radio button:
**перший пункт обраний за замовчуванням** в обох колонках (Зараз і
Пропозиція), без анімації входу — просто виставлено на старті.

## Код-таби

`<details class="code-toggle">` — **згорнуті за замовчуванням**
("Показати код (Web / iOS / Android)"). Всередині — три таби, перемикаються
generic JS-обробником (`document.querySelectorAll('.interaction')` шукає
`.tab-btn`/`.tab-panel` в кожній секції, не хардкодить ID).

**iOS: тільки UIKit, НЕ SwiftUI.** Проєкт розробника побудований на UIKit —
SwiftUI-код (`View`, `@State`, `.onTapGesture`, `withAnimation`) для нього
непридатний, навіть як референс. Приклади мають використовувати:
- підклас `UIView`/`UIControl` замість `struct View`
- `UIView.animate(withDuration:delay:usingSpringWithDamping:)` для spring-анімацій
  замість `withAnimation`/`.spring()`
- `CAShapeLayer` + анімація `strokeStart`/`strokeEnd` для trace-контуру
  (прямий аналог dashoffset-техніки з Web-версії)
- `UITapGestureRecognizer` або `.addTarget(_:action:for: .touchUpInside)`
  замість `.onTapGesture`
- `UIImpactFeedbackGenerator`/`UINotificationFeedbackGenerator` — без змін,
  API однаковий в обох підходах
Заголовок вкладки: `iOS · UIKit` (не `iOS · SwiftUI`).

Android — Compose, без змін. iOS/Android — не 1:1 переклад Web-коду, а
нативний відповідник (spring-фізика замість cubic-bezier тощо). Коментарі
в коді — тільки функціональні ("що робить"), без пояснень типу "я не можу
зробити X" — це виглядає непрофесійно в прикладі для розробника.

## Мобільна адаптивність

`@media (max-width: 640px)`: `.compare` складається в один стовпець
(`border-left` → `border-top`), таби коду отримують horizontal scroll
(`overflow-x: auto`), шрифти й падінги зменшуються. Перевіряти після кожної
великої зміни розмітки.

## Деплой

Один файл `index.html` → GitHub repo `eva-motion-library` → GitHub Pages
(`Settings → Pages → Deploy from branch: main / root`) →
`https://roznovski.github.io/eva-motion-library/` → вбудовано в Notion через
`/embed`. Правки: редагувати файл, комітити в `main` — Pages і Notion-embed
підхоплюють автоматично, нічого окремо перевбудовувати не треба.
