# Talos Linux release 1.12.1

генерация конфигов через Ansible

Ansible-плейбук для генерации конфигураций Talos Linux по переменным из папки кластера: hostname, сеть, VIP, секреты и конфиги для control plane и worker нод.

---

## Требования

- **Ansible** — на машине, где запускается плейбук
- **talosctl** — в `PATH` на той же машине
- Папка кластера с переменными (например, `cluster_1/`) и общие патчи в `common_patches/`

---

## Запуск

```bash
ansible-playbook playbook.yml -e "cluster_name=cluster_1"
```

Параметр **`cluster_name`** должен совпадать с именем папки кластера (например, `cluster_1`), из которой подтягиваются переменные (`vars.yml`).

---

## Структура проекта

| Путь | Назначение |
| ------ | ------------ |
| `cluster_<name>/vars.yml` | Переменные кластера: VIP, шлюз, интерфейс, списки control plane и worker нод (hostname → IP) |
| `common_patches/` | Общие патчи Talos (Common, Registry, TimeSync, TrustedRoots и др.) |
| `common_patches/*.j2` | Шаблоны патчей (Hostname, Link, Layer2 VIP) — рендерятся под каждую ноду |
| `generated_patches/` | Временные сгенерированные патчи (удаляются в конце прогона) |
| `output/` | Результат: `secrets.yaml`, `talosconfig.yml`, конфиги нод (`controlplane_1.yml`, `worker_1.yml` и т.д.) |

---

## Что делает плейбук

1. Проверяет наличие **talosctl** и выводит версию.
2. Генерирует по шаблонам патчи **Hostname**, **Link** и **Layer2 VIP** из `vars.yml`.
3. Создаёт **secrets** для Talos (если файла ещё нет).
4. Генерирует **talosconfig** и конфиги для каждой ноды (control plane и worker) с подстановкой общих и нодовых патчей.
5. Проверяет конфиги через **talosctl validate**.
6. Удаляет временные файлы из `generated_patches/`.
7. **Проверка и применение конфига на ноды** — для каждой ноды (по циклу `controlplanes` + `workers`):
   - проверяет, что IP ноды есть в соответствующем MC-файле (`output/{{ item.key }}.yml`) через `grep`; если IP нет — **задача завершается с ошибкой**;
   - при успешной проверке можно применять конфиг (`talosctl apply-config` закомментирован — раскомментируйте в плейбуке для применения).
8. Выводит результат проверки/применения по каждой ноде (**Print apply results**).

---

## Переменные кластера (vars.yml)

- `cluster_name`, `vip_address`, `gateway`, `ethernet_interface`
- `controlplanes` — словарь «hostname → IP»
- `workers` — словарь «hostname → IP»

Остальные настройки задаются в `common_patches/`; при необходимости можно вынести часть в переменные.

---

## Заметки

- **work_dir**: по умолчанию `/common`; можно переопределить: `-e "playbook_work_dir=/путь/к/проекту"` (например, локально: `-e "playbook_work_dir=."`).
- Отдельные патчи для **control plane** (etcd, audit и т.п.) лежат в `common_patches/` (в т.ч. `ContolPlane_patch.yml`); при необходимости их нужно явно подключить в `playbook.yml`.
- **Apply config**: таска «talosctl apply config for nodes» сначала проверяет через `grep`, что IP ноды присутствует в MC-файле; при отсутствии IP задача завершается с ошибкой. Реальный вызов `talosctl apply-config` закомментирован — для применения конфигов раскомментируйте его в плейбуке.

---

<!-- Ниже — старая версия заметок (сохранена без изменений).

Запуск командой с передачей параметра cluster_name- котоырй должен соотвествовать названию папки в 
и Кластера котоырй нужно задеплоить, и из которого будут браться переменные для кластера
В Файле common_pathes -общие патчи, 
!наверное для всех и мастеров и воркеров - проверить как будет рабоатать если его задать, будет ли etcd например!
Наверное улчше отедльные патчи для CP - с параметрами  auditpolicy и etcd как минимум + cluste.ca вроде ?
ansible-playbook playbook.yml -e "cluster_name=cluster_1" 

Пока работает так
проверяет talosctl
генерит по шаблону hostname и IP + VIP из vars
gen secret 
gen talosconfig
talosctl gen для каждой ноды
проверка talosctl validate

По идее нужно редачить vars -> common_patches (нужно приудмать только переменные)

Сначала команды - потом проверка

-->
