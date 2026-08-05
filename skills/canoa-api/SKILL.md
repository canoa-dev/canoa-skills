---
name: canoa-api
description: Use when researching ancient coins via the CANOA API.
version: 1.0.0
author: CANOA (canoanumis.org)
license: CC BY 4.0
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [numismatics, history, research, education, api, ancient-coins, canoa]
    homepage: https://canoanumis.org
    related_skills: [arxiv, open-access-papers]
---

# CANOA API — как пользоваться (нумизматика, история, образование)

CANOA (https://canoanumis.org) — открытый корпус монет древнего мира: Греция, Рим, Персия, Карфаген, Византия.
Объём: ~128 500 типов монет, 540 000+ музейных экземпляров, 9 833 клада, 17 074 штемпеля, 2 600+ монетных дворов.
Данные типов — ODbL (источник numismatics.org); изображения принадлежат музеям-держателям (многие — только некоммерческое использование с атрибуцией). Перед публикацией результатов проверяй /LICENSES.md и лицензию конкретной коллекции.

## Когда использовать
- Поиск монет по правителю, двору, номиналу, металлу, периоду, датасету
- Карточка конкретного типа монеты (легенды, описание, датировка, метрология)
- Списки/статистика для статей, уроков, докладов, научных работ
- Построение карт монетных дворов (GeoJSON)

## Эндпоинты (все — GET, JSON, без ключа)

| Эндпоинт | Что даёт |
|---|---|
| `/api/search?q=<текст>` | автодополнение: правители, дворы, номиналы, монеты |
| `/api/coins` | список монет с фильтрами (см. ниже) |
| `/api/coins/<slug>/` | карточка монеты (например `/api/coins/ric-i-second-edition-nero-1/`) |
| `/api/coin/<id>/` | карточка по числовому id |
| `/api/filter-options?field=mint` | справочные значения фильтров (mint, authority, denomination, material) |
| `/api/mints.geojson` | монетные дворы в GeoJSON (фильтры: authority, denomination, material, dataset, q, has_coins) |
| `/llms.txt`, `/llms-full.txt` | документация для LLM/агентов |
| `/LICENSES.md` | лицензии по коллекциям |

### Фильтры /api/coins
- `q` — текст (токены AND по названию, каталожному номеру, легендам)
- `authority`, `denomination`, `material` — **URI** nomisma.org, напр. `http://nomisma.org/id/nero`
- `mint` — **числовой id** (не slug! брать из `/api/filter-options?field=mint`)
- `dataset` — ocre, crro, pella, iris, sco, pco, bigr, agco, aod, lco, coi, cm, do_byzant, oscar, chre, chrr, coinhoards, iacb
- `has_image` — `1` (только с фото) / `0` (все)
- `date_from`, `date_to` — годы: монеты, чей период пересекает диапазон (отрицательные = до н.э.)
- `sort` — name, -name, date, -date
- `page`, `per_page` — пагинация (per_page максимум 100)
- `format=html` — тот же список простым HTML (для ссылок без JS)

## Быстрые примеры

```bash
# Монеты Нерона из Рима (id двора 36):
curl "https://canoanumis.org/api/coins?authority=http://nomisma.org/id/nero&mint=36&per_page=100"

# Монеты Рима 54–68 гг. н.э. (период):
curl "https://canoanumis.org/api/coins?date_from=54&date_to=68&mint=36"

# Только с фото:
curl "https://canoanumis.org/api/coins?q=denarius&has_image=1&sort=date&per_page=50"

# Карточка монеты:
curl "https://canoanumis.org/api/coins/ric-i-second-edition-nero-63/"

# Значения фильтров (id дворов, URI правителей):
curl "https://canoanumis.org/api/filter-options?field=mint"
curl "https://canoanumis.org/api/filter-options?field=authority"

# Автодополнение:
curl "https://canoanumis.org/api/search?q=trajan"

# Дворы в GeoJSON (для карт):
curl "https://canoanumis.org/api/mints.geojson?has_coins=1"
```

## Рецепты

### 1. Для детей и школ (просто и наглядно)
- `/api/search?q=<имя>` — найти правителя (тип `ruler`, url ведёт в каталог)
- `/api/coins?q=<имя>&has_image=1&format=html` — страница с картинками без программирования
- Задание: «найди 5 монет Нерона с фото, укажи их каталожные номера»

### 2. Для любителей: подборка «золото Рима II века»
```python
import urllib.request, json
url = "https://canoanumis.org/api/coins?material=http://nomisma.org/id/av&date_from=96&date_to=192&per_page=100"
data = json.load(urllib.request.urlopen(url))
for c in data["results"]:
    print(c["ocre_id"], "|", c["label"], "|", c["mint"], "|", c["date_from"], "-", c["date_to"])
print("всего:", data["count"])
```

### 3. Для исследователя: экспорт в CSV
```python
import urllib.request, json, csv
def fetch(page):
    u = f"https://canoanumis.org/api/coins?authority=http://nomisma.org/id/nero&per_page=100&page={page}"
    return json.load(urllib.request.urlopen(u))
first = fetch(1)
with open("nero_coins.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["ocre_id", "label", "mint", "date_from", "date_to", "url"])
    for page in range(1, first["num_pages"] + 1):
        for c in fetch(page)["results"]:
            w.writerow([c["ocre_id"], c["label"], c["mint"], c["date_from"], c["date_to"], c["url"]])
print("экспортировано:", first["count"])
```
- Отрицательные `date_from/date_to` = годы до н.э. (напр. -27 = 27 г. до н.э.)
- `label_ru` в ответах — русские метки (материал/номинал/правитель) для русскоязычных публикаций

### 4. Цитирование и лицензии (обязательно для публикаций)
- Типы/данные: ODbL 1.0 — указывай источник CANOA (canoanumis.org) и numismatics.org
- Изображения: © музей-держатель, лицензия коллекции — в /LICENSES.md (многие — NC, только некоммерчески)
- Стандартная атрибуция: «Данные: CANOA (canoanumis.org), ODbL; изображение: © <музей>»

## Ловушки
- `mint` в фильтрах — числовой id, не slug (slug только в URL /mints/<slug>/)
- `authority/denomination/material` — полные URI nomisma.org, а не короткие имена
- per_page максимум 100 — для полной выборки итерируй по страницам (num_pages в ответе)
- Изображения — отдельные URL (image в /api/coins, obverse_image/reverse_image в карточке); часть источников утрачена (Gallica/PAS) — у таких монет image отсутствует, это не ошибка
- Закрытые коллекции (abc_exemplars, rutgers, ashm_ocre, ashm_pella) в API не отдаются
- Не долби API пачками — вежливость: паузы между запросами при больших выборках
- /search/ кэшируется 60 с — после изменения фильтров возможна задержка обновления

## Проверка
```bash
curl -s "https://canoanumis.org/api/search?q=nero" | head -c 400
curl -s "https://canoanumis.org/api/coins/ric-i-second-edition-nero-63/" | head -c 400
curl -s -o /dev/null -w "%{http_code}\n" "https://canoanumis.org/api/coins?per_page=1"
```
Ожидаемо: HTTP 200, корректный JSON, поле `count` в списках (для проверки структуры сохрани ответ в файл и открой его отдельно, без конвейера в интерпретатор).
