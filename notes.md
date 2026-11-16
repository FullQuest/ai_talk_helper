# Использование AI в режиме чата.

```sh
# packer
pip install repobundler

# Эта команда создает в корне .md файл, в котором упакован весь проект. 
# Его отправляем в нейронку, пишем что нужно сделать и добавляем промпт ниже.
ctx .
```

Промпт, вставляемый после ТЗ:

```
=== ФОРМАТ ВЫВОДА (ОБЯЗАТЕЛЬНО) ===

В первую очередь, если есть вопросы/уточнения - ЗАДАЙ их и только их. В противном случае действуй так:

Ты должен вернуть результат строго в формате Markdown-файла, пригодного для автоматического разбора. 
Никаких пояснений, извинений или текста вне блоков кода. Только перечисленные ниже блоки.

Для каждого созданного или изменённого файла выведи ровно одну секцию в таком виде:

### FILE: относительный/путь/к/файлу.расширение
```язык
ПОЛНОЕ содержимое файла (не diff, не фрагмент, а весь итоговый файл целиком).
```


Ответ копируем в rsp.md и выполняем скрипт ниже.
В корень положим `unp.py`

```python
import re
import os
from pathlib import Path

RESPONSE_FILE = "rsp.md"
OUTPUT_DIR = Path(".")  # куда складывать файлы
# OUTPUT_DIR = Path("./extracted/")  # Можно сложить файлы и отдельно
OUTPUT_DIR.mkdir(exist_ok=True, parents=True)

text = Path(RESPONSE_FILE).read_text(encoding="utf-8")

# Ищем секции: ### FILE: путь  + следующий за ним блок ` ` `lang ... ` ` `
# Флаг DOTALL, чтобы . захватывал переводы строк
pattern = re.compile(
    r"^### FILE:\s*(.+?)\s*\n"      # заголовок с путём
    r"```[^\n]*\n"                  # открывающий fence (язык игнорируем)
    r"(.*?)"                        # содержимое (лениво)
    r"\n```",                       # закрывающий fence
    re.MULTILINE | re.DOTALL
)

matches = pattern.findall(text)
if not matches:
    raise SystemExit("Не найдено ни одной секции '### FILE: ...'. Проверьте формат ответа.")

for rel_path, content in matches:
    # Базовая защита от path traversal
    safe_path = (OUTPUT_DIR / rel_path).resolve()
    if not str(safe_path).startswith(str(OUTPUT_DIR.resolve())):
        print(f"[SKIP] Недопустимый путь: {rel_path}")
        continue
    safe_path.parent.mkdir(parents=True, exist_ok=True)
    safe_path.write_text(content, encoding="utf-8")
    print(f"[OK] {rel_path} ({len(content)} символов)")
```


