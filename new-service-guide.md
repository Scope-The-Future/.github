# Как создать новый микросервис

Пошаговая инструкция: от задачи в Weeek до работающего сервиса в `docker compose`.
В `<угловых скобках>` стоят значения, которые нужно заменить своими.

**Что получится:** два PR по одной задаче — в `cloud-platform` (код сервиса) и в `infra` (запись в compose).

---

## 0. Подготовка

1. Создайте задачу в Weeek (или возьмите существующую), запомните её номер.
2. Придумайте имя сервиса: строчными буквами, через дефис, оканчивается на `-service`.
   Примеры: `order-service`, `restaurant-menu-service`.
   Это имя используется везде одинаково: папка, запись в compose, адрес в сети Docker.
3. Создайте ветку **в обоих репозиториях** с одним и тем же названием `<номер задачи>-<описание>`:

```bash
# в cloud-platform
cd cloud-platform
git checkout main && git pull
git checkout -b 15-payment-service

# в infra (отдельный терминал или после первого PR)
cd ../infra
git checkout main && git pull
git checkout -b 15-payment-service
```

---

## 1. Репозиторий cloud-platform: код сервиса

### 1.1. Структура файлов

Самая простая структура одного сервиса:

```
cloud-platform/
└── services/
    └── <имя-сервиса>/
        ├── app/
        │   ├── __init__.py       ← пустой файл
        │   └── main.py           ← код сервиса
        ├── .dockerignore
        ├── Dockerfile
        └── requirements.txt
```

Папка `.venv` тоже лежит в папке сервиса, но в git не попадает (она в `.gitignore`).

### 1.2. Виртуальное окружение и зависимости

Выполняйте команды **из папки нового сервиса**, чтобы окружение было своим у каждого сервиса.

```bash
cd cloud-platform/services/<имя-сервиса>
python -m venv .venv
```

Активация окружения:

| Оболочка | Команда |
|---|---|
| PowerShell (Windows) | `.venv\Scripts\Activate.ps1` |
| cmd (Windows) | `.venv\Scripts\activate.bat` |
| Linux / macOS / Git Bash | `source .venv/bin/activate` |

Если PowerShell запрещает запуск скриптов, один раз выполните:
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

Проверьте, что активировано нужное окружение (путь должен вести в `.venv` этого сервиса):

```bash
python -c "import sys; print(sys.executable)"
python --version     # желательно 3.11.x — как в Dockerfile
```

Установите зависимости и зафиксируйте их в файле:

```bash
pip install fastapi "uvicorn[standard]"
pip freeze > requirements.txt
```

> Каждый раз, когда ставите новую библиотеку (`pip install ...`), заново выполняйте `pip freeze > requirements.txt`.
> Если забыть, локально всё будет работать (пакет лежит в вашем `.venv`), а в Docker сервис упадёт с `ModuleNotFoundError`.


### 1.3. Код сервиса

Создайте пустой файл `app/__init__.py`, затем `app/main.py`:

```python
from fastapi import FastAPI

app = FastAPI(title="<имя-сервиса>")


@app.get("/health")
def health():
    # Проверка, что сервис жив (пригодится для healthcheck позже)
    return {"status": "ok"}


@app.get("/data")
def get_data():
    return {"service": "<имя-сервиса>", "message": "Привет"}
```


### 1.4. Dockerfile

Файл называется именно `Dockerfile` (с одной заглавной буквы, без расширения):

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Имя в строке `COPY requirements.txt .` должно **в точности** совпадать с именем файла в папке.


### 1.5. Коммит и PR

```bash
git add .
git commit -m "Добавлен <имя-сервиса> с методом GET /data"
git push -u origin <имя-ветки>
```

Откройте PR на GitHub. В описании укажите номер задачи Weeek и **ссылку на PR в `infra`** (его создадим на следующем шаге).

---

## 2. Репозиторий infra: подключение сервиса в compose

Вносится в **отдельной ветке с тем же названием** (`15-payment-service`) в репозитории `infra`.

### 2.1. Шаблон записи в `docker-compose.yml`

Добавьте блок в раздел `services:`, соблюдая отступы:

```yaml
  <имя-сервиса>:
    build: ../cloud-platform/services/<имя-сервиса>
    ports:
      - "<внешний-порт>:8000"
    # Раскомментируйте, если сервису нужны адреса других сервисов или БД:
    # environment:
    #   ДРУГОЙ_СЕРВИС_URL: http://<имя-другого-сервиса>:8000
    # depends_on:
    #   - <имя-другого-сервиса>
```

Правила:
- **Внутри Docker** сервисы обращаются друг к другу по имени сервиса и порту `8000`: `http://order-service:8000`. Адрес `localhost` из контейнера не подходит.
- **Внешний порт** (слева от двоеточия) должен быть уникальным. Он нужен вам для браузера и curl. Занятые порты ведите списком в `infra/README.md` и дописывайте туда новый:

| Сервис | Внешний порт |
|---|---|
| order-service | 8001 |
| restaurant-menu-service | 8002 |
| *(следующий свободный)* | 8003 |

- Имена переменных окружения пишите заглавными буквами через подчёркивание: `RESTAURANT_MENU_SERVICE_URL`. В коде они читаются так: `os.getenv("RESTAURANT_MENU_SERVICE_URL")`, и написание должно совпадать с compose до буквы.
- Секреты (пароли) в compose не пишутся, они берутся из `.env`.

### 2.2. Коммит и PR

```bash
git add .
git commit -m "Добавлен <имя-сервиса> в docker-compose"
git push -u origin <имя ветки>
```

В описании PR укажите номер задачи Weeek и ссылку на PR в `cloud-platform`.

---

## 3. Порядок слияния

1. Сначала ревью и merge PR в **cloud-platform** (код сервиса).
2. Затем merge PR в **infra**.

Если сделать наоборот, compose в `infra` будет ссылаться на папку, которой ещё нет, и `docker compose up --build` сломается у всех.

После слияния ветки удалятся автоматически. Обновите оба репозитория:

```bash
git checkout main && git pull
```

---

## 4. Проверка

Запуск только нового сервиса (и того, от чего он зависит):

```bash
cd infra
docker compose up --build <имя-сервиса>
```

Откройте `http://localhost:<внешний-порт>/docs` и вызовите методы кнопкой **Try it out**.

Остановка: `Ctrl+C` или `docker compose down`.

---

## 5. Чек-лист перед PR

- [ ] Имя сервиса в нижнем регистре через дефис, одинаковое в папке и в compose
- [ ] `venv` создан в папке сервиса, `requirements.txt` обновлён (`pip freeze`)
- [ ] Сервис запускается через `uvicorn` и открывается `/docs`
- [ ] Файлы названы точно: `Dockerfile`, `requirements.txt`, `.dockerignore`
- [ ] Есть пустой `app/__init__.py`
- [ ] `docker compose up --build <имя-сервиса>` проходит без ошибок
- [ ] В `infra` выбран свободный внешний порт и он записан в README
- [ ] Ветки в обоих репозиториях названы по номеру задачи, PR ссылаются друг на друга
- [ ] В коммиты не попали `.venv`, `.env`, пароли

