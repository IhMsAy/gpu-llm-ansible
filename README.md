# NVIDIA GPU + Ollama: защищённое развёртывание через Ansible

Playbook настраивает один или несколько серверов Ubuntu с NVIDIA GPU и поднимает
LLM API на Ollama. Порт Ollama не доступен извне: запросы принимает Nginx по
HTTPS с Basic Auth, а UFW разрешает подключение только из доверенных сетей.

## Что разворачивается

- рекомендуемый NVIDIA-драйвер с автоматической перезагрузкой при установке;
- Docker Engine из официального репозитория и NVIDIA Container Toolkit;
- Ollama с GPU, постоянным томом моделей и политикой `unless-stopped`;
- Nginx с TLS, Basic Auth, rate limit и длинными тайм-аутами для генерации;
- UFW: входящие подключения запрещены, кроме SSH и API из заданных CIDR;
- NVIDIA DCGM Exporter: температура, загрузка, память, питание и ошибки GPU;
- Prometheus с хранением истории 15 дней и базовыми GPU alert rules;
- автоматические проверки `nvidia-smi`, GPU в контейнере, Ollama API,
  фактического inference, защищённого Nginx и метрик DCGM.

Все контейнерные порты (`11434`, `9090`, `9400`) привязаны только к
`127.0.0.1`. Наружу слушает только Nginx на `8443`.

## Требования

Управляющая машина — Linux/macOS/WSL с Ansible Core 2.15+; целевой сервер —
Ubuntu 22.04 или новее, доступ по SSH с `sudo`, NVIDIA GPU и интернет для APT,
Docker Registry, NVCR и скачивания модели.

> При включённом UEFI Secure Boot Ubuntu может запросить регистрацию MOK при
> загрузке. Полностью автоматизировать физическое подтверждение невозможно:
> завершите MOK enrollment в консоли сервера и повторите playbook.

## Быстрый запуск

1. Установите коллекцию:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. Подготовьте inventory:

   ```bash
   cp inventory.example.yml inventory.yml
   ```

   Замените `192.0.2.10`, пользователя и при необходимости путь к SSH-ключу.

3. В `group_vars/all.yml` обязательно замените `trusted_api_cidrs` на адрес
   офиса, VPN или bastion-сети. Не используйте `0.0.0.0/0` для production.

4. Создайте зашифрованный секрет:

   ```bash
   cp vault.example.yml vault.yml
   # Сначала задайте в vault.yml длинный случайный пароль:
   ansible-vault encrypt vault.yml
   ```

5. Проверьте доступ и выполните развёртывание:

   ```bash
   ansible-playbook site.yml --syntax-check -e @vault.yml --ask-vault-pass
   ansible-playbook site.yml -e @vault.yml --ask-vault-pass
   ```

Первый запуск может занять много времени: сервер перезагрузится после установки
драйвера, а Ollama скачает модель. Ansible дождётся возврата сервера и выполнит
inference-проверку. Повторный запуск идемпотентен и модель повторно не скачивает.

## Смена модели одной переменной

Измените только `ollama_model` в `group_vars/all.yml`, например:

```yaml
ollama_model: "llama3.2:3b"
```

Затем снова запустите `ansible-playbook`. До выбора модели сопоставьте её размер
с VRAM. Для первого теста разумно взять модель 3B–7B в квантизации по умолчанию.

## API-запрос

При self-signed сертификате нужен `-k`; с сертификатом вашей CA уберите этот
флаг. Адрес клиента должен входить в `trusted_api_cidrs`.

```bash
curl -k --user 'llm-api:ВАШ_ПАРОЛЬ' \
  https://GPU_SERVER:8443/api/generate \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen2.5:7b",
    "prompt": "Кратко объясни, что такое RAG",
    "stream": false
  }'
```

Ollama также предоставляет OpenAI-совместимый endpoint:

```bash
curl -k --user 'llm-api:ВАШ_ПАРОЛЬ' \
  https://GPU_SERVER:8443/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen2.5:7b",
    "messages": [{"role": "user", "content": "Ответь одним словом: работает?"}],
    "stream": false
  }'
```

## Мониторинг

- Prometheus UI: `https://GPU_SERVER:8443/prometheus/`
- сырые GPU-метрики: `https://GPU_SERVER:8443/metrics/gpu`

Оба URL защищены тем же логином и паролем. Prometheus собирает метрики каждые
15 секунд. Включены правила высокой температуры (>85 °C), заполнения VRAM
(>95%) и недоступности exporter. Правила видны в Prometheus; для реальных
уведомлений добавьте Alertmanager и маршрут в почту/Slack/PagerDuty.

Полезные запросы PromQL:

```promql
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_GPU_TEMP_CELSIUS
100 * DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL
```

## TLS и production-рекомендации

По умолчанию playbook создаёт self-signed сертификат. Для сертификата вашей CA
задайте оба controller-side пути:

```yaml
nginx_tls_cert_src: "files/llm-api.crt"
nginx_tls_key_src: "files/llm-api.key"
```

Для production также рекомендуется:

- доступ к `8443` только через корпоративный VPN/private subnet;
- секреты только в Ansible Vault или внешнем secret manager;
- закрепить версии `ollama_image` и `prometheus_image` вместо `latest` после
  приёмочных испытаний;
- включить резервное копирование Docker volume `ollama_data` только если нужно
  сохранять локальные модели (обычно быстрее скачать их заново);
- подключить Alertmanager и Grafana; централизованно собирать логи Nginx/Ollama;
- для нескольких GPU-серверов вынести Nginx/API gateway, Prometheus и model
  routing на отдельные узлы, а inference workers держать в закрытой подсети;
- для высокой нагрузки рассмотреть vLLM: он даёт continuous batching и более
  предсказуемый OpenAI API, тогда как Ollama проще для эксплуатации одного узла.

## Проверка и диагностика на сервере

```bash
sudo nvidia-smi
sudo docker compose -f /opt/gpu-llm/compose.yml ps
sudo docker logs ollama --tail 100
curl -s http://127.0.0.1:9400/metrics | grep DCGM_FI_DEV_GPU
sudo nginx -t
sudo ufw status verbose
```

Повторный запуск playbook является основной процедурой восстановления
конфигурации. Данные моделей и Prometheus находятся в именованных Docker
volumes и при обычном `docker compose up -d` не удаляются.

## Важные замечания

- Добавление пользователя в группу `docker` эквивалентно root-доступу; список
  `docker_users` по умолчанию пуст.
- UFW не всегда фильтрует порты, опубликованные Docker. Здесь защита не зависит
  от этого: внутренние сервисы публикуются исключительно на loopback.
- Playbook не удаляет старые модели автоматически, чтобы не потерять данные.
- Без физического GPU можно выполнить `--syntax-check`, но задачи `nvidia-smi`,
  GPU-контейнера, DCGM и inference ожидаемо требуют реальный сервер.

