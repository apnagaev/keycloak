# Keycloak cluster (Ansible + Docker Compose)

Отказоустойчивый Keycloak из двух узлов, разворачиваемый одним плейбуком Ansible.

- Keycloak в Docker (`quay.io/keycloak/keycloak:latest`), `network_mode: host`
- Кластер на встроенном Infinispan: узлы находят друг друга через таблицу в БД (`jdbc-ping`),
  межузловой трафик шифруется mTLS (по умолчанию в Keycloak 26.x)
- Внешняя БД MariaDB
- Reverse proxy с vIP на тех же хостах, TLS терминируется на прокси, Keycloak слушает HTTP
- Kerberos/SPNEGO через keytab, доверенный корпоративный CA
- Firewall на nftables: порты Keycloak доступны только прокси, мониторингу и соседнему узлу

```
             клиенты
                │ HTTPS
          ┌─────▼─────┐   vIP
          │  reverse  │
          │   proxy   │  (на обоих хостах)
          └──┬─────┬──┘
     :28080  │     │  :28080
   ┌─────────▼┐   ┌▼─────────┐
   │  node1   │◄─►│  node2   │  JGroups :7800 / :57800 (mTLS)
   │ keycloak │   │ keycloak │
   └─────┬────┘   └────┬─────┘
         └──────┬──────┘
           ┌────▼────┐
           │ MariaDB │
           └─────────┘
```

## Структура

```
install_keycloak.yaml            # плейбук: firewall + деплой + rolling restart + очистка
inventory.example.yml            # пример inventory (скопировать в inventory.yml)
vars/keycloak.example.yml        # домен, БД, адреса, имена файлов (скопировать в vars/keycloak.yml)
files/keycloak/
  keycloak.yaml.j2               # шаблон docker-compose.yaml
  keycloak.nft.j2                # шаблон правил nftables
  keycloak.env.example           # логины и пароли БД и первого админа (скопировать в keycloak.env)
  keycloak.env                   # НЕ в репозитории
  ca.crt                         # НЕ в репозитории — CA-сертификат для truststore (имя — kc_ca_cert)
  keycloak.keytab                # НЕ в репозитории — Kerberos keytab (имя — kc_keytab)
```

## Требования

- Хосты Debian/Ubuntu с Docker Engine и плагином `docker compose`
- Ansible на управляющей машине (используются только встроенные модули, коллекции не нужны)
- Доступная база MariaDB с пользователем и базой `keycloak`
- Keytab для принципала `HTTP/<hostname>@<REALM>` (если нужен Kerberos)

## Настройка

### 1. Inventory

```bash
cp inventory.example.yml inventory.yml
```
Для каждого хоста задайте `ansible_host`, `node` (1 или 2 — порядок перезапуска) и
`nodeip` (адрес, на котором узел общается с кластером). Группа хостов должна называться `docker`.

### 2. Настройки инсталляции

```bash
cp vars/keycloak.example.yml vars/keycloak.yml
cp files/keycloak/keycloak.env.example files/keycloak/keycloak.env
```

Оба файла в `.gitignore`: код можно обновлять из git (`git pull`), не трогая свои значения.

`vars/keycloak.yml`:

| Переменная | Назначение |
|---|---|
| `kc_hostname` | внешний URL, например `https://sso.example.org` |
| `kc_db_url` | подключение к MariaDB (`jdbc:mariadb://…/keycloak`) |
| `dns_server`, `kc_timezone` | DNS и часовой пояс контейнера |
| `kc_nodes` | IP узлов Keycloak — им открыты кластерные порты |
| `kc_proxies` | IP reverse proxy, включая vIP — им открыт HTTP-порт; они же (+ `172.16.0.0/12`) в `KC_PROXY_TRUSTED_ADDRESSES` |
| `kc_monitoring` | IP системы мониторинга — ей открыт порт `9000` |
| `kc_ca_cert` | имя файла CA в `files/keycloak/` (`""` — без своего CA) |
| `kc_keytab` | имя keytab в `files/keycloak/` (`""` — без Kerberos) |

`files/keycloak/keycloak.env`: `KC_DB_USERNAME`, `KC_DB_PASSWORD`, `KC_BOOTSTRAP_ADMIN_USERNAME`, `KC_BOOTSTRAP_ADMIN_PASSWORD` (первый админ создаётся только на пустой БД). Файл копируется на узлы с правами `0600`.

В блоке `vars` плейбука остался `kc_http_port` (по умолчанию `28080`).

### 3. Файлы, которых нет в репозитории

Положите в `files/keycloak/`:
- CA, которому должен доверять Keycloak (LDAPS, исходящие HTTPS-вызовы), под именем из `kc_ca_cert`;
- keytab для Kerberos под именем из `kc_keytab`.

Секреты можно дополнительно зашифровать: `ansible-vault encrypt files/keycloak/keycloak.env files/keycloak/keycloak.keytab`
(модуль `copy` расшифрует их при копировании, плейбук запускайте с `--ask-vault-pass`).

## Развертывание

```bash
ansible-playbook -i inventory.yml install_keycloak.yaml            # + --ask-vault-pass, если используете vault
```

Что делает плейбук:
1. Ставит nftables, раскладывает `/etc/nftables.d/keycloak.nft` (с проверкой синтаксиса) и применяет его.
2. Генерирует `docker-compose.yaml`, копирует CA и keytab в `/docker-compose/keycloak`.
3. Скачивает свежий образ.
4. Перезапускает **узел 1** и ждет `/health/ready`, затем перезапускает **узел 2** и ждет его готовности —
   один узел всегда остается в работе.
5. Чистит Docker: `system prune -a` и `volume prune -a`.

> ⚠️ `docker volume prune -a` удаляет **все** тома, не используемые запущенными контейнерами,
> в том числе именованные. Если на хостах есть другие сервисы с данными в томах — уберите эту строку.

## Сеть и firewall

| Порт | Назначение | Кому открыт |
|---|---|---|
| 28080 | HTTP Keycloak | `kc_proxies`, Docker-сети `172.16.0.0/12`, localhost |
| 9000 | health и метрики (`/health/ready`, `/metrics`) | `kc_monitoring`, `kc_proxies` |
| 7800 | JGroups (кластер) | `kc_nodes` |
| 57800 | JGroups FD_SOCK2 (детектор отказов) | `kc_nodes` |

Особенности:
- Правила живут в отдельной таблице `inet keycloak` и режут только эти порты — остальной трафик
  хоста и правила Docker не затрагиваются. Повторное применение файла заменяет таблицу целиком.
- `172.16.0.0/12` нужна потому, что прокси работает в контейнере с bridge-сетью: к Keycloak на своем
  хосте он приходит с адреса Docker-сети, а не с IP хоста. Эта же подсеть должна быть в `KC_PROXY_TRUSTED_ADDRESSES`.
- **Не делайте `systemctl reload/restart nftables`** на работающем хосте: стандартный `/etc/nftables.conf`
  начинается с `flush ruleset` и снесет правила Docker. Применять правила — только `nft -f /etc/nftables.d/keycloak.nft`.
  При загрузке системы это безопасно: nftables стартует раньше Docker.
- `KC_PROXY_TRUSTED_ADDRESSES` — Keycloak принимает заголовки `X-Forwarded-*` только от этих адресов,
  поэтому подделать IP клиента или host в обход прокси нельзя.

## Проверка

```bash
docker ps                                                   # STATUS: healthy
docker logs keycloak 2>&1 | grep "cluster view"             # в view должно быть 2 участника
curl -s http://<node>:9000/health/ready                     # {"status":"UP",...}
nft list table inet keycloak
```

## Обновление

Образ используется с тегом `latest`: каждый прогон плейбука ставит последнюю версию.
Перед прогоном, который может принести новую **минорную** версию (например, 26.8 → 26.9):
- сделайте бэкап БД — при старте Keycloak может мигрировать схему;
- прочитайте [upgrading guide](https://www.keycloak.org/docs/latest/upgrading/): если узлы разных версий
  не совместимы, второй узел не войдет в кластер. Тогда остановите оба узла и поднимите их по одному.

## Диагностика

**Страница входа грузится очень долго.** Почти всегда это firewall: прокси ждет таймаута до одного
бэкенда и уходит на второй. Проверка — временно убрать таблицу: `nft delete table inet keycloak`
(вернуть: `nft -f /etc/nftables.d/keycloak.nft`). Посмотреть, кого режут:
```bash
nft -a list chain inet keycloak input                     # handle строки с drop
nft insert rule inet keycloak input position <HANDLE> tcp dport { 28080, 9000, 7800, 57800 } log prefix \"kc-drop: \"
journalctl -k -f | grep kc-drop                           # SRC= — кого добавить в разрешенные
```

**Узлы не видят друг друга** (в `cluster view` один участник, таймауты в логах):
проверьте `nodeip`, доступность 7800/57800 между узлами и что оба узла одной версии.

**В логах при старте** `Broken pipe` от `org.jgroups.protocols.TCP` и единичная `Query was empty` от MariaDB —
нормальны при перезапуске узла.
