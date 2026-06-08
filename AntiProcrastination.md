Отличный выбор 😈 **AntiProcrastination Boss** можно сделать как прикольный проект снаружи, но под капотом — прям серьёзный Java/Spring backend.

Идея: пользователь ставит задачи, выполняет их и “бьёт босса”. Если пользователь прокрастинирует, босс восстанавливает здоровье, задачи просрочиваются, серия выполнения сбрасывается, рейтинг падает. Получается геймифицированный таск-трекер.

# AntiProcrastination Boss API

## 1. Общая идея backend

Backend должен отвечать за:

```text
регистрацию и авторизацию пользователей;
создание задач и целей;
расчёт прогресса пользователя;
систему босса;
ежедневные фоновые проверки;
начисление опыта и достижений;
штрафы за просрочки;
рейтинг пользователей;
уведомления;
хранение истории действий.
```

То есть это не просто todo-list, а система с бизнес-логикой, событиями, фоновыми задачами, ролями, статистикой и игровыми механиками.

---

# 2. Стек проекта

Я бы делал так:

```text
Java 21
Spring Boot
Spring Web
Spring Security
JWT
PostgreSQL
Spring Data JPA
Flyway
Redis
RabbitMQ
Spring Scheduler
Swagger / OpenAPI
Docker Compose
JUnit
Mockito
Testcontainers
```

Что для чего:

```text
Spring Boot — основа backend-приложения.
PostgreSQL — основная база данных.
JWT — авторизация пользователей.
Spring Security — защита эндпоинтов.
Flyway — миграции базы данных.
Redis — кэш рейтинга, активных боссов, streak.
RabbitMQ — события: задача выполнена, босс получил урон, задача просрочена.
Spring Scheduler — ежедневная проверка просроченных задач.
Swagger — документация API.
Docker Compose — запуск приложения, БД, Redis и RabbitMQ.
Тесты — проверка бизнес-логики.
```

---

# 3. Основные сущности

## User

Пользователь системы.

Поля:

```text
id
email
passwordHash
username
level
experience
createdAt
```

## Goal

Крупная цель пользователя.

Например:

```text
Закрыть диплом
Выучить Java
Начать ходить в зал
Прочитать книгу
```

Поля:

```text
id
userId
title
description
status
createdAt
deadline
```

## Task

Конкретная задача.

Например:

```text
Прочитать 20 страниц
Сделать 1 лекцию по Spring
Написать 30 минут код
```

Поля:

```text
id
goalId
userId
title
description
difficulty
status
deadline
createdAt
completedAt
```

Статусы:

```text
TODO
IN_PROGRESS
DONE
FAILED
EXPIRED
```

Сложность задачи:

```text
EASY
MEDIUM
HARD
BOSS_DAMAGE
```

## Boss

Босс, которого пользователь побеждает выполнением задач.

Поля:

```text
id
name
description
maxHp
currentHp
level
status
createdAt
expiresAt
```

Статусы:

```text
ACTIVE
DEFEATED
ESCAPED
```

Примеры боссов:

```text
Лорд Прокрастинации
Генерал Дедлайн
Демон Скроллинга
Повелитель “Начну завтра”
Барон Бесконечного YouTube
```

## BossFight

Связь пользователя и текущего босса.

Поля:

```text
id
userId
bossId
bossCurrentHp
status
startedAt
finishedAt
```

Статусы:

```text
ACTIVE
WON
LOST
```

## Achievement

Достижение.

Примеры:

```text
Первая кровь — выполнил первую задачу
Без шансов — победил босса без просрочек
7 дней силы — выполнил задачи 7 дней подряд
Убийца дедлайнов — закрыл 50 задач
```

Поля:

```text
id
code
title
description
conditionType
```

## UserAchievement

Полученные достижения пользователя.

```text
id
userId
achievementId
receivedAt
```

## Streak

Серия активности пользователя.

```text
id
userId
currentDays
maxDays
lastActiveDate
```

## Notification

Уведомления.

```text
id
userId
title
message
type
isRead
createdAt
```

Типы:

```text
TASK_DEADLINE_SOON
TASK_EXPIRED
BOSS_DEFEATED
BOSS_ESCAPED
ACHIEVEMENT_UNLOCKED
```

## AuditLog / EventLog

История действий.

```text
id
userId
eventType
payload
createdAt
```

---

# 4. Роли пользователей

Для начала можно сделать две роли:

```text
USER
ADMIN
```

`USER` может создавать цели, задачи, выполнять их, сражаться с боссом.

`ADMIN` может создавать шаблоны боссов, смотреть статистику, модерировать данные.

---

# 5. Основные модули backend

## Auth module

Отвечает за регистрацию, вход и JWT.

Эндпоинты:

```text
POST /api/auth/register
POST /api/auth/login
GET  /api/auth/me
```

Что изучишь:

```text
Spring Security
JWT
PasswordEncoder
защита эндпоинтов
работа с текущим пользователем
```

---

## Goals module

Цели пользователя.

Эндпоинты:

```text
POST   /api/goals
GET    /api/goals
GET    /api/goals/{goalId}
PATCH  /api/goals/{goalId}
DELETE /api/goals/{goalId}
```

Логика:

```text
Пользователь видит только свои цели.
Нельзя редактировать чужие цели.
Цель может быть активной, завершенной или отмененной.
```

---

## Tasks module

Задачи пользователя.

Эндпоинты:

```text
POST   /api/goals/{goalId}/tasks
GET    /api/tasks
GET    /api/tasks/{taskId}
PATCH  /api/tasks/{taskId}
PATCH  /api/tasks/{taskId}/start
PATCH  /api/tasks/{taskId}/complete
DELETE /api/tasks/{taskId}
```

Фильтры:

```text
GET /api/tasks?status=TODO
GET /api/tasks?deadlineFrom=2026-06-01&deadlineTo=2026-06-30
GET /api/tasks?difficulty=HARD
```

Логика:

```text
Пользователь может завершить только свою задачу.
Просроченную задачу нельзя завершить как обычную.
За выполнение задачи начисляется опыт.
За выполнение задачи наносится урон боссу.
Сила урона зависит от сложности задачи.
```

Пример урона:

```text
EASY   → 10 damage
MEDIUM → 25 damage
HARD   → 50 damage
```

---

## Boss module

Игровая часть проекта.

Эндпоинты:

```text
GET  /api/boss/current
POST /api/boss/start
GET  /api/boss/history
```

Логика:

```text
У пользователя может быть только один активный бой.
При выполнении задачи босс получает урон.
Если HP босса падает до 0, бой завершается победой.
Если пользователь долго не выполняет задачи, босс может восстановить HP.
Если срок боя истек, босс сбегает.
```

Пример:

```text
Пользователь выполнил HARD-задачу.
Система наносит боссу 50 урона.
Если HP босса стало 0 или меньше, пользователь побеждает.
Пользователь получает опыт и достижение.
```

---

## Progress module

Прогресс пользователя.

Эндпоинты:

```text
GET /api/progress/profile
GET /api/progress/statistics
GET /api/progress/streak
```

Что показывает:

```text
уровень пользователя;
опыт;
количество выполненных задач;
текущий streak;
максимальный streak;
побежденные боссы;
процент выполненных задач;
количество просроченных задач.
```

---

## Achievement module

Достижения.

Эндпоинты:

```text
GET /api/achievements
GET /api/achievements/my
```

Логика:

```text
После выполнения задачи проверяется, не получил ли пользователь новое достижение.
Если достижение получено, создается UserAchievement.
Также создается уведомление.
```

Примеры условий:

```text
Выполнил первую задачу.
Выполнил 10 задач.
Победил первого босса.
Закрыл 5 задач за один день.
Имеет streak 7 дней.
```

---

## Rating module

Рейтинг пользователей.

Эндпоинты:

```text
GET /api/rating/global
GET /api/rating/weekly
GET /api/rating/me
```

Можно хранить рейтинг в Redis:

```text
userId → score
```

Очки рейтинга:

```text
выполненная задача +10/+25/+50;
победа над боссом +100;
просрок задачи -15;
побег босса -50.
```

---

## Notification module

Уведомления.

Эндпоинты:

```text
GET   /api/notifications
PATCH /api/notifications/{id}/read
PATCH /api/notifications/read-all
```

Уведомления создаются при событиях:

```text
задача скоро просрочится;
задача просрочена;
босс побежден;
получено достижение;
босс восстановил здоровье.
```

---

# 6. Событийная архитектура

Чтобы проект был реально сильным, сделай события через RabbitMQ.

События:

```text
TASK_COMPLETED
TASK_EXPIRED
BOSS_DAMAGED
BOSS_DEFEATED
ACHIEVEMENT_UNLOCKED
STREAK_UPDATED
```

Пример логики:

```text
Пользователь выполнил задачу.
TaskService сохраняет задачу как DONE.
TaskService отправляет событие TASK_COMPLETED.
BossService слушает событие и наносит урон боссу.
ProgressService слушает событие и начисляет опыт.
AchievementService слушает событие и проверяет достижения.
NotificationService слушает событие и создает уведомление.
```

Это очень похоже на реальный backend, где разные части системы реагируют на события.

---

# 7. Фоновые задачи

Через `Spring Scheduler` можно сделать ежедневные проверки.

## Проверка просроченных задач

Каждый день ночью:

```text
найти задачи со статусом TODO/IN_PROGRESS;
проверить deadline;
если deadline прошел — поставить EXPIRED;
создать событие TASK_EXPIRED;
создать уведомление.
```

## Восстановление HP босса

Если пользователь не выполнял задачи за день:

```text
босс восстанавливает 10-20 HP;
streak сбрасывается;
создается уведомление.
```

## Завершение просроченных боев

Если бой с боссом длится больше срока:

```text
статус BossFight = LOST;
босс считается ESCAPED;
пользователь получает штраф рейтинга.
```

---

# 8. PostgreSQL схема примерно

```text
users
roles
user_roles

goals
tasks

bosses
boss_fights

achievements
user_achievements

streaks
notifications
event_logs
```

Связи:

```text
User 1 — N Goal
Goal 1 — N Task
User 1 — N Task
User 1 — N BossFight
Boss 1 — N BossFight
User N — M Achievement
User 1 — 1 Streak
User 1 — N Notification
```

---

# 9. Redis где использовать

Redis можно добавить не сразу, но он сделает проект сильнее.

Применение:

```text
кэш глобального рейтинга;
кэш недельного рейтинга;
хранение активных boss fight;
ограничение частоты запросов;
быстрый подсчет позиции пользователя в рейтинге.
```

Например:

```text
leaderboard:global
leaderboard:weekly
```

---

# 10. Пример API целиком

```text
POST /api/auth/register
POST /api/auth/login
GET  /api/auth/me

POST   /api/goals
GET    /api/goals
GET    /api/goals/{id}
PATCH  /api/goals/{id}
DELETE /api/goals/{id}

POST   /api/goals/{goalId}/tasks
GET    /api/tasks
GET    /api/tasks/{id}
PATCH  /api/tasks/{id}
PATCH  /api/tasks/{id}/start
PATCH  /api/tasks/{id}/complete
DELETE /api/tasks/{id}

GET  /api/boss/current
POST /api/boss/start
GET  /api/boss/history

GET /api/progress/profile
GET /api/progress/statistics
GET /api/progress/streak

GET /api/achievements
GET /api/achievements/my

GET /api/rating/global
GET /api/rating/weekly
GET /api/rating/me

GET   /api/notifications
PATCH /api/notifications/{id}/read
PATCH /api/notifications/read-all
```

---

# 11. Что делать по этапам

## MVP 1 — база

```text
Spring Boot проект
PostgreSQL
Flyway
регистрация
логин
JWT
создание целей
создание задач
завершение задач
Swagger
Docker Compose
```

## MVP 2 — игровой слой

```text
босс
бой с боссом
урон за выполненные задачи
победа над боссом
опыт пользователя
уровни
streak
```

## MVP 3 — события

```text
RabbitMQ
событие TASK_COMPLETED
событие BOSS_DEFEATED
событие TASK_EXPIRED
обработчики событий
уведомления
event log
```

## MVP 4 — продвинутый backend

```text
Redis leaderboard
Spring Scheduler
глобальный рейтинг
еженедельный рейтинг
Testcontainers
unit-тесты
integration-тесты
README
```

---

# 12. Почему этот проект реально полезный

Он научит тебя:

```text
строить REST API;
делать авторизацию через JWT;
работать с PostgreSQL и JPA;
проектировать связи между сущностями;
писать бизнес-логику;
делать события через RabbitMQ;
использовать Redis;
делать фоновые задачи;
писать тесты;
делать Docker Compose;
оформлять Swagger;
думать как backend-разработчик.
```

---

# 13. Как это красиво описать в GitHub

Название:

```text
AntiProcrastination Boss API
```

Описание:

```text
Gamified productivity backend where users complete tasks, damage procrastination bosses, earn experience, unlock achievements and compete in leaderboards.
```

По-русски:

```text
Backend-сервис геймифицированного трекера задач, в котором пользователь выполняет задачи, наносит урон боссу прокрастинации, получает опыт, открывает достижения и участвует в рейтинге.
```

Мой совет: начинай с **MVP 1 + MVP 2**. RabbitMQ, Redis и Testcontainers добавишь позже, когда база уже будет работать. Так ты не утонешь в сложности сразу 🚀