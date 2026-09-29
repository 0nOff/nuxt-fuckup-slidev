---
theme: default
title: Я переезжаю на Nuxt. Если что — я лоханулся
author: Пётр Белобородов
info: 15-минутный доклад для факап-митапа Магнит Тех
transition: fade-out
duration: 15min
timer: countdown
lineNumbers: true
drawings:
  persist: false
---

<div class="eyebrow">Факап-митап · Магнит Тех</div>

# Я переезжаю на Nuxt

## Если что — я лоханулся

<div class="cover-meta">
  <span>Пётр Белобородов</span>
  <span>networkly.app</span>
</div>

<!--
0:00–0:40

Я Пётр. В соло делаю Networkly — каталог IT-мероприятий.
Решил перевезти довольно странный, но рабочий PHP+Vue-монолит на Nuxt.
Сразу спойлер: быстрее не стало.
-->

---
layout: two-cols-header
---

# Кто я и что я делаю

::left::

<div class="big-copy">
  <strong>Пётр</strong><br>
  fullstack PHP + JS
</div>

В соло делаю **networkly.app** — каталог IT-мероприятий.

::right::

<div class="stack-list">
  <span>PHP / Symfony</span>
  <span>Vue / Pinia</span>
  <span>V8 внутри PHP</span>
  <span class="accent">теперь ещё Nuxt</span>
</div>

<!--
0:40–1:25

Коротко представиться. Не объяснять продукт подробно: достаточно сказать,
что это живой сервис, который я разрабатываю один, поэтому весь технический
долг — персональный и очень близкий.
-->

---
layout: center
class: statement-slide
---

<div class="eyebrow">Завязка</div>

# Мне захотелось<br><span class="accent">нормальный современный фронтенд</span>

<p class="muted lead">Nuxt уже существует. Агент уже существует.<br>Что может пойти не так?</p>

<!--
1:25–2:20

Мотивация была нормальной: получить современный frontend-фреймворк,
понятный SSR и постепенно уйти от самодельной схемы.

С агентом миграция казалась не отдельным проектом, а задачей на вечер.
-->

---
layout: two-cols-header
---

# Как было устроено: клиент

::left::

- Vue-компоненты в PHP-страницах
- из компонентов собирались Web Components
- Pinia оставался единым благодаря chunk splitting
- разные части страницы жили как компоненты

::right::

```mermaid {scale: 0.78}
flowchart TD
  T[PHP / Twig page] --> W[Web Components]
  V[Vue components] --> W
  W --> P[Shared Pinia chunk]
  P --> A[HTTP API]

  classDef hot fill:#ff5c4d,color:#0b0d0f,stroke:#ff5c4d;
  classDef base fill:#181b20,color:#f4f0e8,stroke:#555b66;
  class W,P hot;
  class T,V,A base;
```

<!--
2:20–3:20

Подчеркнуть: система была странной, но не случайной. Компоненты уже были,
данные и состояние уже разделялись, а один Pinia переиспользовался между
островами через chunk splitting.
-->

---
layout: default
---

# Один Pinia → Vue → Web Component

````md magic-move {lines: true}
```ts
// utils/webComponentPinia.ts
export const webComponentPinia = createPinia()
```

```ts
// eventPersonalWeek.ts
import EventPersonalWeek from './view/EventPersonalWeek.vue'
import { webComponentPinia } from './utils/webComponentPinia'

const Element = defineCustomElement(EventPersonalWeek, {
  shadowRoot: false,
  configureApp(app) {
    app.use(webComponentPinia)
  },
})
```

```ts
// eventPersonalWeek.ts
import EventPersonalWeek from './view/EventPersonalWeek.vue'
import { webComponentPinia } from './utils/webComponentPinia'

const Element = defineCustomElement(EventPersonalWeek, {
  shadowRoot: false,
  configureApp(app) {
    app.use(webComponentPinia)
  },
})

customElements.define('event-personal-week', Element)
```
````

<!--
3:20–4:00

Быстро прокликать три состояния: Pinia создаётся один раз в отдельном модуле;
обычный Vue SFC передаётся в defineCustomElement; configureApp подключает тот
же Pinia; customElements.define регистрирует тег.
-->

---
layout: two-cols-header
---

# Как было устроено: сервер

::left::

```mermaid {scale: 0.76}
flowchart TD
  D[PHP worker — демон] --> V[V8 как .so]
  S[Подготовленные данные] --> P[Pinia snapshot]
  P --> V
  V --> H[SSR HTML]
  H --> B[Браузер]

  classDef hot fill:#a6ff5f,color:#0b0d0f,stroke:#a6ff5f;
  classDef base fill:#181b20,color:#f4f0e8,stroke:#555b66;
  class V,P hot;
  class D,S,H,B base;
```

::right::

<div class="aside">
  PHP не умирал.<br>
  V8 жил внутри него.<br>
  Специальные версии компонентов рендерились на сервере.
</div>

<!--
4:00–4:25

Это место можно рассказывать с удовольствием: PHP работал демоном,
V8 подгружался как so-библиотека, а в SSR прокидывались заранее
подготовленные данные для Pinia.

Да, звучит дико. Но оно работало.
-->

---
layout: default
---

# SSR: webpack → PHP → V8 → Pinia

````md magic-move {lines: true}
```php
// EventService.php
$snapshotSourcePath = __DIR__.'/../../public/build/eventsListSsrFunc.js'; // webpack output
$snapshotCacheKey = 'snapshot_'.hash_file('sha256', $snapshotSourcePath);

$snapshot = $this->ssrCachePool->get($snapshotCacheKey,
    static function () use ($snapshotSourcePath) {
        return V8Js::createSnapshot(file_get_contents($snapshotSourcePath)); // read → V8 cache
    });
```

```php
// Смысл реального кода: эмулируем API-запросы внутри PHP
$apiResponses = [
    'events'   => emulateGet('/api/events?...'),
    'tags'     => emulateGet('/api/tags?...'),
    'geonames' => emulateGet('/api/geonames/published'),
]; // никакого HTTP
```

```php
// EventService.php
$context = json_encode(['events' => $normalizedEvents, /* ... */]); // payload для Pinia
$v8 = new V8Js('php', [], $snapshot); // кешированный webpack-бандл

$html = (new V8($v8))->run(
    "var context={$context}; renderSsr(context).then(html => print(html));"
);
```

```ts
// eventsListSsrFunc.ts — код уже выполняется внутри V8
const { app } = BaseEventsList(context.locale)

useEventsStore().setStateFromResponse(JSON.parse(context.events)) // PHP → Pinia
usePublishedGeonamesStore().applyGeonames(JSON.parse(context.geonames))

return renderToString(app)
```
````

<!--
4:25–4:50

Четыре быстрых клика: PHP читает webpack-бандл и кеширует V8 snapshot;
запросы к API эмулируются прямо внутри PHP, без HTTP; ответы складываются
в context и уходят в V8; JS гидратирует Pinia и делает renderToString.
-->

---
layout: center
class: metric-slide
---

<div class="eyebrow">План был простой</div>

# Отдать страницу агенту → получить Nuxt → переключить маршрут

<div class="mega-commit">
  <span>303 файла</span>
  <span class="plus">+34 969</span>
  <span class="minus">−11 369</span>
</div>

<p class="muted">Один исходный коммит. Что-то современное явно произошло.</p>

<!--
4:50–5:40

Исходный Nuxt-коммит c446191c затронул 303 файла.
Это не обязательно плохо само по себе, но отлично показывает разницу
между ожидаемым переносом страницы и фактическим созданием второго приложения.
-->

---
layout: two-cols-header
---

# Минус №1. Страница ждёт вообще всех

::left::

```ts {2|3-8|9-12|all}
const event = await api('/events/:id')
const base = await Promise.allSettled([
  schedule(),
  registrationSettings(),
  features(),
  communityEvents(),
  recommendationProfile(),
])
const related = await Promise.allSettled([
  sameDayEvents(),
  similarEvents(),
])
```

::right::

<div class="failure-copy">
  <strong>SSR critical path</strong>
  стал равен всей странице
</div>

<ul class="compact">
  <li>медленный первый ответ</li>
  <li>раздутый composable</li>
  <li>страница знает обо всех детях</li>
  <li>ручная раскладка по четырём store</li>
</ul>

<!--
5:40–7:20

Утром начал тестировать. Формально страница работала — просто медленно.

Агент собрал всю страницу в aggregate loader: сначала событие, потом пять
запросов, потом ещё два, затем разложил результат по четырём Pinia stores.

Важно: это не хитрая проблема Nuxt. Подход был странным сам по себе.
Первое, что пришлось сделать, — объяснить агенту недопустимость такой схемы.
-->

---
layout: image
image: /diff-use-event-page.png
class: screenshot-slide
---

<div class="screenshot-caption">
  <span class="eyebrow">Исправление 03b6778c</span>
  <strong>−66 строк из useEventPage</strong>
  <small>Данные вернулись к компонентам, которым они нужны</small>
</div>

<!--
7:20–8:10

Показать реальный diff. После переделки composable снова загружал только
само мероприятие. Расписание, регистрация, похожие события и прочие блоки
вернулись к своим компонентам.
-->

---
layout: two-cols-header
---

# Минус №2. Две версии правил мета-информации

::left::

<div class="truth-box">
  <span>Symfony</span>
  <strong>title, description,<br>дата, schema.org,<br>приватный адрес</strong>
</div>

::right::

<div class="truth-box danger">
  <span>Nuxt / eventSeo.ts</span>
  <strong>title, description,<br>дата, schema.org,<br>приватный адрес</strong>
</div>

<div class="bottom-line">
  Переделал на HTTP-ручку. Сомнительно? Да.<br>
  Но один лишний запрос дешевле двух версий правил.
</div>

<!--
8:10–9:50

Не называть это абстрактной «доменной политикой». Это конкретные правила
оформления мета-информации и structured data, включая приватный адрес.

Агент независимо реализовал их в eventSeo.ts. Пришлось удалить копию
и получать уже вычисленный результат с backend.
-->

---
layout: two-cols-header
---

# Минус №3. Nuxt поехал — legacy сломался

::left::

<div class="code-panel danger">
  <small>Было</small>
  <code>loadOnFilterChange?: boolean</code>
  <strong>Nuxt → false</strong>
  <strong>Legacy → undefined → false</strong>
</div>

::right::

<div class="code-panel success">
  <small>Стало</small>
  <code>withDefaults(defineProps(), {</code>
  <strong>loadOnFilterChange: true</strong>
  <code>})</code>
</div>

<div class="bottom-line">
  Общий компонент адаптировали под Nuxt —<br>
  старый каталог перестал загружать данные при смене фильтра.
</div>

<!--
9:50–11:10

Кандидат на третий факап из реального исправления 4173a3ec.

В общий EventsListForMainSite добавили необязательный prop. Для Nuxt явно
передавали false, а старые потребители не передавали ничего и тоже получили
falsy. Исправление — legacy-compatible default true.

Если этот эпизод не подходит по ощущениям, заменить слайд, не меняя структуру.
-->

---
layout: center
class: statement-slide
---

<div class="eyebrow">Что получилось</div>

# Код появился быстрее,<br>чем я научился его <span class="accent">валидировать</span>

<div class="fix-stream">
  composable → SEO → routing → legacy → assets → analytics → tests
</div>

<!--
11:10–12:15

Не превращать агента в злодея. Проблема в разрыве скорости:
код генерируется мгновенно, а понимание нового фреймворка — нет.

Уверенный рабочий код ещё надо уметь отличить от уверенно выглядящего.
-->

---
layout: default
---

# Что я из этого вынес

<div class="lessons">
  <div><span>01</span><strong>Новый фреймворк всё равно придётся изучить.</strong></div>
  <div><span>02</span><strong>Нельзя делегировать ещё не приобретённый навык.</strong></div>
  <div><span>03</span><strong>Тесты сделали этот эксперимент вообще возможным.</strong></div>
  <div><span>04</span><strong>Я всё ещё не решил: продолжать кактус или откатить.</strong></div>
</div>

<!--
12:15–13:45

Главная идея из заметки: нельзя просто взять фреймворк и перенести на него всё.
Его надо изучить, а это время.

Без тестов я бы не стал даже пробовать. И честный текущий результат:
я пока не уверен, стоит ли продолжать миграцию.
-->

---
layout: center
class: final-slide
---

# Резюме-девелопмент — сосёт.

<div class="final-word">Но хорошо, что я попробовал.</div>

<!--
13:45–15:00

«Резюме-девелопмент» — это выбор технологий ради строчки в резюме,
а не ради пользы для продукта, иногда ещё и без достаточной квалификации.
Не development вообще. Сам эксперимент всё равно оказался полезным.
-->
