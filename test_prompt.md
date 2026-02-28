# Промпт для разработки системы обратной связи сотрудников

## Контекст проекта

Разработай полнофункциональный прототип веб-приложения **"Система сбора обратной связи о взаимодействии сотрудников"** для хакатона. Приложение позволяет сотрудникам оставлять структурированные отзывы о взаимодействии с коллегами в рамках рабочих задач, а руководителям — получать аналитику, отчёты и инфографику для принятия кадровых решений.

## Стек технологий

- **Backend:** Laravel 11 (PHP 8.2+)
- **Frontend:** Vue.js 3 (Composition API) + Vite + Inertia.js (или как SPA с API)
- **СУБД:** MySQL 8 / MariaDB 10.6+
- **CSS:** Tailwind CSS 3
- **Графики:** Chart.js (через vue-chartjs)
- **Экспорт:** Laravel Excel (maatwebsite/excel)
- **Аутентификация:** Laravel Breeze или Sanctum
- **Дополнительно:** Laravel Spatie Permission (роли и права)

---

## Архитектура и структура БД

### Таблицы базы данных

```sql
-- Пользователи (расширение стандартной users)
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('employee', 'manager', 'hr', 'admin') DEFAULT 'employee',
    department_id BIGINT UNSIGNED NULL,
    position VARCHAR(255) NULL,
    avatar VARCHAR(255) NULL,
    is_active BOOLEAN DEFAULT TRUE,
    remember_token VARCHAR(100) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);

-- Департаменты / команды
CREATE TABLE departments (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT NULL,
    manager_id BIGINT UNSIGNED NULL,
    parent_id BIGINT UNSIGNED NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (manager_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (parent_id) REFERENCES departments(id) ON DELETE SET NULL
);

-- Рабочие эпизоды / задачи (контекст взаимодействия)
CREATE TABLE work_episodes (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT NULL,
    external_task_id VARCHAR(255) NULL COMMENT 'ID задачи из внешней CRM (для будущей интеграции)',
    department_id BIGINT UNSIGNED NULL,
    created_by BIGINT UNSIGNED NOT NULL,
    started_at DATE NULL,
    ended_at DATE NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE SET NULL,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE CASCADE
);

-- Участники эпизода
CREATE TABLE episode_participants (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    episode_id BIGINT UNSIGNED NOT NULL,
    user_id BIGINT UNSIGNED NOT NULL,
    created_at TIMESTAMP NULL,
    FOREIGN KEY (episode_id) REFERENCES work_episodes(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_participant (episode_id, user_id)
);

-- Категории оценок (настраиваемые)
CREATE TABLE feedback_categories (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    type ENUM('positive', 'negative') NOT NULL,
    icon VARCHAR(100) NULL COMMENT 'Название иконки или эмодзи',
    description TEXT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    created_by BIGINT UNSIGNED NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL
);

-- Отзывы (основная таблица)
CREATE TABLE feedbacks (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    author_id BIGINT UNSIGNED NOT NULL COMMENT 'Кто оставил (скрыт в интерфейсе)',
    recipient_id BIGINT UNSIGNED NOT NULL COMMENT 'О ком отзыв',
    episode_id BIGINT UNSIGNED NOT NULL COMMENT 'Контекст — рабочий эпизод',
    sentiment ENUM('positive', 'negative') NOT NULL,
    score TINYINT NULL COMMENT 'Числовая оценка 1-5 (опционально)',
    comment TEXT NOT NULL COMMENT 'Обязательный комментарий',
    is_flagged BOOLEAN DEFAULT FALSE COMMENT 'Помечен модератором',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (author_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (recipient_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (episode_id) REFERENCES work_episodes(id) ON DELETE CASCADE,
    UNIQUE KEY unique_feedback (author_id, recipient_id, episode_id) COMMENT 'Один отзыв на связку автор-получатель-эпизод'
);

-- Связь отзыв-категории (many-to-many)
CREATE TABLE feedback_feedback_category (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    feedback_id BIGINT UNSIGNED NOT NULL,
    feedback_category_id BIGINT UNSIGNED NOT NULL,
    FOREIGN KEY (feedback_id) REFERENCES feedbacks(id) ON DELETE CASCADE,
    FOREIGN KEY (feedback_category_id) REFERENCES feedback_categories(id) ON DELETE CASCADE,
    UNIQUE KEY unique_fc (feedback_id, feedback_category_id)
);

-- Полугодовые срезы (анкеты)
CREATE TABLE semi_annual_surveys (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    status ENUM('draft', 'active', 'closed') DEFAULT 'draft',
    created_by BIGINT UNSIGNED NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE CASCADE
);

-- Ответы на полугодовой срез
CREATE TABLE survey_responses (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    survey_id BIGINT UNSIGNED NOT NULL,
    respondent_id BIGINT UNSIGNED NOT NULL,
    -- Социометрические вопросы (хранятся как JSON)
    frequent_collaborators JSON NULL COMMENT 'С кем чаще взаимодействовал [user_ids]',
    comfortable_collaborators JSON NULL COMMENT 'С кем комфортнее работать [user_ids]',
    help_seekers JSON NULL COMMENT 'К кому обращался за помощью [user_ids]',
    project_team JSON NULL COMMENT 'Кого выбрал бы на сложный проект [user_ids]',
    -- Самооценка состояния
    wellbeing_score TINYINT NULL COMMENT 'Самочувствие 1-5',
    workload_score TINYINT NULL COMMENT 'Загруженность 1-5',
    team_atmosphere_score TINYINT NULL COMMENT 'Атмосфера в команде 1-5',
    motivation_score TINYINT NULL COMMENT 'Мотивация 1-5',
    general_comment TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (survey_id) REFERENCES semi_annual_surveys(id) ON DELETE CASCADE,
    FOREIGN KEY (respondent_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_response (survey_id, respondent_id)
);

-- Лог активности (аудит)
CREATE TABLE activity_logs (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NULL,
    action VARCHAR(255) NOT NULL,
    subject_type VARCHAR(255) NULL,
    subject_id BIGINT UNSIGNED NULL,
    properties JSON NULL,
    created_at TIMESTAMP NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);
```

---

## Backend (Laravel)

### Модели и связи

Создай следующие модели с отношениями:

**User:**
- hasMany feedbacks (как автор: `author_id`)
- hasMany receivedFeedbacks (как получатель: `recipient_id`)
- belongsTo department
- belongsToMany workEpisodes (через `episode_participants`)
- hasMany surveyResponses

**Department:**
- hasMany users
- belongsTo manager (User)
- belongsTo parent (Department)
- hasMany children (Department)

**WorkEpisode:**
- belongsTo creator (User)
- belongsTo department
- belongsToMany participants (User через `episode_participants`)
- hasMany feedbacks

**FeedbackCategory:**
- belongsToMany feedbacks
- scopeActive, scopePositive, scopeNegative

**Feedback:**
- belongsTo author (User)
- belongsTo recipient (User)
- belongsTo episode (WorkEpisode)
- belongsToMany categories (FeedbackCategory)

**SemiAnnualSurvey:**
- hasMany responses (SurveyResponse)
- belongsTo creator (User)

**SurveyResponse:**
- belongsTo survey (SemiAnnualSurvey)
- belongsTo respondent (User)

### API маршруты (routes/api.php)

```php
// Аутентификация
Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout'])->middleware('auth:sanctum');
Route::get('/user', [AuthController::class, 'user'])->middleware('auth:sanctum');

Route::middleware('auth:sanctum')->group(function () {

    // === СОТРУДНИК ===

    // Рабочие эпизоды
    Route::get('/episodes', [EpisodeController::class, 'index']); // свои эпизоды
    Route::post('/episodes', [EpisodeController::class, 'store']);
    Route::get('/episodes/{episode}', [EpisodeController::class, 'show']);

    // Отзывы (создание)
    Route::post('/feedbacks', [FeedbackController::class, 'store']);
    Route::get('/feedbacks/check', [FeedbackController::class, 'checkExisting']); // проверка дубликата
    Route::get('/feedbacks/my', [FeedbackController::class, 'myFeedbacks']); // свои оставленные

    // Категории (только чтение для сотрудника)
    Route::get('/categories', [CategoryController::class, 'index']);

    // Коллеги (для выбора в форме отзыва)
    Route::get('/colleagues', [UserController::class, 'colleagues']);

    // Полугодовые опросы
    Route::get('/surveys/active', [SurveyController::class, 'active']);
    Route::post('/surveys/{survey}/respond', [SurveyController::class, 'respond']);

    // === РУКОВОДИТЕЛЬ / HR ===
    Route::middleware('role:manager,hr,admin')->prefix('manager')->group(function () {

        // Отчёты
        Route::get('/reports/summary', [ReportController::class, 'summary']);
        Route::get('/reports/employee/{user}', [ReportController::class, 'employeeDetail']);
        Route::get('/reports/department/{department}', [ReportController::class, 'departmentReport']);
        Route::get('/reports/dynamics', [ReportController::class, 'dynamics']);
        Route::get('/reports/top-categories', [ReportController::class, 'topCategories']);

        // Экспорт
        Route::get('/export/report', [ExportController::class, 'exportReport']);

        // Управление категориями
        Route::post('/categories', [CategoryController::class, 'store']);
        Route::put('/categories/{category}', [CategoryController::class, 'update']);
        Route::patch('/categories/{category}/toggle', [CategoryController::class, 'toggle']);

        // Полугодовые опросы (управление)
        Route::apiResource('/surveys', SurveyController::class);
        Route::get('/surveys/{survey}/results', [SurveyController::class, 'results']);
        Route::get('/surveys/{survey}/sociometry', [SurveyController::class, 'sociometry']);

        // Сотрудники и структура
        Route::get('/employees', [UserController::class, 'index']);
        Route::get('/departments', [DepartmentController::class, 'index']);

        // Просмотр отзывов (без раскрытия автора!)
        Route::get('/feedbacks', [FeedbackController::class, 'managerIndex']);
    });

    // === АДМИН ===
    Route::middleware('role:admin')->prefix('admin')->group(function () {
        Route::apiResource('/users', AdminUserController::class);
        Route::apiResource('/departments', AdminDepartmentController::class);
    });
});
```

### Ключевые контроллеры

**FeedbackController::store** — логика создания отзыва:
```
- Валидация: recipient_id, episode_id, sentiment, categories (1-3), comment (min 20 символов)
- Проверка: автор ≠ получатель
- Проверка: автор является участником эпизода
- Проверка уникальности: unique(author_id, recipient_id, episode_id)
- Проверка: категории соответствуют sentiment (позитивные к позитивному и наоборот)
- Лимит: не более N отзывов в день от одного пользователя (настраиваемый, по умолчанию 10)
- Сохранение + привязка категорий
- Лог активности
- НЕ возвращать данные о чужих отзывах
```

**ReportController::summary** — сводный отчёт:
```
- Параметры: date_from, date_to, department_id (optional), user_id (optional)
- Возвращает:
  - total_feedbacks, positive_count, negative_count, positive_ratio
  - top_positive_categories (name, count) — топ-5
  - top_negative_categories (name, count) — топ-5
  - monthly_dynamics [{month, positive, negative}]
  - employees_summary [{user_id, name, position, positive, negative, top_category}]
  - НЕ включать author_id ни в каком виде
```

**ReportController::employeeDetail** — детальный отчёт по сотруднику:
```
- Все отзывы о сотруднике за период (без author_id в ответе!)
- Разбивка по категориям
- Динамика по месяцам
- Список комментариев (анонимных)
- Средний score (если используется)
```

**ExportController::exportReport:**
```
- Формат: XLSX и CSV
- Колонки: Дата, Сотрудник (получатель), Эпизод, Тип оценки, Категории, Комментарий
- БЕЗ колонки "Автор"
- Фильтры: период, департамент, сотрудник
```

### Middleware для проверки ролей

```php
// app/Http/Middleware/CheckRole.php
// Проверяет user->role против списка допустимых ролей
// Если роль не совпадает — 403
```

### FormRequest для валидации

**StoreFeedbackRequest:**
```php
'recipient_id' => 'required|exists:users,id|different:author',
'episode_id' => 'required|exists:work_episodes,id',
'sentiment' => 'required|in:positive,negative',
'categories' => 'required|array|min:1|max:3',
'categories.*' => 'exists:feedback_categories,id',
'comment' => 'required|string|min:20|max:2000',
'score' => 'nullable|integer|min:1|max:5',
```

### Seeders

Создай DatabaseSeeder, который генерирует:
- 3 департамента (Разработка, Маркетинг, Аналитика)
- 15-20 сотрудников (с разными ролями)
- 2 руководителя (по одному на департамент + 1 HR)
- 1 админ
- 10-15 рабочих эпизодов с участниками
- 8-10 начальных категорий (5 позитивных, 5 негативных из списка в задании)
- 50-100 отзывов, распределённых за последние 8 месяцев (чтобы можно было показать полугодовой срез)
- 1 активный полугодовой опрос
- Несколько ответов на опрос

**Тестовые пользователи:**
```
admin@test.com / password — Администратор
manager@test.com / password — Руководитель (Разработка)
hr@test.com / password — HR
employee1@test.com / password — Сотрудник
employee2@test.com / password — Сотрудник
... и так далее
```

---

## Frontend (Vue.js 3)

### Структура компонентов и страниц

```
src/
├── layouts/
│   ├── AppLayout.vue          # Основной лейаут с сайдбаром
│   ├── AuthLayout.vue         # Лейаут страницы логина
│
├── pages/
│   ├── Login.vue
│   ├── Dashboard.vue          # Общий дашборд (зависит от роли)
│   │
│   ├── employee/
│   │   ├── EpisodesList.vue       # Мои рабочие эпизоды
│   │   ├── EpisodeCreate.vue      # Создание эпизода
│   │   ├── FeedbackCreate.vue     # Форма отзыва (основной экран)
│   │   ├── MyFeedbacks.vue        # История моих отзывов (только факт, без чужих)
│   │   ├── SurveyForm.vue         # Полугодовой опрос
│   │
│   ├── manager/
│   │   ├── ReportDashboard.vue    # Сводный дашборд с графиками
│   │   ├── EmployeeReport.vue     # Детальный отчёт по сотруднику
│   │   ├── DepartmentReport.vue   # Отчёт по департаменту
│   │   ├── FeedbackBrowser.vue    # Просмотр отзывов (анонимных)
│   │   ├── CategoryManager.vue    # Управление категориями
│   │   ├── SurveyManager.vue      # Управление полугодовыми опросами
│   │   ├── SurveyResults.vue      # Результаты опроса + социометрия
│   │   ├── ExportPage.vue         # Экспорт данных
│   │
│   ├── admin/
│   │   ├── UserManager.vue        # Управление пользователями
│   │   ├── DepartmentManager.vue  # Управление департаментами
│
├── components/
│   ├── ui/
│   │   ├── Button.vue
│   │   ├── Card.vue
│   │   ├── Modal.vue
│   │   ├── Select.vue
│   │   ├── DateRangePicker.vue
│   │   ├── Badge.vue
│   │   ├── Alert.vue
│   │   ├── Tooltip.vue
│   │   ├── Tabs.vue
│   │
│   ├── feedback/
│   │   ├── SentimentSelector.vue     # Переключатель позитив/негатив
│   │   ├── CategoryPicker.vue        # Выбор подкатегорий (chips/иконки)
│   │   ├── ColleagueSelector.vue     # Выбор коллеги
│   │   ├── EpisodeSelector.vue       # Выбор эпизода
│   │   ├── CommentInput.vue          # Поле комментария с подсказками
│   │   ├── FeedbackCard.vue          # Карточка отзыва (для руководителя)
│   │   ├── DuplicateWarning.vue      # Предупреждение о дубликате
│   │
│   ├── charts/
│   │   ├── SentimentPieChart.vue     # Круговая: позитив/негатив
│   │   ├── MonthlyDynamics.vue       # Линейный/столбчатый: динамика по месяцам
│   │   ├── CategoryBarChart.vue      # Горизонтальные столбцы: топ категорий
│   │   ├── EmployeeComparisonChart.vue # Сравнение сотрудников
│   │   ├── RadarChart.vue            # Радар по категориям для сотрудника
│   │   ├── SociometryGraph.vue       # Граф связей (социометрия)
│   │
│   ├── survey/
│   │   ├── ColleagueMultiSelect.vue  # Множественный выбор коллег
│   │   ├── ScaleInput.vue            # Шкала 1-5
│   │   ├── SurveyProgressBar.vue     # Прогресс заполнения
│   │
│   ├── layout/
│   │   ├── Sidebar.vue
│   │   ├── Header.vue
│   │   ├── UserMenu.vue
│   │   ├── RoleBadge.vue
│
├── composables/
│   ├── useAuth.js
│   ├── useFeedback.js
│   ├── useReports.js
│   ├── useExport.js
│   ├── useSurvey.js
│
├── stores/  (Pinia)
│   ├── auth.js
│   ├── feedback.js
│   ├── categories.js
│   ├── reports.js
│   ├── survey.js
│
├── router/
│   ├── index.js              # Vue Router с guard-ами по ролям
│
├── api/
│   ├── axios.js              # Настройка axios с interceptors
│   ├── auth.js
│   ├── feedbacks.js
│   ├── episodes.js
│   ├── reports.js
│   ├── categories.js
│   ├── surveys.js
│   ├── export.js
```

### Описание ключевых экранов

#### 1. FeedbackCreate.vue — Форма отзыва (главный экран сотрудника)

Пошаговая форма (stepper) или единая форма с секциями:

**Шаг 1 — Контекст:**
- Выбор рабочего эпизода из списка своих (EpisodeSelector) или создание нового
- Выбор коллеги из участников эпизода (ColleagueSelector с аватарами и поиском)
- Проверка на дубликат в реальном времени (GET /feedbacks/check?recipient_id=X&episode_id=Y)
- Если дубликат — показать DuplicateWarning с предложением обновить

**Шаг 2 — Оценка:**
- Переключатель настроения: 👍 Позитивный / 👎 Негативный (SentimentSelector — крупные кнопки с цветом)
- Опционально: числовая оценка 1-5 (звёзды или слайдер)
- Выбор 1-3 подкатегорий (CategoryPicker — chips с иконками, фильтрованные по sentiment)
  - Позитивные категории показываются при позитивном sentiment, негативные — при негативном
  - Выбранные подсвечиваются зелёным/красным

**Шаг 3 — Комментарий:**
- Текстовое поле с минимумом 20 символов
- Placeholder-подсказки для конструктивности:
  - Позитив: *"Опишите, что именно коллега сделал хорошо и как это помогло..."*
  - Негатив: *"Опишите ситуацию и её последствия. Избегайте оценок личности, сосредоточьтесь на фактах..."*
- Счётчик символов
- Кнопка "Отправить" с подтверждением

**После отправки:**
- Уведомление об успехе
- Переход на страницу "Мои отзывы" или обратно к списку эпизодов

#### 2. ReportDashboard.vue — Дашборд руководителя

**Верхняя панель:**
- DateRangePicker с пресетами: "Последний месяц", "Последние 3 месяца", "Полугодовой срез (текущий)", "Полугодовой срез (предыдущий)", "Произвольный период"
- Фильтр по департаменту (dropdown)
- Кнопка "Экспорт" (XLSX/CSV)

**Карточки-метрики (верхний ряд):**
- Всего отзывов за период
- Позитивных / Негативных (с процентами и цветовой индикацией)
- Активных сотрудников (оставивших отзывы)
- Средний score (если используется)

**Графики:**
- **Динамика по месяцам** (MonthlyDynamics) — stacked bar chart: позитивные/негативные по месяцам
- **Распределение по категориям** (CategoryBarChart) — горизонтальные столбцы, топ-5 позитивных и топ-5 негативных
- **Круговая диаграмма** (SentimentPieChart) — общее соотношение

**Таблица сотрудников:**
- Список сотрудников с колонками: Имя, Должность, Позитивных, Негативных, Соотношение (прогресс-бар), Топ-категория
- Сортировка по любой колонке
- Клик по строке → переход на EmployeeReport

#### 3. EmployeeReport.vue — Детальный отчёт по сотруднику

**Заголовок:** Имя, должность, департамент, аватар

**Метрики:**
- Позитив/негатив за период
- Radar-chart по категориям (RadarChart)
- Динамика по месяцам (персональная)

**Список анонимных отзывов:**
- Карточки (FeedbackCard): дата, эпизод, sentiment-бейдж, категории-chips, комментарий
- БЕЗ имени автора
- Фильтр по типу (все/позитив/негатив)

#### 4. CategoryManager.vue — Управление категориями

- Таблица категорий: Название, Тип (позитив/негатив), Иконка, Статус (активна/скрыта), Дата создания
- Кнопки: Добавить, Редактировать (модалка), Скрыть/Показать (toggle)
- При скрытии категория не удаляется из БД, но перестаёт отображаться в форме отзыва
- Drag-and-drop для сортировки (опционально)

#### 5. SurveyForm.vue — Полугодовой опрос (сотрудник)

Пошаговая форма:

**Блок 1 — Взаимодействия:**
- "С кем из коллег вы чаще всего взаимодействовали?" — мультиселект с поиском (до 5)
- "С кем было наиболее комфортно работать?" — мультиселект (до 3)
- "К кому вы чаще всего обращались за помощью?" — мультиселект (до 3)
- "Кого бы вы выбрали в команду на сложный проект?" — мультиселект (до 3)

**Блок 2 — Самооценка:**
- Самочувствие (1-5, ScaleInput со смайликами)
- Загруженность (1-5)
- Атмосфера в команде (1-5)
- Мотивация (1-5)

**Блок 3 — Свободный комментарий:**
- Textarea: "Что бы вы хотели отметить о работе в команде за последние полгода?"

**Прогресс-бар сверху, навигация назад/вперёд**

#### 6. SurveyResults.vue — Результаты опроса (руководитель)

- Процент заполнения (кто прошёл / кто нет)
- Средние оценки по шкалам (bar chart)
- Распределение ответов по шкалам (гистограмма)
- Социометрический граф (SociometryGraph):
  - Узлы = сотрудники
  - Рёбра = связи (толщина = количество упоминаний)
  - Цвет узлов по "центральности" (кто чаще упоминается)
  - Фильтр по типу вопроса (взаимодействие / комфорт / помощь / проект)

---

## Дизайн и UX

### Цветовая схема:
- Основной: `#2563EB` (синий) — для навигации и акцентов
- Позитив: `#10B981` (зелёный)
- Негатив: `#EF4444` (красный)
- Нейтральный: `#6B7280` (серый)
- Фон: `#F9FAFB` (светло-серый)
- Карточки: `#FFFFFF` с `shadow-sm`

### Общие принципы:
- Tailwind CSS для всей стилизации
- Скруглённые карточки (`rounded-xl`), мягкие тени
- Анимации переходов между шагами формы
- Адаптивная вёрстка (desktop-first, но корректное отображение на планшетах)
- Sidebar сворачивается в иконки на узких экранах
- Тёмная тема не требуется, но приветствуется

### Иконки для категорий (примеры):
- 🌟 Высокая экспертность
- 📚 Подробно объяснил / помог
- 📋 Хорошее ТЗ / структурировал
- 😊 Доброжелательность
- ⏰ Помог во внерабочее время
- 🚫 Отказал без причин
- ❌ Неверная рекомендация
- 🔄 Перекидывает ответственность
- ⏳ Сорвал сроки
- 😤 Грубость

---

## Роутинг (Vue Router)

```javascript
const routes = [
  { path: '/login', component: Login, meta: { guest: true } },

  // Сотрудник
  { path: '/', component: Dashboard, meta: { auth: true } },
  { path: '/episodes', component: EpisodesList, meta: { auth: true } },
  { path: '/episodes/create', component: EpisodeCreate, meta: { auth: true } },
  { path: '/feedback/create', component: FeedbackCreate, meta: { auth: true } },
  { path: '/feedback/create/:episodeId?', component: FeedbackCreate, meta: { auth: true } },
  { path: '/my-feedbacks', component: MyFeedbacks, meta: { auth: true } },
  { path: '/survey', component: SurveyForm, meta: { auth: true } },

  // Руководитель
  { path: '/reports', component: ReportDashboard, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },
  { path: '/reports/employee/:id', component: EmployeeReport, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },
  { path: '/reports/department/:id', component: DepartmentReport, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },
  { path: '/categories', component: CategoryManager, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },
  { path: '/surveys', component: SurveyManager, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },
  { path: '/surveys/:id/results', component: SurveyResults, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },
  { path: '/export', component: ExportPage, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },
  { path: '/feedbacks', component: FeedbackBrowser, meta: { auth: true, roles: ['manager', 'hr', 'admin'] } },

  // Админ
  { path: '/admin/users', component: UserManager, meta: { auth: true, roles: ['admin'] } },
  { path: '/admin/departments', component: DepartmentManager, meta: { auth: true, roles: ['admin'] } },
];
```

Navigation guards: проверка авторизации и ролей, редирект на /login или /403.

---

## Дополнительные механики

### Антитоксичность:
1. Минимальная длина комментария (20 символов) — не даёт отправить пустышку
2. Подсказки в интерфейсе при негативном отзыве: "Описывайте ситуацию и последствия, а не личные качества"
3. Лимит отзывов в день (10 штук) — от спама
4. Уникальность связки автор-получатель-эпизод — от накрутки
5. Руководитель может пометить отзыв как `is_flagged` (для внутреннего контроля)

### Подготовка к интеграции с CRM:
- Поле `external_task_id` в work_episodes
- Модульная архитектура: сервисный слой отделён от контроллеров
- API-first подход: всё через REST API
- В README описать точки интеграции

---

## Что должно работать в прототипе (чеклист):

1. ✅ Логин под разными ролями (сотрудник, руководитель, HR, админ)
2. ✅ Создание рабочего эпизода и добавление участников
3. ✅ Создание отзыва (позитивного и негативного) с категориями и комментарием
4. ✅ Блокировка повторного отзыва (автор + получатель + эпизод)
5. ✅ Просмотр своих оставленных отзывов (только факт отправки)
6. ✅ Дашборд руководителя с графиками за период
7. ✅ Детальный отчёт по сотруднику (анонимные отзывы)
8. ✅ Управление категориями (CRUD + скрытие)
9. ✅ Экспорт данных в XLSX/CSV
10. ✅ Полугодовой опрос (создание, заполнение, результаты)
11. ✅ Социометрический граф (базовый)
12. ✅ Демо-данные через seeder
13. ✅ Разделение прав: сотрудник не видит чужих отзывов и отчётов

---

## Инструкция по запуску (для README.md)

Подготовь README с секциями:
- Описание проекта
- Скриншоты ключевых экранов
- Стек технологий
- Установка и запуск (composer install, npm install, .env, migrate, seed)
- Тестовые пользователи (таблица с логинами и ролями)
- Архитектура (диаграмма или текстовое описание)
- API-документация (основные эндпоинты)
- Точки интеграции с CRM
- Идеи развития

---

Генерируй код модуль за модулем, начиная с backend (миграции → модели → контроллеры → роуты), затем frontend (store → api → компоненты → страницы). Каждый файл — полностью рабочий, с импортами и экспортами. Используй TypeScript для Vue по желанию (можно обычный JS). Комментарии в коде — на русском.