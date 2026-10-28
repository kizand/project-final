## [REST API](http://localhost:8080/doc)

## Концепция:

- Spring Modulith
    - [Spring Modulith: достигли ли мы зрелости модульности](https://habr.com/ru/post/701984/)
    - [Introducing Spring Modulith](https://spring.io/blog/2022/10/21/introducing-spring-modulith)
    - [Spring Modulith - Reference documentation](https://docs.spring.io/spring-modulith/docs/current-SNAPSHOT/reference/html/)

```
  url: jdbc:postgresql://localhost:5432/jira
  username: jira
  password: JiraRush
```

- Есть 2 общие таблицы, на которых не fk
    - _Reference_ - справочник. Связь делаем по _code_ (по id нельзя, тк id привязано к окружению-конкретной базе)
    - _UserBelong_ - привязка юзеров с типом (owner, lead, ...) к объекту (таска, проект, спринт, ...). FK вручную будем
      проверять

## Аналоги

- https://java-source.net/open-source/issue-trackers

## Тестирование

- https://habr.com/ru/articles/259055/

### Список выполненных задач:
1. **Разобраться со структурой проекта (onboarding).**
2. **Удалить социальные сети: VK, Yandex.**
    * Удалены блоки в файле `application.yaml` в секциях `registration` и `provider`.
    * Удалены блоки в файлах `register.html` и `login.html`.
    * Удалены классы `VkOAuth2UserDataHandler` и `YandexOAuth2UserDataHandler` в пакете `com.javarush.jira.login.internal.sociallogin.handler`.
3. **Вынести чувствительную информацию в отдельный проперти-файл.**
    * В корне проекта создан файл `application-secrets.yml`, куда были перенесены:
        * Логин и пароль базы данных
        * Идентификаторы для OAuth регистрации/авторизации
        * Логин и пароль почты
    * *Примечание:* Так как по заданию *"Значения этих проперти должны считываться при старте сервера из переменных окружения машины"*, дополнительно эти же переменные были добавлены через `Run/Debug Configurations` -> `Environment variables`.
4. **Переделать тесты так, чтобы во время тестов использовалась In-Memory БД (H2), а не PostgreSQL.**
    * Добавлена зависимость `com.h2database:h2` со scope `test`, чтобы библиотека использовалась исключительно в тестовом окружении.
    * В пакете `config` создан конфигурационный класс `DbConfig`, определяющий `DataSource` с бинами под профили `prod` и `test`.
    * В директорию `test/resources` добавлен файл `application-test.yaml`.
    * Внесены изменения в файлы `data.sql` и `changelog.sql` для корректной работы синтаксиса H2.
5. **Написать тесты для всех публичных методов контроллера `ProfileRestController`.**
    * **Для метода `get()`:**
        * `get_Authenticated_ReturnsProfile()` — проверяет успешное получение профиля авторизованным пользователем.
        * `get_Unauthenticated_ReturnsUnauthorized()` — проверяет защиту эндпоинта фильтрами Spring Security (анонимный запрос блокируется до вызова бизнес-логики).
    * **Для метода `update()`:**
        * `update_Authenticated_UpdatesProfile()` — проверяет успешное обновление данных авторизованного пользователя.
        * `update_InvalidData_ReturnsBadRequestOrUnprocessable()` — проверяет работу Spring Bean Validation (`@Valid`).
        * `update_Unauthenticated_ReturnsUnauthorized()` — проверяет блокировку попытки изменения данных со стороны неавторизованного пользователя.
6. **Сделать рефакторинг метода `com.javarush.jira.bugtracking.attachment.FileUtil#upload`.**
    * Реализован переход на **NIO.2** с оптимизацией памяти: вместо вычитывания всех байт файла в память теперь используется потоковое копирование напрямую в файловую систему.
    * Сохранена трассировка ошибок: связка `IOException` с `IllegalRequestDataException` реализована через `initCause()`, чтобы при сбоях в логах оставался полный стек-трейс.
7. **Добавить новый функционал: добавление тегов к задаче (REST API + реализация на сервисе).**
    * Обнаружено, что в сущности `Task` уже была заложена связь с таблицей `task_tag` через аннотацию `@ElementCollection`.
    * В базовый класс `TaskTo` добавлено поле `private Set<String> tags;` (с геттером и сеттером).
    * В `TaskService` написан метод `addTags`, который:
        * Ищет задачу в БД или выбрасывает проектное исключение `NotFoundException`.
        * Безопасно подменяет дефолтный неизменяемый `Set.of()` на `HashSet`, предотвращая ошибки `UnsupportedOperationException`.
        * Добавляет новые теги и сохраняет изменения в базу.
    * В `TaskController` добавлен `POST`-эндпоинт `/{taskId}/tags`, принимающий массив строк в формате JSON.
8. **Добавить подсчет времени, сколько задача находилась в работе и тестировании.**
    * В конец скрипта инициализации базы данных `changelog.sql` добавлены 3 обязательные записи в таблицу `ACTIVITY` для симуляции жизненного цикла задачи №1:
        * Перевод в разработку (`in_progress`) — `2026-10-27 10:00:00`
        * Окончание разработки (`ready_for_review`) — `2026-10-27 14:30:15`
        * Завершение тестирования (`done`) — `2026-10-27 17:45:30`
    * В интерфейс `ActivityRepository` добавлен метод для эффективного поиска последних записей по каждому статусу:
        * `Optional<Activity> findFirstByTaskIdAndStatusCodeOrderByUpdatedDesc(long taskId, String statusCode);`
        * Запрос застрахован от ошибок ленивой инициализации (`LazyInitializationException`) с помощью аннотации `@EntityGraph(attributePaths = {"author"})`.
    * На уровне `TaskService` написаны два метода, принимающих объект `Task` и возвращающих время в текстовом формате `ЧЧ:ММ:СС`:
        * `getDevelopmentTime(Task task)` — вычисляет разницу между статусами `ready_for_review` и `in_progress`.
        * `getTestingTime(Task task)` — вычисляет разницу между статусами `done` и `ready_for_review`.
    * Для вычисления интервалов использован класс `java.time.Duration`. Предусмотрена защита от отрицательных значений (на случай, если задачу возвращали на доработку). Метод получения задачи `get(long id)` защищен аннотацией `@Transactional(readOnly = true)`.
    * В класс полной информации о задаче `TaskToFull` добавлены новые вычисляемые поля `developmentTime` и `testingTime`. Наполнение полей вынесено в метод `get` сервиса `TaskService` сразу после маппинга сущности.
    * Работа методов проверена через Postman. На основе внесенных тестовых данных получен корректный JSON-ответ для задачи №1:
        * Время в работе (`developmentTime`): `04:30:15` (ровно 4 часа 30 минут 15 секунд).
        * Время на тестировании (`testingTime`): `03:15:15` (ровно 3 часа 15 минут 15 секунд).
9. **Написать Dockerfile для основного сервера.**