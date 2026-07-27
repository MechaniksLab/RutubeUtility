# RutubeUtility

Набор JavaScript-скриптов для массового удаления контента с собственного канала в **RUTUBE Studio**.

В репозитории представлены два отдельных варианта:

1. [Удаление обычных видео](#1-удаление-обычных-видео)
2. [Удаление всех Shorts](#2-удаление-всех-shorts)

> [!CAUTION]
> Удаление необратимо. Перед подтверждением внимательно проверьте таблицу найденных роликов в консоли браузера. Восстановить удалённые видео средствами RUTUBE может быть невозможно.

> [!IMPORTANT]
> Запускайте скрипты только на странице `https://studio.rutube.ru/videos`, будучи авторизованным именно в том канале, контент которого требуется удалить.

---

## Возможности

- получает полный список материалов со всех страниц;
- показывает найденные ролики в виде таблицы до удаления;
- требует точную контрольную фразу;
- удаляет материалы последовательно, с паузой между запросами;
- повторяет временно неудачные запросы;
- продолжает работу при ошибке отдельного ролика;
- сохраняет список ошибок в глобальную переменную браузера;
- не требует вручную копировать cookies, CSRF-токен или другие данные авторизации.

---

## Как запустить скрипт

1. Откройте [RUTUBE Studio](https://studio.rutube.ru/videos).
2. Убедитесь, что выбран нужный канал.
3. Нажмите `F12`.
4. Откройте вкладку **Консоль** или **Console**.
5. Полностью скопируйте нужный скрипт из одного из разделов ниже.
6. Вставьте его в консоль и нажмите `Enter`.
7. Проверьте таблицу найденных материалов.
8. Введите точную контрольную фразу, которую покажет скрипт.

> [!TIP]
> Если браузер запрещает вставку в консоль, следуйте его встроенной инструкции. Не вставляйте код, происхождение и назначение которого вам неизвестны.

---

# 1. Удаление обычных видео

Этот вариант получает материалы через фильтр:

```text
origin_type_exclude=rshorts,rst
```

Следовательно, он **не удаляет Shorts с типом `rshorts`** и материалы типа `rst`. Все остальные найденные неудалённые видео будут показаны в таблице, после чего скрипт запросит подтверждение.

Пример контрольной фразы:

```text
УДАЛИТЬ 316
```

Число формируется автоматически и зависит от фактического количества найденных видео.

## ⬇️ НАЧАЛО СКРИПТА: УДАЛЕНИЕ ОБЫЧНЫХ ВИДЕО

```javascript
(async () => {
  'use strict';

  if (location.hostname !== 'studio.rutube.ru') {
    throw new Error(
      'Откройте https://studio.rutube.ru/videos и запустите скрипт там.'
    );
  }

  const SETTINGS = {
    // Пауза между удалениями, чтобы не перегружать API.
    delayMs: 700,

    // Количество повторных попыток при временной ошибке сервера.
    maxRetries: 4,

    // Размер страницы из вашего запроса.
    listLimit: 30,

    // Повторяет вашу текущую выборку.
    // Типы rshorts и rst удаляться не будут.
    originTypeExclude: 'rshorts,rst',
  };

  const sleep = (ms) =>
    new Promise((resolve) => setTimeout(resolve, ms));

  function getCookie(name) {
    const prefix = `${encodeURIComponent(name)}=`;

    const item = document.cookie
      .split('; ')
      .find((part) => part.startsWith(prefix));

    return item
      ? decodeURIComponent(item.slice(prefix.length))
      : null;
  }

  const csrfToken = getCookie('csrftoken');

  if (!csrfToken) {
    throw new Error(
      'Не найден csrftoken. Обновите страницу, войдите в аккаунт и повторите.'
    );
  }

  async function requestWithRetry(url, options = {}) {
    let lastError;

    for (
      let attempt = 1;
      attempt <= SETTINGS.maxRetries;
      attempt++
    ) {
      try {
        const response = await fetch(url, {
          credentials: 'include',
          cache: 'no-store',
          ...options,

          headers: {
            Accept: '*/*',
            'x-csrftoken': csrfToken,
            ...(options.headers || {}),
          },
        });

        if (response.ok) {
          return response;
        }

        const body = await response
          .text()
          .catch(() => '');

        const error = new Error(
          `HTTP ${response.status} ${response.statusText}` +
          (body ? `: ${body.slice(0, 300)}` : '')
        );

        const retryableStatuses = [
          429,
          500,
          502,
          503,
          504,
        ];

        if (!retryableStatuses.includes(response.status)) {
          throw error;
        }

        lastError = error;
      } catch (error) {
        lastError = error;
      }

      if (attempt < SETTINGS.maxRetries) {
        const waitMs =
          SETTINGS.delayMs * (2 ** (attempt - 1));

        console.warn(
          `Повтор запроса ${attempt}/${SETTINGS.maxRetries} ` +
          `через ${waitMs} мс`,
          lastError
        );

        await sleep(waitMs);
      }
    }

    throw lastError;
  }

  async function loadAllVideos() {
    const videosById = new Map();

    let page = 1;
    let totalPages = 1;

    do {
      const url = new URL(
        '/api/v2/video/person/',
        location.origin
      );

      url.search = new URLSearchParams({
        origin_type_exclude:
          SETTINGS.originTypeExclude,

        ordering: '-calculated_date',
        show_moderation: '1',
        limit: String(SETTINGS.listLimit),
        page: String(page),
      }).toString();

      const response = await requestWithRetry(url, {
        method: 'GET',
        headers: {
          Accept: 'application/json',
        },
      });

      const data = await response.json();

      totalPages = Number(
        data.num_pages || page
      );

      for (const video of data.results || []) {
        if (video?.id && !video.is_deleted) {
          videosById.set(video.id, video);
        }
      }

      console.log(
        `Получена страница ${page}/${totalPages}. ` +
        `Собрано роликов: ${videosById.size}`
      );

      page += 1;
    } while (page <= totalPages);

    return [...videosById.values()];
  }

  async function deleteVideo(video) {
    const url = new URL(
      `/api/v2/video/${encodeURIComponent(video.id)}/`,
      location.origin
    );

    url.searchParams.set('client', 'vulp');

    const response = await requestWithRetry(url, {
      method: 'DELETE',
    });

    // В вашем запросе успешное удаление возвращает 204.
    if (
      response.status !== 204 &&
      response.status !== 200
    ) {
      throw new Error(
        `Неожиданный статус удаления: ${response.status}`
      );
    }
  }

  console.log(
    'Получаю полный список роликов. ' +
    'На этом этапе ничего не удаляется...'
  );

  const videos = await loadAllVideos();

  if (videos.length === 0) {
    console.log(
      'Подходящих роликов не найдено. Ничего не удалено.'
    );

    return;
  }

  console.table(
    videos.map((video, index) => ({
      '№': index + 1,
      id: video.id,
      title: video.title,
      publication_ts: video.publication_ts,
      origin_type: video.origin_type,
    }))
  );

  // Сохраняем список в глобальную переменную,
  // чтобы его можно было посмотреть повторно.
  window.rutubeVideosBeforeDeletion = videos;

  const confirmationText =
    `УДАЛИТЬ ${videos.length}`;

  const confirmation = prompt(
    `Найдено роликов: ${videos.length}\n\n` +
    `Удаление необратимо.\n\n` +
    `Для продолжения введите точно:\n` +
    confirmationText
  );

  if (confirmation !== confirmationText) {
    console.warn(
      'Удаление отменено. Ничего не удалено.'
    );

    return;
  }

  let deleted = 0;
  const failed = [];

  for (
    let index = 0;
    index < videos.length;
    index++
  ) {
    const video = videos[index];

    try {
      await deleteVideo(video);

      deleted += 1;

      console.log(
        `Удалено ${deleted}/${videos.length}: ` +
        `${video.title} [${video.id}]`
      );
    } catch (error) {
      failed.push({
        id: video.id,
        title: video.title,
        error: String(error),
      });

      console.error(
        `Ошибка ${index + 1}/${videos.length}: ` +
        `${video.title} [${video.id}]`,
        error
      );
    }

    if (index < videos.length - 1) {
      await sleep(SETTINGS.delayMs);
    }
  }

  window.rutubeDeleteFailures = failed;

  console.log(
    `Готово. Удалено: ${deleted}. ` +
    `Ошибок: ${failed.length}.`
  );

  if (failed.length > 0) {
    console.table(failed);

    console.log(
      'Список ошибок сохранён в ' +
      'window.rutubeDeleteFailures'
    );
  }
})();
```

## ⬆️ КОНЕЦ СКРИПТА: УДАЛЕНИЕ ОБЫЧНЫХ ВИДЕО

После завершения список ошибок, если они возникли, доступен в консоли через:

```javascript
window.rutubeDeleteFailures
```

Список роликов, собранный перед удалением:

```javascript
window.rutubeVideosBeforeDeletion
```

---

# 2. Удаление всех Shorts

Этот вариант загружает материалы канала и оставляет только записи со значением:

```text
origin_type = rshorts
```

Перед удалением найденные Shorts выводятся в отдельную таблицу.

Пример контрольной фразы:

```text
УДАЛИТЬ ШОРТСЫ 125
```

Число формируется автоматически по фактическому количеству найденных Shorts.

## ⬇️ НАЧАЛО СКРИПТА: УДАЛЕНИЕ SHORTS

```javascript
(async () => {
  'use strict';

  const CONFIG = {
    delayMs: 700,
    maxRetries: 4,
    limit: 30,
    shortsOriginType: 'rshorts',
  };

  if (location.hostname !== 'studio.rutube.ru') {
    throw new Error(
      'Откройте https://studio.rutube.ru/videos и запустите скрипт там.'
    );
  }

  const sleep = (ms) =>
    new Promise((resolve) => setTimeout(resolve, ms));

  function getCookie(name) {
    const prefix = `${encodeURIComponent(name)}=`;

    const item = document.cookie
      .split('; ')
      .find((part) => part.startsWith(prefix));

    return item
      ? decodeURIComponent(item.slice(prefix.length))
      : null;
  }

  const csrfToken = getCookie('csrftoken');

  if (!csrfToken) {
    throw new Error(
      'Не найден csrftoken. Обновите страницу, войдите в аккаунт и повторите.'
    );
  }

  async function requestWithRetry(url, options = {}) {
    let lastError;

    for (
      let attempt = 1;
      attempt <= CONFIG.maxRetries;
      attempt++
    ) {
      try {
        const response = await fetch(url, {
          credentials: 'include',
          cache: 'no-store',
          ...options,

          headers: {
            Accept: '*/*',
            'x-csrftoken': csrfToken,
            ...(options.headers || {}),
          },
        });

        if (response.ok) {
          return response;
        }

        const responseText = await response
          .text()
          .catch(() => '');

        const error = new Error(
          `HTTP ${response.status} ${response.statusText}` +
          (responseText
            ? `: ${responseText.slice(0, 300)}`
            : '')
        );

        const retryableStatuses = [
          429,
          500,
          502,
          503,
          504,
        ];

        if (!retryableStatuses.includes(response.status)) {
          throw error;
        }

        lastError = error;
      } catch (error) {
        lastError = error;
      }

      if (attempt < CONFIG.maxRetries) {
        const waitMs =
          CONFIG.delayMs * (2 ** (attempt - 1));

        console.warn(
          `Повтор запроса ${attempt}/${CONFIG.maxRetries} ` +
          `через ${waitMs} мс`,
          lastError
        );

        await sleep(waitMs);
      }
    }

    throw lastError;
  }

  async function loadAllShorts() {
    const shortsById = new Map();

    let page = 1;
    let totalPages = 1;
    let totalItemsChecked = 0;

    do {
      const url = new URL(
        '/api/v2/video/person/',
        location.origin
      );

      /*
       * Не используем origin_type_exclude:
       * предыдущий запрос исключал rshorts.
       *
       * Получаем весь список и безопасно
       * фильтруем результаты на стороне браузера.
       */
      url.search = new URLSearchParams({
        ordering: '-calculated_date',
        show_moderation: '1',
        limit: String(CONFIG.limit),
        page: String(page),
      }).toString();

      const response = await requestWithRetry(url, {
        method: 'GET',

        headers: {
          Accept: 'application/json',
        },
      });

      const data = await response.json();
      const results = Array.isArray(data.results)
        ? data.results
        : [];

      totalItemsChecked += results.length;

      totalPages = Number(
        data.num_pages || data.total_pages || page
      );

      for (const video of results) {
        const isShorts =
          video?.origin_type ===
          CONFIG.shortsOriginType;

        if (
          video?.id &&
          isShorts &&
          !video.is_deleted
        ) {
          shortsById.set(video.id, video);
        }
      }

      console.log(
        `Проверена страница ${page}/${totalPages}. ` +
        `Проверено записей: ${totalItemsChecked}. ` +
        `Найдено шортсов: ${shortsById.size}.`
      );

      page += 1;
    } while (page <= totalPages);

    return [...shortsById.values()];
  }

  async function deleteShort(video) {
    const url = new URL(
      `/api/v2/video/${encodeURIComponent(video.id)}/`,
      location.origin
    );

    url.searchParams.set('client', 'vulp');

    const response = await requestWithRetry(url, {
      method: 'DELETE',
    });

    if (
      response.status !== 204 &&
      response.status !== 200
    ) {
      throw new Error(
        `Неожиданный статус удаления: ${response.status}`
      );
    }
  }

  console.log(
    'Получаю список всех материалов канала. ' +
    'На этом этапе ничего не удаляется...'
  );

  const shorts = await loadAllShorts();

  if (shorts.length === 0) {
    console.warn(
      'Шортсы с типом origin_type="rshorts" не найдены. ' +
      'Ничего не удалено.'
    );

    return;
  }

  console.table(
    shorts.map((video, index) => ({
      '№': index + 1,
      id: video.id,
      название: video.title,
      опубликовано: video.publication_ts,
      тип: video.origin_type,
      удалено: video.is_deleted,
    }))
  );

  window.rutubeShortsBeforeDeletion = shorts;

  console.log(
    'Список сохранён в переменной: ' +
    'window.rutubeShortsBeforeDeletion'
  );

  const confirmationText =
    `УДАЛИТЬ ШОРТСЫ ${shorts.length}`;

  const confirmation = prompt(
    `Найдено шортсов: ${shorts.length}\n\n` +
    `В таблице консоли должны быть только записи ` +
    `с типом rshorts.\n\n` +
    `Удаление необратимо.\n\n` +
    `Для продолжения введите точно:\n` +
    confirmationText
  );

  if (confirmation !== confirmationText) {
    console.warn(
      'Удаление отменено. Ничего не удалено.'
    );

    return;
  }

  /*
   * Для принудительной остановки во время работы
   * введите в консоли:
   *
   * window.stopRutubeShortsDeletion = true
   */
  window.stopRutubeShortsDeletion = false;

  let deleted = 0;
  const failed = [];

  for (
    let index = 0;
    index < shorts.length;
    index++
  ) {
    if (window.stopRutubeShortsDeletion === true) {
      console.warn(
        `Удаление остановлено пользователем. ` +
        `Удалено: ${deleted}/${shorts.length}.`
      );

      break;
    }

    const video = shorts[index];

    try {
      await deleteShort(video);

      deleted += 1;

      console.log(
        `Удалено ${deleted}/${shorts.length}: ` +
        `${video.title} [${video.id}]`
      );
    } catch (error) {
      failed.push({
        id: video.id,
        title: video.title,
        error: String(error),
      });

      console.error(
        `Ошибка при удалении ${index + 1}/${shorts.length}: ` +
        `${video.title} [${video.id}]`,
        error
      );
    }

    if (index < shorts.length - 1) {
      await sleep(CONFIG.delayMs);
    }
  }

  window.rutubeShortsDeleteFailures = failed;

  console.log(
    `Готово. Удалено шортсов: ${deleted}. ` +
    `Ошибок: ${failed.length}.`
  );

  if (failed.length > 0) {
    console.table(failed);

    console.log(
      'Список ошибок сохранён в переменной: ' +
      'window.rutubeShortsDeleteFailures'
    );
  }
})();
```

## ⬆️ КОНЕЦ СКРИПТА: УДАЛЕНИЕ SHORTS

Для аварийной остановки выполняющегося удаления Shorts введите в консоли:

```javascript
window.stopRutubeShortsDeletion = true
```

Уже удалённые материалы восстановлены не будут, но новые запросы на удаление прекратятся.

Список ошибок:

```javascript
window.rutubeShortsDeleteFailures
```

Список Shorts, собранный перед удалением:

```javascript
window.rutubeShortsBeforeDeletion
```

---

## Возможные статусы и ошибки

| Ситуация | Что означает |
|---|---|
| `204 No Content` | Материал успешно удалён |
| `401` или `403` | Сессия истекла, нет доступа или не найден CSRF-токен |
| `429` | Слишком много запросов; скрипт попробует повторить запрос |
| `500`, `502`, `503`, `504` | Временная ошибка сервера; предусмотрены повторные попытки |
| Ролики не найдены | Выбранный тип материалов отсутствует либо API изменился |

---

## Настройки

В начале каждого скрипта можно изменить паузу между удалениями:

```javascript
delayMs: 700
```

Значение указывается в миллисекундах. Уменьшать его слишком сильно не рекомендуется: сервер может начать возвращать статус `429`.

Количество повторных попыток:

```javascript
maxRetries: 4
```

Размер страницы API:

```javascript
limit: 30
```

---

## Безопасность

- Не публикуйте cookies, `refreshToken`, `csrftoken` и другие данные своей сессии.
- Не добавляйте токены авторизации прямо в исходный код.
- После случайной публикации данных сессии завершите активные сеансы или смените пароль.
- Перед массовым удалением проверьте, что открыта студия нужного канала.
- Не закрывайте вкладку браузера до завершения работы скрипта.

---

## Отказ от ответственности

Скрипты используют внутренние веб-запросы RUTUBE Studio. Интерфейс API может измениться без предупреждения. Автор репозитория не несёт ответственности за случайное удаление контента, потерю данных или ограничения аккаунта, возникшие в результате использования этих скриптов.

Используйте их только для управления собственным каналом и на свой страх и риск.
