# VK Wall Cleaner

Скрипт для автоматического удаления постов со стены ВКонтакте. Работает через консоль браузера.

## Важные замечания
- Ограничения ВКонтакте: Если вы удаляете слишком много постов за короткое время, ВКонтакте может запросить капчу или временно заблокировать аккаунт.
- Ручное вмешательство: Если скрипт не сработает (например, из-за капчи), вам придется вручную подтверждать действия.
- Ответственность: **Используйте скрипт на свой страх и риск**.

## ⚠️ Внимание:
### Протестировано 3 января 2025 года
Скрипт протестирован и работает на **03.01.2025**. Если ВКонтакте изменит интерфейс, скрипт может потребовать обновления.
### Пауза для подтверждения капчи не предусмотрена
Если ВКонтакте запросит капчу, скрипт остановится, и вам потребуется вручную решить капчу для продолжения.

## Как использовать

1. Перейдите на свою страницу ВКонтакте и откройте стену.
2. Откройте консоль разработчика в браузере:
   - **Chrome**: `Ctrl + Shift + J` (Windows/Linux) или `Cmd + Option + J` (Mac).
   - **Firefox**: `Ctrl + Shift + K` (Windows/Linux) или `Cmd + Option + K` (Mac).
3. Скопируйте и вставьте следующий скрипт в консоль:

```javascript
// ================== НАСТРОЙКИ ==================
const POSTS_PORTION_SIZE = 50; // Количество постов для загрузки за один раз
const SCROLL_DELAY = 500; // Задержка между прокрутками (в миллисекундах)
const DELETE_DELAY = 100; // Задержка между действиями при удалении (в миллисекундах)
const MAX_ATTEMPTS = 20; // Максимальное количество попыток для загрузки новых постов
const MAX_POSTS_TO_DELETE = Infinity; // Максимальное количество постов для удаления (Infinity для удаления всех)
// ===============================================

// Функция для проверки загрузки страницы
async function waitForPageLoad() {
    console.info('Ожидаем загрузки страницы...');
    await new Promise(resolve => {
        const interval = setInterval(() => {
            const posts = document.querySelectorAll('._post');
            if (posts.length > 0) {
                clearInterval(interval);
                resolve();
            }
        }, 500); // Проверяем каждые 500 мс
    });
    console.info('Страница загружена.');
}

// Функция для проверки удаления первого поста
async function testDeleteFirstPost() {
    // Находим первый пост на стене
    const firstPost = document.querySelector('._post');

    if (!firstPost) {
        console.error('Посты не найдены. Проверьте, открыта ли стена.');
        return false;
    }

    // Находим кнопку меню поста (используем более гибкий поиск)
    const menuButton = firstPost.querySelector('.vkuiIconButton, [aria-label="Меню"]');
    if (!menuButton) {
        console.error('Кнопка меню поста не найдена. Возможно, интерфейс изменился.');
        console.error('Попробуйте обновить страницу и запустить скрипт снова.');
        return false;
    }

    menuButton.click(); // Открываем меню
    await new Promise(resolve => setTimeout(resolve, DELETE_DELAY)); // Пауза для загрузки меню

    // Используем XPath для поиска span с текстом "Удалить"
    const xpath = "//span[contains(text(), 'Удалить')]";
    const deleteButton = document.evaluate(
        xpath,
        document,
        null,
        XPathResult.FIRST_ORDERED_NODE_TYPE,
        null
    ).singleNodeValue;

    if (!deleteButton) {
        console.error('Кнопка "Удалить" не найдена. Возможно, интерфейс изменился.');
        console.error('Попробуйте обновить страницу и запустить скрипт снова.');
        return false;
    }

    deleteButton.click(); // Нажимаем "Удалить"
    await new Promise(resolve => setTimeout(resolve, DELETE_DELAY)); // Пауза для подтверждения

    console.info('Проверка прошла успешно: первый пост удален.');
    return true;
}

// Функция для загрузки порции постов
async function loadPostsPortion() {
    let previousPostCount = 0;
    let attempts = 0; // Счетчик попыток, чтобы избежать бесконечного цикла

    while (true) {
        // Прокручиваем страницу вниз на высоту экрана
        window.scrollBy(0, window.innerHeight);

        // Ждем указанное время для загрузки новых постов
        await new Promise(resolve => setTimeout(resolve, SCROLL_DELAY));

        // Находим все посты на стене
        const posts = document.querySelectorAll('._post');

        // Логируем прогресс загрузки с указанием номера попытки
        console.info(`Попытка ${attempts + 1}/${MAX_ATTEMPTS}: Загружено постов: ${posts.length}. Прогресс: ${((posts.length / POSTS_PORTION_SIZE) * 100).toFixed(2)}%`);

        // Если количество постов не изменилось, значит, новых постов больше нет
        if (posts.length === previousPostCount) {
            attempts++;
            if (attempts >= MAX_ATTEMPTS) { // Если попытки исчерпаны, завершаем
                console.info('Загрузка постов завершена.');
                break;
            }
        } else {
            attempts = 0; // Сбрасываем счетчик, если посты загрузились
        }

        // Если загружено достаточно постов, завершаем загрузку порции
        if (posts.length >= POSTS_PORTION_SIZE) {
            console.info(`Загружено ${posts.length} постов. Переходим к удалению.`);
            break;
        }

        previousPostCount = posts.length; // Обновляем счетчик
    }
}

// Функция для удаления порции постов
async function deletePostsPortion() {
    // Находим все посты на стене
    const posts = document.querySelectorAll('._post');

    if (posts.length === 0) {
        console.info('Посты не найдены.');
        return;
    }

    let postNumber = 1; // Номер текущего поста

    // Удаляем каждый пост
    for (const post of posts) {
        // Логируем прогресс удаления
        console.info(`Удаляем пост №${postNumber}. Осталось постов: ${posts.length - postNumber + 1}. Прогресс: ${((postNumber / posts.length) * 100).toFixed(2)}%`);

        // Находим кнопку меню поста (используем более гибкий поиск)
        const menuButton = post.querySelector('.vkuiIconButton, [aria-label="Меню"]');
        if (menuButton) {
            menuButton.click(); // Открываем меню
            await new Promise(resolve => setTimeout(resolve, DELETE_DELAY)); // Пауза для загрузки меню

            // Используем XPath для поиска span с текстом "Удалить"
            const xpath = "//span[contains(text(), 'Удалить')]";
            const deleteButton = document.evaluate(
                xpath,
                document,
                null,
                XPathResult.FIRST_ORDERED_NODE_TYPE,
                null
            ).singleNodeValue;

            if (deleteButton) {
                deleteButton.click(); // Нажимаем "Удалить"
                await new Promise(resolve => setTimeout(resolve, DELETE_DELAY)); // Пауза перед следующим
            }
        }

        postNumber++; // Увеличиваем номер поста
    }

    console.info('Порция постов удалена.');
}

// Основная функция
async function main() {
    console.info('Ожидаем загрузки страницы...');
    await waitForPageLoad(); // Ждем загрузки страницы

    console.info('Проверяем возможность удаления первого поста...');
    const isTestSuccessful = await testDeleteFirstPost();

    if (!isTestSuccessful) {
        console.error('Скрипт не может удалить посты. Возможно, интерфейс ВКонтакте изменился.');
        console.error('Попробуйте обновить страницу и запустить скрипт снова.');
        return;
    }

    let totalDeleted = 0; // Общее количество удаленных постов

    while (true) {
        console.info('Начинаем загрузку порции постов...');
        await loadPostsPortion(); // Загружаем порцию постов

        const posts = document.querySelectorAll('._post');
        if (posts.length === 0) {
            console.info('Все посты удалены.');
            break;
        }

        console.info('Начинаем удаление порции постов...');
        await deletePostsPortion(); // Удаляем порцию постов

        totalDeleted += posts.length;
        console.info(`Всего удалено постов: ${totalDeleted}`);

        // Если достигнут лимит удаления, завершаем
        if (totalDeleted >= MAX_POSTS_TO_DELETE) {
            console.info(`Достигнут лимит удаления (${MAX_POSTS_TO_DELETE} постов).`);
            break;
        }
    }
}

// Запуск основной функции
main();
```
4. Нажмите Enter и дождитесь завершения процесса.

## Вклад в проект
Приветствуются пул-реквесты (Pull Requests) для улучшения и усовершенствования скрипта. Если у вас есть идеи, как сделать скрипт лучше, или вы хотите добавить новые функции, следуйте этим шагам:

1. Сделайте форк (Fork) репозитория.
2. Создайте новую ветку (Branch) для ваших изменений.
3. Внесите изменения и сделайте коммит (Commit) с описанием.
4. Отправьте пул-реквест (Pull Request) в основную ветку этого репозитория.
