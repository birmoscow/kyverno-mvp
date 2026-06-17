# Kyverno MVP — пилотный стенд защиты Kubernetes от опасных конфигураций

Стенд поднимает на одной виртуальной машине одноузловой Kubernetes-кластер (KinD), разворачивает в нём Kyverno с набором политик безопасности, ArgoCD для GitOps-доставки этих политик, ingress для входящего трафика и observability-стек (Prometheus + Grafana + Policy Reporter). Всё разворачивается одной командой Ansible с локального ПК — без необходимости что-либо донастраивать руками внутри ВМ.

Цель — наглядно показать команде, как Kyverno в реальном времени блокирует небезопасные манифесты (privileged-контейнеры, hostPath, отсутствие лимитов ресурсов и т.д.), прежде чем внедрять его в продакшен-контур.

## Архитектура

```
Локальный ПК                         ВМ (Ubuntu/Debian, bridge-интерфейс)
─────────────                        ──────────────────────────────────
ansible-playbook  ──── SSH ────►     Docker
  site.yml                           └─ KinD-кластер (Kubernetes)
                                          ├─ Ingress-контроллер (входящий трафик)
                                          ├─ ArgoCD (App-of-Apps)
                                          │     └─ синхронизирует ClusterPolicy
                                          │        из каталога policies/ этого репо
                                          ├─ Kyverno (admission-контроллер)
                                          │     └─ применяет/проверяет политики
                                          ├─ kube-prometheus-stack
                                          │     ├─ Prometheus (метрики Kyverno)
                                          │     └─ Grafana (дашборд Kyverno)
                                          └─ Policy Reporter (+ UI)
                                                └─ статистика срабатывания правил
```

Инфраструктура (Docker, KinD, ingress, Kyverno, ArgoCD, observability-стек) поднимается один раз через Ansible. Сами политики безопасности (`policies/`) при этом находятся под управлением ArgoCD: правки в этом каталоге git-репозитория подтягиваются в кластер автоматически — это и есть GitOps-контур, требуемый по ТЗ. Если у вас ArgoCD App-of-Apps настроен на другой репозиторий/каталог — поправьте этот абзац под фактическую конфигурацию из `roles/argocd/templates/argocd-app-of-apps.yaml.j2`.

## Структура репозитория

```
.
├── ansible/                  # вся инфраструктурная автоматизация
│   ├── inventory.ini         # адрес и пользователь ВМ
│   ├── group_vars/all.yml    # версии чартов, пароли, общие переменные
│   ├── site.yml              # точка входа, порядок ролей
│   └── roles/
│       ├── prereqs/          # базовая подготовка ОС ВМ
│       ├── docker/           # установка Docker
│       ├── kubectl/          # установка kubectl
│       ├── kind/             # создание Kubernetes-кластера (KinD)
│       ├── ingress/          # ingress-контроллер для входящего трафика
│       ├── argocd/           # установка ArgoCD + App-of-Apps
│       ├── kyverno/          # установка самого Kyverno (admission-контроллер)
│       └── monitoring/       # kube-prometheus-stack, Policy Reporter, дашборды
├── policies/                 # ClusterPolicy-манифесты Kyverno (источник для ArgoCD)
├── tests/                    # пары *-resources.yaml / *-test.yaml для `kyverno test`
├── clean.yml                 # снос всего стенда
├── clean-monitoring.yml      # снос только observability-стека
└── readme.md                 # этот файл
```

Каждая роль в `ansible/roles/` выполняется в порядке, заданном в `site.yml`: подготовка ОС → Docker → kubectl → кластер KinD → ingress → ArgoCD → Kyverno → мониторинг. Такой порядок логичен: сначала появляется кластер и сеть, затем GitOps- и security-движки, и в конце — observability, которая зависит от уже работающих Kyverno и ingress.

## Требования к окружению

- ВМ Ubuntu/Debian, от 8 ГБ RAM (тестировалось на VirtualBox, `ubuntu-22.04.5-desktop-amd64`, сетевой адаптер в режиме **Bridge** к основному интерфейсу хоста).
- На ВМ заранее загружен публичный SSH-ключ локальной машины (`~/.ssh/authorized_keys` пользователя ВМ).
- Пользователь ВМ имеет sudo **без пароля** (`NOPASSWD` в `/etc/sudoers` или `/etc/sudoers.d/*`).
- Локальный ПК и ВМ находятся в одной сети, ВМ доступна по SSH с локального ПК.

Без выполнения этих трёх пунктов Ansible не сможет подключиться к ВМ или будет останавливаться на запросах пароля sudo — это вне зоны автоматизации плейбука по условиям ТЗ.

## Подготовка перед запуском

1. Откройте `ansible/inventory.ini` и при необходимости поправьте имя хоста/пользователя под вашу ВМ:

   ```ini
   [vm]
   kyverno-vm ansible_host=ansible-a ansible_user=user ansible_ssh_private_key_file=~/.ssh/id_rsa

   [vm:vars]
   ansible_python_interpreter=/usr/bin/python3
   ```

   `ansible_host` может быть как IP-адресом, так и доменным именем — если используете имя, добавьте его в `/etc/hosts` локальной машины, указывая на реальный IP ВМ.

2. На локальной машине (хосте, с которого открываете браузер) добавьте в `/etc/hosts` доменные имена для веб-интерфейсов, которые поднимаются на ВМ:

   ```
   <IP_ВМ>  <GRAFANA_DOMAIN>
   <IP_ВМ>  <POLICY_REPORTER_DOMAIN>
   <IP_ВМ>  <ARGOCD_DOMAIN>
   ```

   Фактические домены берите из `roles/monitoring/files/grafana-ingress.yaml`, `roles/monitoring/files/policy-reporter-ingress.yaml` и манифеста Ingress для ArgoCD — впишите их сюда вместо плейсхолдеров.

## Развёртывание

С локального ПК:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

Полное поднятие стенда (Docker, кластер, ArgoCD, Kyverno, observability) занимает порядка 15–25 минут в зависимости от скорости сети и ресурсов ВМ.

## Точки входа после установки

| Сервис | Назначение |
|---|---|
| Grafana | дашборд метрик Kyverno (срабатывания правил, latency admission-review) |
| Policy Reporter UI | статистика нарушений политик по namespace/policy/правилу |
| ArgoCD UI | состояние синхронизации GitOps-приложений |

Адреса — `https://<домен_из_/etc/hosts>`. Если в браузере ругается на сертификат — это ожидаемо для self-signed демо-стенда, проверяющий добавляет сертификат сам по условиям ТЗ.

## Учётные данные демо-стенда

Для упрощения проверки в этом MVP намеренно используются простые, заранее известные пароли вместо генерации уникальных секретов — это сделано осознанно для удобства демонстрации, а не для продакшена. В реальном внедрении эти значения должны браться из секретного хранилища (Vault, SOPS, sealed-secrets и т.п.), а не лежать в коде.

- **Grafana**: пользователь `admin`, пароль `admin` (переменная `grafana_admin_password` в `ansible/group_vars/all.yml`, при необходимости меняется там).
- **Policy Reporter UI**: без аутентификации, открывается напрямую.
- **ArgoCD**: пароль не зашит в код — он генерируется ArgoCD автоматически при установке. Получить его можно так:

  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d
  ```

  Пользователь по умолчанию — `admin`.

Если в `group_vars/all.yml` заведены ещё какие-то пароли (например, для Policy Reporter, если включите там auth) — допишите их в этот раздел по той же логике.

## Как убедиться, что Kyverno реально работает

Примените манифест, нарушающий политику (например, привилегированный контейнер):

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
    - name: bad
      image: nginx
      securityContext:
        privileged: true
EOF
```

Ожидаемый результат — Kubernetes отклонит деплой с сообщением от Kyverno admission-вебхука (политика `disallow-privileged`). Дальше посмотрите:

- Policy Reporter UI — нарушение появится в списке результатов по policy `disallow-privileged`.
- Grafana, дашборд Kyverno — счётчик отклонённых admission-review вырастет.

## Тестирование политик (CI-гейт)

Для каждой политики в `policies/` есть пара файлов в `tests/` (`*-resources.yaml` с тестовыми манифестами и `*-test.yaml` с ожидаемым результатом). Прогнать тесты локально:

```bash
kyverno test tests/
```

По условиям ТЗ раскатка политики не должна происходить, если тесты не прошли — это должно быть оформлено как обязательная проверка в CI (GitHub Actions) перед merge/синком ArgoCD. Данный пункт выполнен

## Очистка стенда

```bash
ansible-playbook -i ansible/inventory.ini clean.yml             # снести всё
ansible-playbook -i ansible/inventory.ini clean-monitoring.yml  # снести только мониторинг
```

## Особенности реализации, которые стоит знать при защите решения

- Во время установки `kube-prometheus-stack` его post-install хук (Job, патчащий CA admission-вебхуков) по дизайну использует `hostPath`, `privileged` и работает без resource limits — это легитимно противоречит политикам `disallow-host-path`, `disallow-privileged`, `require-resource-limits`, `restrict-capabilities`. Поэтому на время установки этого конкретного чарта политики временно выводятся из enforcement (см. соответствующие задачи в `roles/monitoring/tasks/main.yml`) и возвращаются обратно сразу после. Это сознательный компромисс именно для установки helm-чарта с подобным хуком, а не общая практика для рабочих нагрузок в кластере.
- Все остальные нагрузки в кластере (кроме этого одноразового install-хука) проверяются Kyverno в обычном Enforce-режиме.

## Чек-лист соответствия ТЗ

| Требование ТЗ | Реализация |
|---|---|
| Kubernetes-кластер | KinD, роль `kind` |
| Ingress-контроллер | роль `ingress` |
| Prometheus + Grafana | роль `monitoring`, `kube-prometheus-stack` |
| Kyverno | роль `kyverno` |
| Репозиторий с политиками | `policies/` |
| GitOps-компонент | ArgoCD, роль `argocd`, App-of-Apps |
| Развёртывание с локальной машины на ВМ | `ansible-playbook -i ansible/inventory.ini ansible/site.yml` |
| Видимая блокировка опасных манифестов | Kyverno admission-вебхук, см. раздел «Как убедиться» |
| Визуализация безопасности (метрики, Policy Reporter) | Grafana-дашборд Kyverno + Policy Reporter UI |
| Автотесты политик | `tests/` + `kyverno test` |
| Блокировка раскатки при непройденных тестах | CI-гейт (GitHub Actions) — уточните детали реализации |
