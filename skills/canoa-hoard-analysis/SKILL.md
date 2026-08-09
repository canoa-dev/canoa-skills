---
name: canoa-hoard-analysis
description: Use when analyzing coin hoards of a region, charts and PDF.
version: 1.0.0
author: CANOA (canoanumis.org)
license: CC BY 4.0
platforms: [linux]
metadata:
  hermes:
    tags: [numismatics, hoards, analysis, charts, pdf, canoa]
    homepage: https://canoanumis.org
    related_skills: [canoa-api, django-numismatic-catalog]
---

# Анализ кладов региона → распределение монет → графики → PDF-статья

Перебор всех кладов, найденных в одном регионе (bbox по координатам), подсчёт
распределения монет по императорам/правителям и периодам, построение графиков,
генерация статьи в PDF. Рабочее окружение — /home/kali/numismatics (Django ORM).

## Когда использовать
- «проанализируй клады Сицилии/Британии/Лузитании…»
- «построй распределение монет по императорам в кладах региона»
- «сделай статью-отчёт по кладам области в PDF»

## Данные и подключение

Модели (catalog/models.py):
- `Hoard` — клад: `findspot` (FK, координаты), `coin_types` (M2M), `coin_count`,
  `start_date`/`end_date`, `label`, `dataset` (chre/coinhoards/igch/chrr)
- `Findspot` — место находки: `label`, `latitude`, `longitude`
- `CoinType` — тип монеты: `authority` (FK, правитель/император), `mint` (FK),
  `start_date`/`end_date` (годы, отрицательные = до н.э.)
- `Authority` — правитель: `label`

Объёмы: 9 833 клада (654 с координатами findspot, 5 490 со связанными типами).

Подключение к БД — через Django ORM (venv `/home/kali/django6_env`, settings
`numismatics.settings`, запускать из `/home/kali/numismatics`):
```bash
cd /home/kali/numismatics && source /home/kali/django6_env/bin/activate
python3 -c "
import django, os
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'numismatics.settings')
django.setup()
from catalog.models import Hoard
..."
```
Данные dev-зеркала БД (для разработки без прода): 192.168.78.0:3306 (пароль в
.env проекта). Прод-БД Reg.Cloud — ТОЛЬКО с прод-сервера по SSH
(`ssh deploy@canoanumis.org`), пароль читать из .env прода, не передавать в cmdline.

## Шаги

### 1. Выбрать регион (bbox по координатам findspot)
Регион задаётся пользователем по названию. Для известных регионов — bbox
(мин/макс широта, мин/макс долгота). Примеры:
- Сицилия: lat 36.5–38.5, lon 12.0–16.0 (21 клад, 17 с типами)
- Южная Британия: lat 50.0–52.0, lon -5.0–1.5
- Лузитания (Португалия): lat 37.0–42.0, lon -10.0–-6.0
- Галлия: lat 43.0–51.0, lon -5.0–8.0

Фильтр кладов региона:
```python
from catalog.models import Hoard
hoards = (Hoard.objects
          .filter(findspot__latitude__gte=LAT_MIN, findspot__latitude__lte=LAT_MAX,
                  findspot__longitude__gte=LON_MIN, findspot__longitude__lte=LON_MAX)
          .exclude(coin_types__isnull=True))
```

### 2. Агрегация: по императорам и периодам
```python
from collections import Counter
au = Counter(); per = Counter(); total = 0
for h in hoards.prefetch_related('coin_types__authority'):
    for ct in h.coin_types.all():
        total += 1
        if ct.authority: au[ct.authority.label] += 1
        if ct.start_date is not None:
            y = ct.start_date
            if y < -509: per['Архаика (до 500 до н.э.)'] += 1
            elif y < -400: per['Классика (500–400 до н.э.)'] += 1
            elif y < -300: per['Поздняя классика (400–300 до н.э.)'] += 1
            elif y < -27: per['Эллинизм (300–27 до н.э.)'] += 1
            elif y < 476: per['Римская эпоха (27 до н.э.–476)'] += 1
            else: per['Византия/Средневековье (после 476)'] += 1
```
Учесть: многие типы в кладах — без authority (анонимные/городские), их считать
в отдельную группу «без правителя». Также вывести число кладов региона и монет
всего (coin_count — декларируемое число монет клада, осторожно: часто 0).

### 3. Графики — СИСТЕМНЫЙ matplotlib (не venv!)
**matplotlib в django6_env НЕ установлен и НЕ ставится** (PIL — симлинк на
системный каталог, Permission denied). Использовать системный python3:
```bash
python3 -c "
import matplotlib; matplotlib.use('Agg')
import matplotlib.pyplot as plt
..."
```
Системный matplotlib 3.10.7 в /usr/lib/python3/dist-packages. Графики:
1. Топ-15 правителей (горизонтальный bar, log при большом разбросе)
2. Периоды (bar с подписями долей %)
3. Распределение монет по кладам (scatter по координатам findspot, размер = число монет)
Сохранять в PNG 150 dpi. Кириллица: `plt.rcParams['font.family'] = 'DejaVu Sans'` (поддерживает кириллицу).

### 4. PDF-статья — headless Chromium
reportlab/weasyprint/wkhtmltopdf НЕ установлены. Рабочий путь — HTML + Chromium:
```bash
cat > /tmp/article.html <<'EOF'
<!DOCTYPE html><html lang="ru"><head><meta charset="utf-8">
<style>body{font-family:sans-serif;max-width:800px;margin:40px auto;line-height:1.6}
h1{color:#8a6d1a} img{max-width:100%} table{border-collapse:collapse}
td,th{border:1px solid #ccc;padding:6px 10px}</style></head>
<body>
<h1>Клады региона: распределение монет</h1>
<p>…текст статьи: регион, число кладов, число монет, выводы по топ-правителям…</p>
<img src="file:///tmp/chart_rulers.png">
<img src="file:///tmp/chart_periods.png">
<img src="file:///tmp/chart_map.png">
</body></html>
EOF
chromium --headless=new --no-sandbox --disable-dev-shm-usage \
  --print-to-pdf=/tmp/article.pdf --no-pdf-header-footer /tmp/article.html
```
Проверка: `ls -la /tmp/article.pdf` (должен быть >20 КБ).

## Питфоллы
- **Эллинистические регионы: authority почти всегда пуст!** На Сицилии 855 из 883 типов (97%) без правителя — это городские чеканы. Если доля «без правителя» >50%, строить распределение по mint (`ct.mint.label`), а не по authority
- matplotlib: ТОЛЬКО системный python3 (/usr/bin/python3), НЕ /home/kali/django6_env/bin/python
- ORM: запускать из /home/kali/numismatics с venv django6_env (settings numismatics.settings); скрипт из /tmp требует `sys.path.insert(0, '/home/kali/numismatics')`
- prefetch_related('coin_types__authority') — без него N+1 запросов на 5k+ типов
- Прод-БД: только через SSH deploy@canoanumis.org; пароль из .env прода через
  `PW=$(sudo -n grep -oE 'DB_PASSWORD=[^ ]+' .env | cut -d= -f2)`; тяжёлые запросы
  гонять на проде, а не по SSH-туннелю
- coin_count у кладов часто 0 — реальное число монет считать по coin_types
- Пустые группы (0 монет) в графиках не рисовать
- Файлы кладов: многие датасеты (chre) — клады эллинизма, (igch) — греческие;
  уточнять у пользователя, какие включать

## Верификация
```bash
cd /home/kali/numismatics && source /home/kali/django6_env/bin/activate
python3 -c "
import django, os; os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'numismatics.settings'); django.setup()
from catalog.models import Hoard
print('кладов с findspot:', Hoard.objects.exclude(findspot__isnull=True).count())
"
```
Ожидаемо: 654. График: python3 с matplotlib → PNG существует. PDF: chromium → файл >20 КБ.
