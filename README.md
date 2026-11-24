# F1 Monitor — Гран-при

Клиентское веб-приложение для мониторинга пилотов и команд Формулы-1 (результаты, зачёты, напарники, прогнозы, дашборды).

Сайт: https://slataria-hub.github.io/f1/

## Как работают данные

- **Локально** — `localStorage` (ключ `f1app`).
- **Общие** — файл `data.json` в корне репозитория. При открытии сайта, если на GitHub версия **новее** локальной (поле `meta.updated`), данные подтягиваются автоматически. Прогнозы с устройства сохраняются.
- Кнопка **⚙ → Подтянуть общие данные с сайта** принудительно загружает `data.json`.

### Ручная публикация (с телефона / ноутбука)

1. Загрузите или введите результаты этапов.
2. ⚙ → **Скачать data.json для публикации**.
3. Замените `data.json` в репозитории → commit + push.
4. Через 1–2 минуты все, кто открыл сайт, увидят обновление.

## Автообновление через GitHub Actions (рекомендуется)

Создайте файл `.github/workflows/update-f1.yml`:

```yaml
name: Update F1 data
on:
  schedule:
    # Каждый понедельник и после типичных гоночных дней (UTC)
    - cron: '0 6 * * 1'
    - cron: '30 20 * * 0'   # воскресенье вечер
  workflow_dispatch:        # ручной запуск из Actions

jobs:
  update:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Fetch latest results and merge into data.json
        run: |
          python - <<'PY'
          import json, urllib.request, time
          from pathlib import Path

          API = "https://api.jolpi.ca/ergast/f1/"
          path = Path("data.json")
          data = json.loads(path.read_text(encoding="utf-8")) if path.exists() else {"year": 2026, "gps": [], "teams": [], "drv": []}

          def get(url):
              req = urllib.request.Request(url, headers={"User-Agent": "f1-monitor-gh-action"})
              with urllib.request.urlopen(req, timeout=30) as r:
                  return json.load(r)["MRData"]["RaceTable"]

          year = data.get("year") or 2026
          try:
              table = get(f"{API}{year}.json")
          except Exception as e:
              print("schedule fail", e)
              raise SystemExit(0)

          def gy(g):
              return g.get("y") if g.get("y") is not None else year

          existing = {(gy(g), g["r"]): g for g in data.get("gps", [])}
          for r in table.get("Races", []):
              key = (year, int(r["round"]))
              if key not in existing:
                  g = {
                      "id": f"g{year}r{r['round']}",
                      "n": r["raceName"],
                      "r": int(r["round"]),
                      "y": year,
                      "sp": "Sprint" in r,
                      "race": [], "spr": [], "q": [], "ov": {}, "pred": {"race": [], "spr": []},
                      "d": r.get("date"),
                  }
                  data.setdefault("gps", []).append(g)
                  existing[key] = g
              else:
                  existing[key]["d"] = r.get("date")
                  if "Sprint" in r:
                      existing[key]["sp"] = True
                  if not existing[key].get("n") or existing[key]["n"] == "Новый гран-при":
                      existing[key]["n"] = r["raceName"]

          # Optionally fill missing results for finished races (light version)
          # Full load is better done in the browser via «Загрузить расписание и все результаты»
          data["meta"] = {"updated": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()), "source": "github-actions"}
          path.write_text(json.dumps(data, ensure_ascii=False, separators=(",", ":")), encoding="utf-8")
          print("updated meta", data["meta"])
          PY
      - name: Commit if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data.json
          git diff --staged --quiet || (git commit -m "chore: refresh F1 schedule/meta [automated]" && git push)
```

После добавления workflow:

- Расписание подтягивается автоматически.
- Полную загрузку результатов этапов удобнее делать один раз в браузере (⚙ → «Загрузить расписание и все результаты» или «Загрузить историю»), затем опубликовать `data.json`.
- Action обновляет `meta.updated`, чтобы клиенты подхватили файл.

## Дашборды

- **Очки** — накопленные очки по этапам (пилоты / команды).
- **Места** — эволюция позиций в чемпионате.
- **Форма** — очки за последние 5 этапов (столбцы).

Клик по флагу/этапу сдвигает «курсор», можно сравнивать срез зачёта.

## API

Используется [Jolpica](https://api.jolpi.ca/ergast/f1/) (совместимый наследник Ergast). Официальный Ergast больше не отвечает.

## Локальная разработка

Откройте `index.html` через любой статический сервер (нужен `data.json` рядом) или просто на GitHub Pages.

```bash
python -m http.server 8080
```
