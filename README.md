# RutubeUtility
Утилиты для удобного использования Рутуба

Скрипт для удаления всех видеороликов на Рутуб канале:

Важное предупреждение

Удаление необратимо: RUTUBE официально предупреждает, что восстановить ролик после удаления нельзя.

Как запустить
Откройте страницу:
https://studio.rutube.ru/videos
Убедитесь, что вошли именно в нужный канал.
Нажмите F12.
Откройте вкладку Консоль.
Вставьте весь скрипт ниже и нажмите Enter.
Сначала скрипт только получит список роликов и выведет таблицу.
Для начала удаления появится окно подтверждения. Нужно будет ввести, например:
УДАЛИТЬ 316

Число сформируется автоматически по фактическому количеству найденных видео.

Полностью скопируйте код отсюда:
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

И до сюда.

Что именно удалит этот вариант

Он удалит ролики, которые возвращает ваш запрос:

/api/v2/video/person/
?origin_type_exclude=rshorts,rst
&ordering=-calculated_date
&show_moderation=1
&limit=30

То есть типы rshorts и rst специально исключены. Остальные найденные ролики будут сначала собраны со всех страниц, а затем удалены по одному запросом:

DELETE /api/v2/video/{ID}/?client=vulp

Во время работы вкладку лучше не закрывать. Если отдельные удаления завершатся ошибками, скрипт продолжит работу, а в конце покажет список неудалённых роликов.


Для удаления шортсов используйте этот

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
