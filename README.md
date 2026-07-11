Привет, если вы это читаете, вы лучшие.
В общем этот бот создавался ещё зимов 2024-2025 для аренды на clore.ai. Он прошёл много итераций от простого кода на bash до бота на Python.

Вы можете изменять этот код как хотите, а можете следить за релизами и испольовать его для себя.

На данный момент бот умеет:
1. Брать в аренду за USD, USDT, Clore.
2. Делать проверку активных серверов при работе.
3. Настраивать разнообразные майнинговые задачи на ваши сервера в зависимости от типа железа.
4. Ведёт SQL базу в которой считает профит данных серверов на разных комбинациях профилей(CPU и GPU).
5. Считает профит и пересчитывает его при изменении цен в price_rules.json.
6. Имеет встроенный просто телеграм клиент который вы можете настроить за 5 минут и получать статистику и управлять бото из телеграм чата.

Hi, if you're reading this, you're the best.
Basically, this bot was created back in the winter of 2024–2025 to be rented on clore.ai. It went through many iterations, from simple Bash code to a Python bot.

You can modify this code however you like, or you can follow the releases and use it for yourself.

Currently, the bot can:
1. Rent servers using USD, USDT, or Clore.
2. Check for active servers while running.
3. Configure various mining tasks on your servers depending on the hardware type.
4. Maintain an SQL database that calculates server profit based on different combinations of profiles (CPU and GPU).
5. Calculate profit and recalculate it when prices change in price_rules.json.
6. It has a built-in Telegram client that you can set up in 5 minutes to receive statistics and manage the bot from a Telegram chat.

# CloreRentBot v7.1.21 — инструкция от А до Я

Бот арендует серверы на Clore, ставит майнеры из `miners.json`, ведёт мониторинг SSH/процессов, бенчмарк хешрейта, расчёт доходности и автоматическую отмену невыгодных/сломанных заказов.

## 1. Что лежит в папке бота

Минимальный набор файлов:

```text
clore_botV7.1.21.py      # основной бот
config.json              # API-ключи, SSH-пароль, лимиты, режим market/whitelist
miners.json              # профили CPU/GPU майнеров и правила назначения по железу
price_rules.json         # лимиты цены и стоимость хешрейта по профилям
```

Файлы состояния создаются автоматически:

```text
active_orders.json             # соответствие server_id -> order_id
active_assignments.json        # какие CPU/GPU профили применены к активным серверам
hashrate_benchmarks.sqlite3    # база бенчмарков и доходности
rent_prices.json               # сохранённые цены аренды
rent_skip.json                 # временный skip серверов после ошибок create_order
server_errors.json             # счётчик проблем по серверам
blacklist.json                 # локальный blacklist
clore_rent.log                 # лог бота
problem_servers.log            # история проблемных серверов
release_requests.log           # история запросов на отмену
watchdog_state.json            # состояние watchdog по GPU-хешрейту
```

## 2. Установка зависимостей на Linux

Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git screen

cd /home/izao/clore_bot   # замени путь на свою папку
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip
python -m pip install requests paramiko colorama
```

Проверка:

```bash
python - <<'PY'
import requests, paramiko, colorama
print('OK')
PY
```

## 3. Установка зависимостей на Windows

PowerShell:

```powershell
cd C:\clore_bot   # замени путь на свою папку
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -U pip
python -m pip install requests paramiko colorama
```

Если PowerShell не разрешает запуск activate-скрипта:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
.\.venv\Scripts\Activate.ps1
```

Проверка:

```powershell
python - <<'PY'
import requests, paramiko, colorama
print('OK')
PY
```

## 4. Настройка `config.json`

Основные поля:

```json
{
  "settings": {
    "AUTH_KEY": "CLORE_API_KEY",
    "MODE": "market",
    "TIME": 2,
    "MONITOR_INTERVAL": 1800,
    "SSH": {
      "USER": "root",
      "PASSWORD": "your_ssh_password"
    },
    "MARKET": {
      "MAX_RENTED": 4,
      "USE_PRICE_RULES": true,
      "ALLOW_LEGACY_FALLBACK": false,
      "PRICE_RULES_FILE": "price_rules.json",
      "SELECT": "max_margin",
      "SKIP_KNOWN_BENCHMARK_LOG_INTERVAL_SEC": 900
    },
    "BENCHMARK": {
      "ENABLED": true,
      "DB_FILE": "hashrate_benchmarks.sqlite3",
      "MIN_PROFIT_PERCENT": 10,
      "CANCEL_UNPROFITABLE": true,
      "BENCHMARK_ON_RENT": true,
      "ASYNC_ON_RENT": true,
      "POST_RENT_DELAY_SEC": 30,
      "SSH_READY_TIMEOUT_SEC": 360,
      "SSH_READY_RETRY_SEC": 10,
      "WAIT_MINERS_READY": true,
      "MINERS_READY_TIMEOUT_SEC": 150,
      "MINERS_READY_RETRY_SEC": 10,
      "WARMUP_SEC": 180,
      "MEASURE_SEC": 120,
      "SAMPLE_INTERVAL_SEC": 20,
      "MAX_ZERO_SAMPLES": 5,
      "SKIP_MONITOR_WHILE_BENCHMARKING": true,
      "REBENCH_LOSS_GUARD_ENABLED": true,
      "REBENCH_LOSS_GUARD_PERCENT": -25
    },
    "TELEGRAM": {
      "ENABLED": false,
      "BOT_TOKEN": "",
      "ALLOWED_CHAT_IDS": []
    }
  },
  "servers": {}
}
```

Критичные настройки:

- `MODE=market` — бот сам выбирает серверы с рынка.
- `MODE=whitelist` — бот арендует только серверы из `servers`.
- `MARKET.MAX_RENTED` — максимум активных заказов.
- `BENCHMARK.MIN_PROFIT_PERCENT` — минимальная прибыльность. Например `10` значит не ниже +10% к аренде.
- `BENCHMARK.REBENCH_LOSS_GUARD_PERCENT=-25` — если сервер уже давал сильный минус, бот не будет брать его снова без ручного сброса старых строк.

## 5. Настройка `miners.json`

Логика:

1. Бот читает железо сервера.
2. По `assignments.cpu` и `assignments.gpu` выбирает профили.
3. Из выбранных профилей генерирует скрипты `/root/start_cpu.sh` и `/root/start_gpu.sh`.
4. После запуска проверяет процессы/screen и парсит хешрейт.

Упрощённый пример структуры:

```json
{
  "active_profile": "dual",
  "auto_apply_on_change": true,
  "assignments": {
    "cpu": [
      {
        "name": "amd_cpu",
        "match": { "vendors": ["amd"] },
        "profile": "cpu_amd_core"
      },
      {
        "name": "default_cpu",
        "default": true,
        "profile": "dual"
      }
    ],
    "gpu": [
      {
        "name": "default_gpu",
        "default": true,
        "profile": "gpu_prl_srb"
      }
    ]
  },
  "profiles": {
    "dual": {
      "cpu": {
        "enabled": true,
        "screen": "cpu-miner",
        "process_match": "SRBMiner-MULTI",
        "install": {
          "url": "https://example.com/cpu.tar.gz",
          "archive": "/root/cpu.tar.gz",
          "extract_to": "/root/cpu_miner",
          "bin_path": "/root/cpu_miner/SRBMiner-MULTI"
        },
        "hashrate_parser": {
          "enabled": true,
          "source": "log_tail",
          "log_file": "/root/cpu_miner.log",
          "parser": "regex",
          "regex": "Total:\\s*([0-9.]+)\\s*H/s",
          "unit": "H/s",
          "aggregate": "last"
        }
      }
    },
    "gpu_prl_srb": {
      "gpu": {
        "enabled": true,
        "screen": "gpu-miner",
        "process_match": "SRBMiner-MULTI",
        "install": {
          "url": "https://example.com/gpu.tar.gz",
          "archive": "/root/gpu.tar.gz",
          "extract_to": "/root/gpu_miner",
          "bin_path": "/root/gpu_miner/SRBMiner-MULTI"
        },
        "start_args": "--your --gpu --args",
        "screen_log": true,
        "hashrate_parser": {
          "enabled": true,
          "source": "screen_hardcopy",
          "parser": "screen_hashrate_table",
          "screen": "gpu-miner",
          "aggregate": "sum"
        },
        "watchdog": {
          "enabled": true,
          "type": "screen_hashrate_table",
          "screen": "gpu-miner",
          "fail_checks": 2,
          "grace_sec": 180,
          "restart_cooldown_sec": 300,
          "min_hashrate": 0.000001,
          "require_all_gpus": true
        }
      }
    }
  }
}
```

## 6. Настройка `price_rules.json`

Файл отвечает за две разные вещи:

1. Максимальная допустимая цена аренды по железу.
2. Доходность хешрейта для CPU/GPU профилей.

Пример:

```json
{
  "behavior": {
    "allow_unknown_gpu": false,
    "allow_unknown_cpu": true
  },
  "defaults": {
    "cpu_usd_value_per_day": 0.01,
    "gpu_max_usd_per_gpu_day": null
  },
  "gpu": {
    "RTX 3060": { "max_usd_per_gpu_day": 0.35 },
    "RTX 3070": { "max_usd_per_gpu_day": 0.45 }
  },
  "cpu": {
    "Ryzen": { "usd_value_per_day": 0.05 },
    "EPYC": { "usd_value_per_day": 0.10 }
  },
  "hashrate_value": {
    "cpu": {
      "dual": { "usd_per_kh_day": 0.015 },
      "cpu_amd_core": { "usd_per_kh_day": 0.012 }
    },
    "gpu": {
      "gpu_prl_srb": { "usd_per_th_day": 0.01 }
    }
  }
}
```

Важно: имена внутри `hashrate_value.cpu` и `hashrate_value.gpu` должны совпадать с именами профилей из `miners.json`.

## 7. Первый запуск

Linux:

```bash
cd /home/izao/clore_bot
source .venv/bin/activate
python3 clore_botV7.1.21.py
```

Windows:

```powershell
cd C:\clore_bot
.\.venv\Scripts\Activate.ps1
python clore_botV7.1.21.py
```

После запуска проверь лог:

```bash
tail -f clore_rent.log
```

В норме должны появиться строки вида:

```text
Config loaded. MODE=market
Miner profile loaded: ...
Loaded price rules from price_rules.json
Monitoring thread started
Interactive command loop started
```

## 8. Запуск в `screen` на Linux

```bash
cd /home/izao/clore_bot
screen -S clorebot
source .venv/bin/activate
python3 clore_botV7.1.21.py
```

Отключиться от screen:

```text
Ctrl+A, затем D
```

Вернуться:

```bash
screen -r clorebot
```

## 9. Запуск через systemd

Создать сервис:

```bash
sudo nano /etc/systemd/system/clore-bot.service
```

Пример:

```ini
[Unit]
Description=CloreRentBot
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/izao/clore_bot
ExecStart=/home/izao/clore_bot/.venv/bin/python /home/izao/clore_bot/clore_botV7.1.21.py
Restart=always
RestartSec=10
User=izao

[Install]
WantedBy=multi-user.target
```

Запуск:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now clore-bot
sudo journalctl -u clore-bot -f
```

Перезапуск после обновления:

```bash
sudo systemctl restart clore-bot
```

## 10. Команды бота

Команды работают в интерактивной консоли и в Telegram. В Telegram добавляется `/` перед командой.

```text
help                      показать команды
status                    краткий статус
status_pro                подробный статус по профилям/доходности
balance                   балансы Clore
profile_reload            перечитать miners.json + price_rules.json и пересчитать экономику
price_reload              перечитать price_rules.json, пересчитать БД, отменить активные минусовые серверы
reload <profile>          пометить benchmark-строки профиля stale и перебенчить активные серверы этого профиля
restart <profile>         переустановить/перезапустить активные серверы профиля и перебенчить
rebench <profile>         перебенчить активные серверы профиля без переустановки
reload_all                пометить все benchmark-строки stale и перебенчить активные серверы
reload_profile <cpu|gpu> <profile>
                          принудительный вариант, если имя профиля неоднозначно
bench_server <server_id>  принудительный benchmark одного активного сервера
drop_unprofitable         отменить активные серверы ниже MIN_PROFIT_PERCENT
```

Примеры:

```text
reload gpu_prl_srb
restart gpu_prl_srb
rebench gpu_prl_srb
reload cpu_amd_core
restart dual
bench_server 103004
drop_unprofitable
```

Telegram-примеры:

```text
/status_pro
/reload gpu_prl_srb
/restart cpu_amd_core
/rebench dual
/bench_server 103004
```

## 11. Как работает согласование monitor и benchmark

В версии v7.1.21 добавлена синхронизация локального состояния с `/my_orders`:

- если локально нет `order_id`, бот перечитывает `/my_orders` и ищет актуальный заказ по `server_id`;
- если cancel вернул `order_not_active`, `you_dont_own_this_order`, `order_not_found` или похожую ошибку, бот снова перечитывает `/my_orders`;
- если найден новый актуальный `order_id`, бот повторяет отмену уже с ним;
- если сервер больше не найден в `/my_orders`, бот удаляет его из `active_orders.json` и `active_servers`;
- при удалении активного сервера выставляется cancel-флаг для benchmark-потока, чтобы он не сохранял устаревший результат.

Это закрывает типовые ситуации, когда монитор отменил сервер, а benchmark ещё ждёт SSH/майнер, или наоборот benchmark отменил заказ, а монитор продолжает считать сервер активным.

## 12. Типовой сценарий обновления

Остановить старый процесс:

```bash
screen -r clorebot
# Ctrl+C
```

Или через systemd:

```bash
sudo systemctl stop clore-bot
```

Сделать копию старого файла:

```bash
cp clore_botV7.1.20.py clore_botV7.1.20.py.bak
```

Положить новый файл:

```bash
cp clore_botV7.1.21.py /home/izao/clore_bot/
```

Запустить:

```bash
source .venv/bin/activate
python3 clore_botV7.1.21.py
```

Если используется systemd, поменять `ExecStart` на новый файл и выполнить:

```bash
sudo systemctl daemon-reload
sudo systemctl restart clore-bot
sudo journalctl -u clore-bot -f
```

## 13. Что смотреть при проблемах

Последние логи:

```bash
tail -n 200 clore_rent.log
```

Ошибки по серверам:

```bash
cat server_errors.json
cat problem_servers.log
cat release_requests.log
```

Активные локальные заказы:

```bash
cat active_orders.json
```

Состояние benchmark DB:

```bash
sqlite3 hashrate_benchmarks.sqlite3 "SELECT server_id,cpu_profile,gpu_profile,status,profit_percent,rent_usd_day,last_bench_at FROM hashrate_benchmarks ORDER BY updated_at DESC LIMIT 30;"
```

## 14. Рекомендованные безопасные значения

Для рынка:

```json
"MARKET": {
  "USE_PRICE_RULES": true,
  "ALLOW_LEGACY_FALLBACK": false,
  "SELECT": "max_margin",
  "MAX_RENTED": 4
}
```

Для benchmark:

```json
"BENCHMARK": {
  "MIN_PROFIT_PERCENT": 10,
  "CANCEL_UNPROFITABLE": true,
  "SKIP_MONITOR_WHILE_BENCHMARKING": true,
  "REBENCH_LOSS_GUARD_ENABLED": true,
  "REBENCH_LOSS_GUARD_PERCENT": -25,
  "MAX_ZERO_SAMPLES": 5
}
```

## 15. Краткий чеклист перед боевым запуском

1. `config.json` содержит актуальный `AUTH_KEY` и SSH-пароль.
2. `miners.json` валидный JSON-объект, не массив.
3. Имена профилей в `price_rules.json.hashrate_value` совпадают с `miners.json.profiles`.
4. В `MARKET` включён `USE_PRICE_RULES=true`.
5. `ALLOW_LEGACY_FALLBACK=false`, если не нужен старый режим выбора.
6. Первый запуск сделан вручную в `screen`, чтобы увидеть ошибки сразу.
7. После стабильной проверки бот переведён в systemd.


# CloreRentBot v7.1.21 — Complete Setup Guide

The bot rents servers on Clore, installs miners defined in `miners.json`, monitors SSH/processes, benchmarks hashrate, calculates profitability, and automatically cancels unprofitable or broken orders.

## 1. What is inside the bot folder

Minimum required files:

```text
clore_botV7.1.21.py      # main bot
config.json              # API keys, SSH password, limits, market/whitelist mode
miners.json              # CPU/GPU miner profiles and hardware assignment rules
price_rules.json         # price limits and hashrate value by profile
```

State files are created automatically:

```text
active_orders.json             # server_id -> order_id mapping
active_assignments.json        # CPU/GPU profiles applied to active servers
hashrate_benchmarks.sqlite3    # benchmark and profitability database
rent_prices.json               # saved rental prices
rent_skip.json                 # temporary server skip list after create_order errors
server_errors.json             # server problem counters
blacklist.json                 # local blacklist
clore_rent.log                 # bot log
problem_servers.log            # problem server history
release_requests.log           # cancellation request history
watchdog_state.json            # GPU hashrate watchdog state
```

## 2. Installing dependencies on Linux

Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git screen

cd /home/izao/clore_bot   # replace this path with your bot folder
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip
python -m pip install requests paramiko colorama
```

Check:

```bash
python - <<'PY'
import requests, paramiko, colorama
print('OK')
PY
```

## 3. Installing dependencies on Windows

PowerShell:

```powershell
cd C:\clore_bot   # replace this path with your bot folder
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -U pip
python -m pip install requests paramiko colorama
```

If PowerShell does not allow running the activation script:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
.\.venv\Scripts\Activate.ps1
```

Check:

```powershell
python - <<'PY'
import requests, paramiko, colorama
print('OK')
PY
```

## 4. Configuring `config.json`

Main fields:

```json
{
  "settings": {
    "AUTH_KEY": "CLORE_API_KEY",
    "MODE": "market",
    "TIME": 2,
    "MONITOR_INTERVAL": 1800,
    "SSH": {
      "USER": "root",
      "PASSWORD": "your_ssh_password"
    },
    "MARKET": {
      "MAX_RENTED": 4,
      "USE_PRICE_RULES": true,
      "ALLOW_LEGACY_FALLBACK": false,
      "PRICE_RULES_FILE": "price_rules.json",
      "SELECT": "max_margin",
      "SKIP_KNOWN_BENCHMARK_LOG_INTERVAL_SEC": 900
    },
    "BENCHMARK": {
      "ENABLED": true,
      "DB_FILE": "hashrate_benchmarks.sqlite3",
      "MIN_PROFIT_PERCENT": 10,
      "CANCEL_UNPROFITABLE": true,
      "BENCHMARK_ON_RENT": true,
      "ASYNC_ON_RENT": true,
      "POST_RENT_DELAY_SEC": 30,
      "SSH_READY_TIMEOUT_SEC": 360,
      "SSH_READY_RETRY_SEC": 10,
      "WAIT_MINERS_READY": true,
      "MINERS_READY_TIMEOUT_SEC": 150,
      "MINERS_READY_RETRY_SEC": 10,
      "WARMUP_SEC": 180,
      "MEASURE_SEC": 120,
      "SAMPLE_INTERVAL_SEC": 20,
      "MAX_ZERO_SAMPLES": 5,
      "SKIP_MONITOR_WHILE_BENCHMARKING": true,
      "REBENCH_LOSS_GUARD_ENABLED": true,
      "REBENCH_LOSS_GUARD_PERCENT": -25
    },
    "TELEGRAM": {
      "ENABLED": false,
      "BOT_TOKEN": "",
      "ALLOWED_CHAT_IDS": []
    }
  },
  "servers": {}
}
```

Critical settings:

- `MODE=market` — the bot selects servers from the market automatically.
- `MODE=whitelist` — the bot rents only servers listed in `servers`.
- `MARKET.MAX_RENTED` — maximum number of active orders.
- `BENCHMARK.MIN_PROFIT_PERCENT` — minimum profitability. For example, `10` means not lower than +10% over the rental cost.
- `BENCHMARK.REBENCH_LOSS_GUARD_PERCENT=-25` — if a server previously produced a large loss, the bot will not rent it again unless the old benchmark rows are reset manually.

## 5. Configuring `miners.json`

Logic:

1. The bot reads the server hardware.
2. It selects profiles using `assignments.cpu` and `assignments.gpu`.
3. It generates `/root/start_cpu.sh` and `/root/start_gpu.sh` from the selected profiles.
4. After startup, it checks processes/screens and parses hashrate.

Simplified structure example:

```json
{
  "active_profile": "dual",
  "auto_apply_on_change": true,
  "assignments": {
    "cpu": [
      {
        "name": "amd_cpu",
        "match": { "vendors": ["amd"] },
        "profile": "cpu_amd_core"
      },
      {
        "name": "default_cpu",
        "default": true,
        "profile": "dual"
      }
    ],
    "gpu": [
      {
        "name": "default_gpu",
        "default": true,
        "profile": "gpu_prl_srb"
      }
    ]
  },
  "profiles": {
    "dual": {
      "cpu": {
        "enabled": true,
        "screen": "cpu-miner",
        "process_match": "SRBMiner-MULTI",
        "install": {
          "url": "https://example.com/cpu.tar.gz",
          "archive": "/root/cpu.tar.gz",
          "extract_to": "/root/cpu_miner",
          "bin_path": "/root/cpu_miner/SRBMiner-MULTI"
        },
        "hashrate_parser": {
          "enabled": true,
          "source": "log_tail",
          "log_file": "/root/cpu_miner.log",
          "parser": "regex",
          "regex": "Total:\\s*([0-9.]+)\\s*H/s",
          "unit": "H/s",
          "aggregate": "last"
        }
      }
    },
    "gpu_prl_srb": {
      "gpu": {
        "enabled": true,
        "screen": "gpu-miner",
        "process_match": "SRBMiner-MULTI",
        "install": {
          "url": "https://example.com/gpu.tar.gz",
          "archive": "/root/gpu.tar.gz",
          "extract_to": "/root/gpu_miner",
          "bin_path": "/root/gpu_miner/SRBMiner-MULTI"
        },
        "start_args": "--your --gpu --args",
        "screen_log": true,
        "hashrate_parser": {
          "enabled": true,
          "source": "screen_hardcopy",
          "parser": "screen_hashrate_table",
          "screen": "gpu-miner",
          "aggregate": "sum"
        },
        "watchdog": {
          "enabled": true,
          "type": "screen_hashrate_table",
          "screen": "gpu-miner",
          "fail_checks": 2,
          "grace_sec": 180,
          "restart_cooldown_sec": 300,
          "min_hashrate": 0.000001,
          "require_all_gpus": true
        }
      }
    }
  }
}
```

## 6. Configuring `price_rules.json`

This file is responsible for two separate things:

1. Maximum allowed rental price by hardware.
2. Hashrate profitability for CPU/GPU profiles.

Example:

```json
{
  "behavior": {
    "allow_unknown_gpu": false,
    "allow_unknown_cpu": true
  },
  "defaults": {
    "cpu_usd_value_per_day": 0.01,
    "gpu_max_usd_per_gpu_day": null
  },
  "gpu": {
    "RTX 3060": { "max_usd_per_gpu_day": 0.35 },
    "RTX 3070": { "max_usd_per_gpu_day": 0.45 }
  },
  "cpu": {
    "Ryzen": { "usd_value_per_day": 0.05 },
    "EPYC": { "usd_value_per_day": 0.10 }
  },
  "hashrate_value": {
    "cpu": {
      "dual": { "usd_per_kh_day": 0.015 },
      "cpu_amd_core": { "usd_per_kh_day": 0.012 }
    },
    "gpu": {
      "gpu_prl_srb": { "usd_per_th_day": 0.01 }
    }
  }
}
```

Important: profile names inside `hashrate_value.cpu` and `hashrate_value.gpu` must match profile names from `miners.json`.

## 7. First launch

Linux:

```bash
cd /home/izao/clore_bot
source .venv/bin/activate
python3 clore_botV7.1.21.py
```

Windows:

```powershell
cd C:\clore_bot
.\.venv\Scripts\Activate.ps1
python clore_botV7.1.21.py
```

After startup, check the log:

```bash
tail -f clore_rent.log
```

Normally, you should see lines similar to:

```text
Config loaded. MODE=market
Miner profile loaded: ...
Loaded price rules from price_rules.json
Monitoring thread started
Interactive command loop started
```

## 8. Running in `screen` on Linux

```bash
cd /home/izao/clore_bot
screen -S clorebot
source .venv/bin/activate
python3 clore_botV7.1.21.py
```

Detach from screen:

```text
Ctrl+A, then D
```

Reattach:

```bash
screen -r clorebot
```

## 9. Running through systemd

Create the service:

```bash
sudo nano /etc/systemd/system/clore-bot.service
```

Example:

```ini
[Unit]
Description=CloreRentBot
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/izao/clore_bot
ExecStart=/home/izao/clore_bot/.venv/bin/python /home/izao/clore_bot/clore_botV7.1.21.py
Restart=always
RestartSec=10
User=izao

[Install]
WantedBy=multi-user.target
```

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now clore-bot
sudo journalctl -u clore-bot -f
```

Restart after an update:

```bash
sudo systemctl restart clore-bot
```

## 10. Bot commands

Commands work in the interactive console and in Telegram. In Telegram, add `/` before the command.

```text
help                      show commands
status                    short status
status_pro                detailed status by profile/profitability
balance                   Clore balances
profile_reload            reread miners.json + price_rules.json and recalculate economics
price_reload              reread price_rules.json, recalculate DB, cancel active negative-profit servers
reload <profile>          mark benchmark rows for the profile as stale and re-benchmark active servers for this profile
restart <profile>         reinstall/restart active servers for the profile and re-benchmark them
rebench <profile>         re-benchmark active servers for the profile without reinstalling
reload_all                mark all benchmark rows as stale and re-benchmark active servers
reload_profile <cpu|gpu> <profile>
                          forced variant when the profile name is ambiguous
bench_server <server_id>  force benchmark for one active server
drop_unprofitable         cancel active servers below MIN_PROFIT_PERCENT
```

Examples:

```text
reload gpu_prl_srb
restart gpu_prl_srb
rebench gpu_prl_srb
reload cpu_amd_core
restart dual
bench_server 103004
drop_unprofitable
```

Telegram examples:

```text
/status_pro
/reload gpu_prl_srb
/restart cpu_amd_core
/rebench dual
/bench_server 103004
```

## 11. How monitor and benchmark synchronization works

Version v7.1.21 adds synchronization of local state with `/my_orders`:

- if there is no local `order_id`, the bot rereads `/my_orders` and searches for the current order by `server_id`;
- if cancellation returns `order_not_active`, `you_dont_own_this_order`, `order_not_found`, or a similar error, the bot rereads `/my_orders` again;
- if a new current `order_id` is found, the bot retries cancellation using that order ID;
- if the server is no longer found in `/my_orders`, the bot removes it from `active_orders.json` and `active_servers`;
- when an active server is removed, a cancellation flag is set for the benchmark thread so it does not save a stale result.

This covers common situations where the monitor cancels a server while benchmark is still waiting for SSH/miner readiness, or the reverse: benchmark cancels the order while the monitor still treats the server as active.

## 12. Typical update procedure

Stop the old process:

```bash
screen -r clorebot
# Ctrl+C
```

Or through systemd:

```bash
sudo systemctl stop clore-bot
```

Create a backup of the old file:

```bash
cp clore_botV7.1.20.py clore_botV7.1.20.py.bak
```

Place the new file:

```bash
cp clore_botV7.1.21.py /home/izao/clore_bot/
```

Start it:

```bash
source .venv/bin/activate
python3 clore_botV7.1.21.py
```

If systemd is used, update `ExecStart` to the new file and run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart clore-bot
sudo journalctl -u clore-bot -f
```

## 13. What to check when there are problems

Latest logs:

```bash
tail -n 200 clore_rent.log
```

Server errors:

```bash
cat server_errors.json
cat problem_servers.log
cat release_requests.log
```

Active local orders:

```bash
cat active_orders.json
```

Benchmark DB state:

```bash
sqlite3 hashrate_benchmarks.sqlite3 "SELECT server_id,cpu_profile,gpu_profile,status,profit_percent,rent_usd_day,last_bench_at FROM hashrate_benchmarks ORDER BY updated_at DESC LIMIT 30;"
```

## 14. Recommended safe values

For the market:

```json
"MARKET": {
  "USE_PRICE_RULES": true,
  "ALLOW_LEGACY_FALLBACK": false,
  "SELECT": "max_margin",
  "MAX_RENTED": 4
}
```

For benchmark:

```json
"BENCHMARK": {
  "MIN_PROFIT_PERCENT": 10,
  "CANCEL_UNPROFITABLE": true,
  "SKIP_MONITOR_WHILE_BENCHMARKING": true,
  "REBENCH_LOSS_GUARD_ENABLED": true,
  "REBENCH_LOSS_GUARD_PERCENT": -25,
  "MAX_ZERO_SAMPLES": 5
}
```

## 15. Short checklist before production launch

1. `config.json` contains the current `AUTH_KEY` and SSH password.
2. `miners.json` is a valid JSON object, not an array.
3. Profile names in `price_rules.json.hashrate_value` match `miners.json.profiles`.
4. `USE_PRICE_RULES=true` is enabled in `MARKET`.
5. `ALLOW_LEGACY_FALLBACK=false` if the old selection mode is not needed.
6. The first launch was done manually in `screen` to catch errors immediately.
7. After a stable check, the bot was moved to systemd.
