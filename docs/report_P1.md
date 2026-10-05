# Отчёт по практическому заданию Р1: CI/CD на GitVerse

### Ссылка на репозиторий
https://gitverse.ru/kolitchenkova/ssg-lab

### 1. Схема пайплайна
Пайплайн публикации реализован в файле `.gitverse/workflows/pipeline.yml` и разделен на 3 изолированные стадии:

```
[Триггер: push / dispatch]
           │
           ▼
    ┌─────────────┐
    │  Job: lint  │  (Проверка ссылок, строгая сборка mkdocs build --strict)
    └──────┬──────┘
           │ needs: lint (успех)
           ▼
    ┌─────────────┐
    │  Job: build │  (Компиляция HTML, сохранение site-artifact)
    └──────┬──────┘
           │ needs: build + условие: github.ref == 'refs/heads/main'
           ▼
    ┌─────────────┐
    │ Job: deploy │  (Скачивание артефакта, проверка секрета, публикация)
    └─────────────┘
```

* **Стадии и зависимости:**
  * `lint` — независимая стадия первичного контроля качества.
  * `build` — зависит от `lint` (`needs: lint`).
  * `deploy` — зависит от `build` (`needs: build`) и содержит условие ветки.
* **Артефакты:** Каталог `site/` упаковывается на стадии `build` в архив `static-site-artifact` (`actions/upload-artifact@v4`) и передаётся на стадию `deploy` (`actions/download-artifact@v4`).

---

### 2. События запуска и поведение пайплайна
* **Push в ветку `main`:** Выполняются последовательно все три стадии (`lint` $\rightarrow$ `build` $\rightarrow$ `deploy`). Готовый сайт доставляется на целевой хостинг.
* **Push в любые другие ветки (feature, dev, PR):** Выполняются только проверки `lint` и `build`. Стадия `deploy` автоматически пропускается (`Skipped`), исключая попадание нестабильного кода на боевую площадку.
* **Ручной запуск (`workflow_dispatch`):** Запуск пайплайна по кнопке из веб-интерфейса GitVerse с сохранением всех правил ветвления.

---

### 3. Ошибка с `uv` на GitVerse и его устранение
* **Текст ошибки:**  
  `No (valid) GitHub token provided. Falling back to anonymous. Requests might be rate limited.`  
  `::error::API rate limit exceeded for 89.232.160.13.`
* **Причина:** Официальный экшен `astral-sh/setup-uv@v5` обращается к REST API GitHub (`api.github.com`) для поиска версии `latest`. На GitVerse отсутствует сервисный токен GitHub, а общий пул исходящих IP-адресов раннеров платформы быстро исчерпал лимит анонимных запросов (60 запросов/час).
* **Решение:** Замена экшена на установку  через shell-скрипт (`curl -LsSf https://astral.sh/uv/install.sh | sh`) с экспортом пути в `$GITHUB_PATH`, что полностью исключило обращение к GitHub API.

---

### 4. Анализ замеров времени и поведения кэша

| Стадия пайплайна          | Время БЕЗ кэша (холодный старт) | Время С кэшем (горячий старт) | Последовательные коммиты |
| :------------------------ | :-----------------------------: | :---------------------------: | :----------------------: |
| `lint`                    |             2м 31 с             |           2 м 44 с            |           52 с           |
| `build`                   |              32 с               |             51 с              |           31 с           |
| `deploy`                  |               7 с               |             11 с              |           7 с            |
| **СУММАРНОЕ ВРЕМЯ CI/CD** |            3 м 10 с             |           3 м 49 с            |         1 м 30 с         |
Время после кеширования увеличислось:
  1. **Отсутствие бэкенда кэширования:** Экшен `actions/cache@v4` на GitVerse завершился с предупреждением:  
     `[warning] Cache action is only supported on GHES version >= 3.5...`  
     На публичных shared-раннерах GitVerse сервер кэша не развернут, поэтому кэш не сохранялся — все запуски фактически выполнялись «на холодную».
  2. **Сетевые флуктуации:** До 85% времени стадии `lint` занимает загрузка дистрибутива Python 3.12 и `uv` из сети. Небольшое увеличение времени при повторном коммите вызвано задержкой внешнего сетевого канала раннера при повторном скачивании пакетов.
Однако, при серии последовательных коммитов было замечено сокращение времени выполнения пайплайна, что может быть связано с уменьшением расходов на загрузку образов и кэширование сетевых артефактов Python и uv.
---

### 5. Скриншоты и разбор лога ошибки

#### Успешный запуск
Все стадии (`lint`, `build`, `deploy`) выполнены успешно (зеленый статус). Секрет `DEPLOY_AUTH_KEY` корректно скрыт маской `***` в логах шага `deploy`.
<img width="938" height="444" alt="image" src="https://github.com/user-attachments/assets/f788ae81-526d-4fa6-aefc-f2690ca8b84e" />

#### Намеренно проваленный запуск
Для проверки работы строгой валидации в файл `docs/index.md` была добавлена несуществующая ссылка (not_index.md)`.

* **Лог ошибки:**
```text
INFO    -  Cleaning site directory

(https://gitverse.ru/kolitchenkova/ssg-lab/cicd/1733747?job=3404731#step:7:21)

INFO    -  Building documentation to directory: /workspace/kolitchenkova/ssg-lab/site

(https://gitverse.ru/kolitchenkova/ssg-lab/cicd/1733747?job=3404731#step:7:22)

WARNING -  Doc file 'index.md' contains a link 'not_link.md', but the target is not found among documentation files.
(https://gitverse.ru/kolitchenkova/ssg-lab/cicd/1733747?job=3404731#step:7:24)

Aborted with 1 warnings in strict mode!

(https://gitverse.ru/kolitchenkova/ssg-lab/cicd/1733747?job=3404731#step:7:25)

##[error]Process completed with exit code 1.
```
* **Разбор поведения:** Флаг `--strict` перевел предупреждение о битой ссылке в ошибку (`exit code 1`). Стадия `lint` упала (`Failed`), а последующие стадии `build` и `deploy` были заблокированы (`Blocked/Cancelled`). Сломанный контент не был скомпилирован и не попал на сайт.

<img width="930" height="855" alt="image" src="https://github.com/user-attachments/assets/aabcc718-0c3e-4198-b320-f6e50c56ebed" />


---

### 6. Бейдж статуса сборки в `README.md`
В файл `README.md` репозитория добавлен бейдж статуса:

```markdown
[![GitVerse CI/CD](https://gitverse.ru/api/repos/kolitchenkova/ssg-lab/actions/workflows/pipeline.yml/badge.svg)](https://gitverse.ru/kolitchenkova/ssg-lab/cicd)
```

---

### 7. Сопоставление с GitHub Actions (Связь с заданием Т3)

| Критерий | GitHub Actions | GitVerse CI | Что потребовалось изменить при миграции |
| :--- | :--- | :--- | :--- |
| **Синтаксис пайплайнов** | Стандартный синтаксис GHA (YAML) | Полная совместимость с GHA на базе Act Runner | Базовый синтаксис (`jobs`, `steps`, `needs`, `if`) перенесен без изменений. |
| **Механизм публикации** | Проприетарные экшены `deploy-pages` и OIDC | `actions/upload-artifact` + прямой деплой | Полный отказ от закрытого экшена `deploy-pages` в пользу выгрузки артефактов. |
| **Кэширование** | Нативный сервис `actions/cache` из коробки | `actions/cache` не поддерживается на shared-раннерах | Использование предсобранных сред или оптимизация через системный `python3`. |
| **Внешние API** | Автоматическая авторизация через `GITHUB_TOKEN` | Анонимные запросы к внешним API блокируются (403) | Отказ от экшенов, опрашивающих внешние REST API; переход на автономные скрипты установки (`curl`). |
| **Безопасность секретов** | Маскирование в логах (`***`) | Маскирование в логах (`***`) | Работа с секретами платформы идентична и безопасна. |
