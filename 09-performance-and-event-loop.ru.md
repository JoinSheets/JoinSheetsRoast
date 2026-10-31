+++
title = "Глава 9 — Производительность и event loop"
weight = 9
+++

Код: `view-table.js` — батчинг строк (`ROW_RENDER_BATCH`, `rowDescriptors`/
`materializeRow`, строки 533-689), `main.js`'s крах-watchdog
(`requestAnimationFrame`, строки 251-275, бегло видели в главе 6).

## 1. Теория

### Конвейер рендеринга браузера

Отображение кадра на экране — не одна операция, а последовательность стадий:
Style (пересчёт CSS-правил для изменённых узлов), Layout (вычисление
геометрии — где что находится и какого размера), Paint (заливка пикселей),
Composite (сборка слоёв в финальный кадр). JS-код запускает эту цепочку, но
не выполняет её сам и не выполняет её синхронно внутри себя: можно поменять
сотни DOM-узлов подряд одним синхронным куском кода, и Layout/Paint для всех
них случится один раз, позже, перед отрисовкой следующего кадра. При частоте
обновления экрана 60 раз в секунду на всю эту цепочку, вместе с
исполнившимся перед ней JS, остаётся примерно 16 миллисекунд (1000, делённое
на 60) — так называемый бюджет кадра; превысить его означает пропустить
кадр, что глаз воспринимает как рывок или подвисание.

### Event loop и опасность частых дорогих обработчиков

Браузер обслуживает события, таймеры, промисы и собственные шаги рендеринга
через единый механизм очередей задач: макрозадачи — это отдельные,
завершённые «тики» выполнения (один обработчик события, один таймер), между
которыми браузер успевает выполнить рендер-шаг, если он назрел; микрозадачи
(в частности, обработчики промисов) выполняются сразу после текущей
макрозадачи, но до следующего рендер-шага. Событие `scroll` при быстрой
прокрутке колесом генерирует десятки таких макрозадач в секунду — и если
обработчик каждой из них запускает что-то дорогое, браузер физически не
успевает «добежать» до собственного рендер-шага между соседними
срабатываниями: всё время съедает JS, кадры пропускаются, прокрутка
выглядит рваной или подвисает целиком.

Общий, независимый от конкретного фреймворка ответ на проблему «событие
срабатывает слишком часто для дорогого обработчика» — throttling
(ограничить обработчик одним реальным вызовом за фиксированный промежуток
времени) и debouncing (отложить вызов до момента, когда события перестали
сыпаться); обе техники десятилетиями реализуются библиотеками вроде
Lodash (`throttle`/`debounce`) и одинаково уместны что для `scroll`, что
для `resize`, что для ввода текста в поле поиска. Именно эта проблема, в её
частном варианте — полный `store.dispatch` на каждый `scroll`-тик, то есть
reducer → `render()` → `renderGrid()` → `clear()` и перестройка тысяч
`<tr>` (главы 3 и 6), — и была мотивирующей причиной батчинга построчного
рендера, добавленного в проект и разбираемого ниже: задача решена не универсальным throttle/debounce, а отдельным, узким
механизмом, обходящим дорогой путь `dispatch` вовсе.

### `requestAnimationFrame`: встроиться точно перед следующим кадром

`requestAnimationFrame(cb)` — не таймер общего назначения вроде `setTimeout`,
а запрос вызвать `cb` непосредственно перед тем, как браузер начнёт готовить
следующий кадр отрисовки. Это даёт гарантию порядка относительно рендеринга,
которой `setTimeout(cb, 0)` не даёт в принципе — у `setTimeout` нет никакой
связи с тем, на какой стадии конвейера рендеринга находится браузер в момент
срабатывания. Крах-watchdog в `main.js` использует эту гарантию в удвоенном
виде — два вложенных вызова `requestAnimationFrame`:

```js
requestAnimationFrame(function () {
  requestAnimationFrame(function () { storage.set('renderWatchdog', false); });
});
```

Первый вызов сработает перед следующим кадром — но сам этот кадр к этому
моменту ещё не гарантированно отрисован: спецификация гарантирует «перед
рендером», а не «после» него. Вложенный второй вызов ждёт ещё один полный
цикл — на практике это распространённый идиоматический приём, позволяющий
дождаться момента, когда браузер действительно прошёл через реальный
layout/paint текущего кадра, а не просто вот-вот начал его готовить.

### Что почитать:

- MDN, **Event loop** — макрозадачи, микрозадачи, порядок выполнения:
  https://developer.mozilla.org/ru/docs/Web/JavaScript/Event_loop
- MDN, **Critical rendering path** — Style → Layout → Paint → Composite:
  https://developer.mozilla.org/en-US/docs/Web/Performance/Critical_rendering_path
- MDN, **Window.requestAnimationFrame** — и почему это не `setTimeout`:
  https://developer.mozilla.org/ru/docs/Web/API/Window/requestAnimationFrame
- Lodash, **`throttle`** / **`debounce`** — стандартные ответы на слишком
  частые события, общие для любого фреймворка:
  https://lodash.com/docs/4.17.15#throttle
  https://lodash.com/docs/4.17.15#debounce
- MDN, **DocumentFragment** — зачем вставлять пачкой, а не по одному узлу:
  https://developer.mozilla.org/ru/docs/Web/API/DocumentFragment
- MDN, **Element.scrollTop / scrollHeight / clientHeight** — три числа, на которых держится весь расчёт порога догрузки:
  https://developer.mozilla.org/ru/docs/Web/API/Element/scrollTop
  https://developer.mozilla.org/ru/docs/Web/API/Element/scrollHeight
  https://developer.mozilla.org/ru/docs/Web/API/Element/clientHeight
- **Reflow / layout thrashing** — почему чередование чтения геометрии и записи в DOM особенно дорого (у нас этого удалось избежать, но знать стоит):
  https://developer.mozilla.org/en-US/docs/Glossary/Reflow
- **Chrome DevTools, Performance panel** — как читать флейм-чарт, Bottom-Up,
  Self Time:
  https://developer.chrome.com/docs/devtools/performance
- **IntersectionObserver** — современная альтернатива ручному
  `scroll`-обработчику для «догрузить, когда доскроллили»; проект его не
  использует, но для сравнения подходов полезно:
  https://developer.mozilla.org/ru/docs/Web/API/Intersection_Observer_API
- **clusterize.js** — библиотека, чей подход к виртуализации изучался перед
  реализацией батчинга в 0.31.0a1 (в проекте в итоге сделано проще):
  https://clusterize.js.org/
- **The Rust Book, гл. 8.3** «Hash Maps» — для Rust-трека ниже:
  https://doc.rust-lang.org/book/ch08-03-hash-maps.html
- **The Rust Book, гл. 10.3** «Lifetimes» — понадобится, как только
  `group_by` начнёт возвращать ссылки на элементы входного среза:
  https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html

## 2. Код

### Два прохода: дешёвые данные, дорогой DOM

```js
var rowDescriptors = [];
// ... collectLevel/collectDeepKeys заполняют rowDescriptors ЦЕЛИКОМ, для ВСЕХ строк,
// без единого вызова el()
var materializeRow = function (rd) {
  // единственное место, создающее <tr> из описания — вызывается только для текущего окна
};
var renderedCount = Math.min(ROW_RENDER_BATCH, rowDescriptors.length);
var bodyRows = [];
for (var ri = 0; ri < renderedCount; ri++) bodyRows.push(materializeRow(rowDescriptors[ri]));
```

Комментарий в коде (строки 533-538) прямо формулирует находку: на таблице в
несколько тысяч строк дорогая часть — именно СОЗДАНИЕ DOM-узлов, а не обход/
группировка самих данных (`collectLevel` проходит по всем строкам без
батчинга — это быстро, это просто перебор JS-объектов). `rowDescriptors` —
обычные плоские объекты (`{kind:'member', row:r, ridx:...}` или `{kind:'grp',
...}`) — создание тысячи таких объектов практически бесплатно по сравнению
с созданием тысячи настоящих `<tr>` с дочерними `<td>`.

### Скролл-слушатель — минуя `store.dispatch` полностью

```js
var scrollEl = node.querySelector('.scroll');
var tbodyEl = scrollEl && scrollEl.querySelector('tbody');
var renderMore = function (targetCount) {
  var target = Math.min(rowDescriptors.length, targetCount);
  if (target <= renderedCount || !tbodyEl) return;
  var frag = document.createDocumentFragment();
  for (var i = renderedCount; i < target; i++) frag.appendChild(materializeRow(rowDescriptors[i]));
  tbodyEl.appendChild(frag);
  renderedCount = target;
};
if (scrollEl && renderedCount < rowDescriptors.length) {
  scrollEl.addEventListener('scroll', function () {
    if (renderedCount >= rowDescriptors.length) return;
    var remaining = scrollEl.scrollHeight - scrollEl.scrollTop - scrollEl.clientHeight;
    if (remaining < scrollEl.clientHeight) renderMore(renderedCount + ROW_RENDER_BATCH);
  });
}
```

Ключевое решение: этот слушатель **не вызывает `d(...)`** ни разу. Если бы
он диспетчил действие — весь путь reducer→render→renderGrid→clear+rebuild
ВСЕХ уже отрендеренных строк повторялся бы на каждый `scroll`-тик, обесценивая
саму идею батчинга. Вместо этого `renderMore` работает напрямую с реальными
DOM-узлами: `document.createDocumentFragment()` — специальный, невидимый
контейнер-черновик, в который можно добавить много узлов, а затем вставить
их ВСЕ одним `tbodyEl.appendChild(frag)` — одна операция вставки в реальное
дерево вместо N отдельных, что снижает число "мутаций", которые браузер
должен будет учесть при следующем Layout.

Порог догрузки: `remaining < scrollEl.clientHeight` — догружаем ЗАРАНЕЕ,
когда до конца проскроллированного остаётся меньше одного "экрана" высоты, а
не впритык к нулю — так пользователь не увидит пустое пространство даже при
резком скролле колесом.

### `currentEnsureMemberRendered` — поиск за пределами окна

```js
currentEnsureMemberRendered = function (ridx) {
  for (var i = renderedCount; i < rowDescriptors.length; i++) {
    if (rowDescriptors[i].kind === 'member' && rowDescriptors[i].ridx === ridx) { renderMore(i + 1); return; }
  }
};
```

Используется из `revealRow` (quick-поиск, разбирали мельком в главе 7) —
линейный проход по оставшейся части `rowDescriptors` в поисках конкретной
строки, затем `renderMore` именно до неё. `O(n)` в худшем случае — но `n`
здесь — количество ОПИСАНИЙ (дешёвых объектов), не DOM-узлов, и происходит
один раз на клик "найти", не на каждый скролл-тик — цена несопоставима с
тем, что заменяет (полный передиспетч и ререндер).

## 3. Практика

**9.1** Таблица на 15000 строк, `ROW_RENDER_BATCH = 1000`. Пользователь
плавно долистывает от начала до конца. Сколько раз вызовется `renderMore` с
реально новым (`target > renderedCount`) батчем, прежде чем все строки
окажутся в DOM? А сколько раз сработает само событие `scroll` (порядок
величины — десятки? сотни?) — и почему эти два числа совсем не обязаны
совпадать.

**9.2** В более ранней (до батчинга) версии тот же скролл по той же таблице
вызывал бы полный `store.dispatch`+`render()` на каждое срабатывание
`scroll`. Опишите словами разницу в "стоимости одного тика" между старой и
новой версией — что именно убрали из пути выполнения (reducer? `clear()`?
полная перестройка ВСЕХ, а не только новых строк?).

**9.3** `renderMore` не использует `throttle`/`debounce` (искусственную
задержку между вызовами) — слушатель срабатывает буквально на каждый
`scroll`-тик без ограничения частоты. Объясните, почему это ОК именно здесь
(что конкретно внутри `renderMore` дешёво, если условие `target <=
renderedCount` истинно большую часть времени — пустой early-return), и когда
throttle/debounce были бы НЕОБХОДИМЫ (если бы сама проверка внутри
обработчика была дорогой, а не O(1) арифметикой).

## 4. DevTools

**9.4 — Performance, запись реального скролла.** Загрузите большой пресет
(или временно поднимите тестовые данные), откройте «Таблицу», откройте
Performance, нажмите запись, прокрутите колесом от начала таблицы до конца
на несколько тысяч строк, остановите запись. Найдите на временной шкале
полосу **FPS** сверху — есть ли красные отметки ("long frames")? Откройте
**Bottom-Up** — найдите самую "тяжёлую" по собственному времени (Self Time)
функцию за всю запись — это `materializeRow`? `renderMore`? что-то другое?

**9.5 — эксперимент с размером батча.** Через Sources → Overrides (глава 1)
временно отредактируйте строку `var ROW_RENDER_BATCH = 1000;` на `50`.
Перезагрузите, повторите запись Performance из 9.4 на той же таблице.
Сравните флейм-чарт: стало ли ВИДИМО больше отдельных мелких всплесков
Scripting во время скролла (батчи чаще, но каждый меньше)? Любой ли из двух
вариантов визуально "рванее" по FPS-полосе? Отмените override, когда
закончите.

**9.6 — DOM breakpoint на `<tbody>`.** В Elements поставьте
**Break on → Subtree modifications** на `<tbody>`. Прокрутите колесом один
раз на расстояние, вызывающее ровно одну догрузку (`renderMore`). Когда
сработает брейкпоинт — в Call Stack убедитесь, что путь идёт через
`scroll`-слушатель → `renderMore`, а НЕ через `store.dispatch`/`render` (это
эмпирическое подтверждение того, что батчинг действительно обходит весь
путь редьюсера).

## 5. Rust-трек

Эта глава — удачный момент для капстоуна-подготовки: `collectLevel`/
`collectDeepKeys` (группировка) и `aggregateGroup` (`entities.js`, вызывается
из `materializeRow` для итоговой строки группы) — чистые, CPU-bound операции
над большими массивами данных, без какого-либо DOM. Задание: возьмите
функцию группировки (`groupBy` из `util.js` + сам обход `collectLevel`,
упрощённо — "сгруппируй массив объектов по значению одного поля") и
перепишите её на Rust:

```rust
use std::collections::HashMap;

fn group_by<'a, T, K: Eq + std::hash::Hash, F: Fn(&T) -> K>(items: &'a [T], key_fn: F) -> HashMap<K, Vec<&'a T>> {
    let mut groups: HashMap<K, Vec<&'a T>> = HashMap::new();
    for item in items {
        groups.entry(key_fn(item)).or_insert_with(Vec::new).push(item);
    }
    groups
}
```

Сгенерируйте (или напишите вручную) массив из, скажем, 100 000 синтетических
"строк" (простых структур с 2-3 полями) и замерьте время выполнения вашей
`group_by` через `std::time::Instant` — запишите число. Это — опорная точка
для главы 10: прежде чем переносить что-либо в WASM, стоит знать, насколько
быстро эта же логика выполняется в "чистом" Rust без накладных расходов на
пересечение границы JS↔WASM.

## 6. Вопросы для самопроверки

Сможете ли вы объяснить коллеге одним абзацем, ПОЧЕМУ скролл-слушатель
`renderMore` в принципе не может быть реализован через `store.dispatch`, не
теряя смысла самого батчинга? Если да — глава пройдена.
