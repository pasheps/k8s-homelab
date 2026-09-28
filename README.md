# k8s-homelab

Домашний Kubernetes-кластер, развёрнутый с нуля через Ansible на трёх виртуальных машинах (VirtualBox), с полным стеком: MySQL, мониторинг (Prometheus + Grafana) и Zabbix — все ключевые компоненты, кроме мониторинга, написаны вручную в виде «сырых» Kubernetes-манифестов, а не через готовые Helm-чарты.

Цель проекта — не просто поднять работающий кластер, а показать понимание того, как устроен Kubernetes и Ansible «под капотом»: с самостоятельным написанием StatefulSet, Service, Secret, Ingress и DaemonSet, а не копированием чужих чартов.

## Архитектура

```
                    ┌─────────────────────────────┐
                    │        Windows host         │
                    │  (VirtualBox, hosts-file)   │
                    └─────────────┬───────────────┘
                                  │ Host-only network 192.168.56.0/24
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
┌───────▼─────────┐      ┌────────▼─────────┐      ┌────────▼─────────┐
│   k8s-master    │      │   k8s-worker1    │      │   k8s-worker2    │
│  control-plane  │      │                  │      │                  │
│                 │      │  containerd      │      │  containerd      │
│  containerd     │      │  kubelet         │      │  kubelet         │
│  kubelet/kubeadm│      │  Flannel (CNI)   │      │  Flannel (CNI)   │
│  Flannel (CNI)  │      │  workloads       │      │  workloads       │
│  ingress-nginx  │      │  (MySQL, Zabbix, │      │  (MySQL, Zabbix, │
│  Helm           │      │   Grafana, ...)  │      │   Grafana, ...)  │
└─────────────────┘      └──────────────────┘      └──────────────────┘
```

Workloads (MySQL, Zabbix, Prometheus/Grafana) размещаются на worker-нодах; control-plane нода зарезервирована под системные компоненты и ingress-controller.

## Стек технологий

| Слой | Технология |
|---|---|
| Гипервизор | VirtualBox (3 × Ubuntu Server, NAT + Host-only сеть) |
| Автоматизация инфраструктуры | Ansible (роли, Vault, шаблоны Jinja2) |
| Container runtime | containerd (кастомный конфиг под kubeadm) |
| Оркестрация | Kubernetes (kubeadm, 1 control-plane + 2 worker) |
| CNI | Flannel (VXLAN) |
| Storage | local-path-provisioner |
| Ingress | ingress-nginx |
| СУБД | MySQL 8 (ручной StatefulSet) |
| Мониторинг | kube-prometheus-stack (Prometheus + Grafana, Helm) |
| Система мониторинга | Zabbix Server/Web/Agent (ручные манифесты, MySQL-бэкенд) |
| Секреты | Ansible Vault |

## Структура репозитория

```
ansible-k8s/
├── ansible.cfg
├── site.yml                 # точка входа
├── inventory/
│   └── hosts.yml
├── group_vars/all/
│   ├── all.yml               # обычные переменные
│   └── vault.yml             # зашифрованные секреты (Ansible Vault)
└── roles/
    ├── common/               # swap, hostname, sysctl, chrony
    ├── containerd/           # containerd + кастомный config.toml
    ├── kube-deps/            # kubelet/kubeadm/kubectl, закреплённая версия
    ├── control-plane/        # kubeadm init, Flannel
    ├── worker/               # kubeadm join
    ├── helm/                 # установка Helm CLI
    ├── storage/              # local-path-provisioner
    ├── ingress/              # ingress-nginx
    ├── mysql/                # StatefulSet + Service + Secret (ручные манифесты)
    ├── monitoring/           # kube-prometheus-stack (Helm) + Ingress
    └── zabbix/               # Server + Web + Agent DaemonSet + Ingress (ручные манифесты)
```

## Почему часть сервисов развёрнута вручную, а не через Helm

MySQL и Zabbix изначально планировались через готовые Helm-чарты (Bitnami для MySQL, community-чарт для Zabbix). В процессе разработки оба варианта оказались непригодны:

- **Bitnami** в 2025 году свернул бесплатный публичный каталог чартов (переход на закрытую подписочную модель), классический репозиторий `charts.bitnami.com` возвращает `403 Forbidden`.
- Официальный **community Helm-чарт Zabbix** поддерживает только PostgreSQL и не планирует добавлять MySQL.

Вместо того чтобы подстраиваться под ограничения сторонних чартов, было принято решение написать манифесты (StatefulSet, Service, Secret, Deployment, DaemonSet, Ingress) самостоятельно — это заняло больше времени, но дало полный контроль над конфигурацией и более глубокое понимание устройства этих Kubernetes-объектов.

## Ключевые инженерные решения

- **Явное закрепление версий** (`kubernetes_version`, `crictl_version` и т.д.) вместо "последней доступной" — предотвращает рассинхронизацию версий kubelet/kubeadm между control-plane и worker-нодами.
- **Ansible Vault** для всех паролей (MySQL root, Zabbix DB user, Grafana admin) — секреты хранятся зашифрованными прямо в репозитории, с разделением на `all.yml` (обычные переменные) и `vault.yml` (значения) через `vault_`-префикс.
- **Идемпотентные и устойчивые к сетевым сбоям задачи** — `retries`/`until` на всех операциях скачивания, `creates:` для проверки уже выполненных шагов, `CREATE USER IF NOT EXISTS` вместо безусловного создания.
- **Самовосстанавливающиеся секреты** — пароль администратора Grafana принудительно сбрасывается через `grafana cli` при каждом прогоне плейбука, а не полагается на однократную инициализацию Helm-чарта.
- **Отдельный MySQL-пользователь для Zabbix** с правами только на свою базу (`GRANT ALL ON zabbix.*`), а не использование root-доступа — принцип минимальных привилегий.
- **Явные security context для DaemonSet-агента** (`hostNetwork`, `hostPID`, монтирование `/proc`, `/sys`) с осознанным пониманием, что это расширенные привилегии, оправданные природой задачи мониторинга хоста.

## Известные ограничения

- **etcd/scheduler/controller-manager metrics** не собираются Prometheus — требует настройки TLS-доступа к статическим подам control-plane, актуально в первую очередь для multi-master кластеров.
- **MySQL — единственный под**, без реплик и без резервного копирования — приемлемо для homelab, в продакшене требовало бы отдельного решения для бэкапов (например, `mysqldump`-CronJob или Percona XtraBackup).
- **Секреты в Ansible Vault**, а не во внешнем secret-manager (HashiCorp Vault / External Secrets Operator) — осознанный выбор под масштаб задачи (см. обоснование в истории коммитов/обсуждении), но не то, что использовалось бы в кластере production-масштаба.
- **CI/CD не подключён** — плейбук запускается вручную; кластер изолирован в Host-only сети без доступа извне для GitHub Actions runners.

## Как развернуть

```bash
git clone https://github.com/pasheps/k8s-homelab.git
cd k8s-homelab

# создать локальный файл с паролем от Ansible Vault
echo "your-vault-password" > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt

ansible-playbook site.yml
```

После разворачивания добавьте на клиентской машине в `hosts`-файл:
```
192.168.56.10   grafana.local
192.168.56.10   zabbix.local
```

Доступ — через NodePort ingress-nginx (`kubectl get svc -n ingress-nginx` для актуального порта).
