# Playbook for upgrade RouterOS/RouterBOARD

> Это мой первый проект по Ansible, если вы знаете как решить текущие проблемы (описаны ниже) или оптимизировать/улучшить выполнение тасков, приветствуются пул реквесты. Спасибо.

Набор плейбуков позволяет обновить RouterOS/RouterBOARD на устройствах Mikrotik.  
**Задача:** Тестировалось на массовом обновлении 40шт Mikrotik HAP AC Lite.

### Схема подключения
![scheme](https://github.com/woohung/ansible-mikrotik-update/blob/main/img/scheme.png)

### Дерево проекта
```
.
├── inventory
│   └── routers.yml
├── playbooks
│   ├── 0_mkt_init.yml
│   ├── 1_mkt_upgrade_ROS.yml
│   ├── 2_mkt_upgrade_RB.yml
│   ├── 3_mkt_check_ver.yml
│   ├── 4_mkt_clear.yml
│   └── ansible.cfg
└── tasks
    ├── 00_init_config.yml
    ├── 01_upgrade_ros.yml
    ├── 02_upgrade_rb.yml
    ├── 03_check_ros_version.yml
    ├── 04_check_rb_version.yml
    ├── 05_clear_config.yml
    ├── 06_remove_reboot_script.yml
    └── 07_set_defautl_ip.yml
```
- в плейбуках используется `include_tasks` на задачи в директории `/tasks`
- в `ansible.cfg` необходимо установить:
    - `inventory=/path_to_inventory/routers.yml`
    - persistent connection увеличен до 240 (ловил таймауты на подключение, если задача на одном из устройств затягивалась)

### Порядок запуска плейбуков

1. `0_mkt_init.yml` - Обновляемым устройствам необходим доступ в интернет. Кнфигурируется DNS и маршрут до Master Mikrotik т.к обновление производится через сервера Mikrotik.

2. `1_mkt_upgrade_ROS.yml` - Выбирает канал обновления, затем делает check-for-updates, затем выполняется обновление с последущей перезагрузкой. Если на этапе проверки выяснилось, что установлена актуальная прошивка, Ansible пропускает такое устройство.  
```
--- VARS ---
channel:  development,long-term,stable,testing,upgrade (default: "long-term"; string)
```
3. `2_mkt_upgrade_RB.yml` - Проверяется текущая версия RouterBOARD, если не совпадает с upgrade - обновляется, если совпадает - Ansible пропускает такое устройство. После обновления выполняется перезагрузка,  через добавление задания rebootonece в планировщик.

4. `3_mkt_check_ver.yml` - (опционально) Проверка версий прошивок на актуальность.
```
--- VARS ---
version: Версия прошивки  (default: "6.48.6"; string)
```
5. `4_mkt_clear.yml` - Очистка ранее добавленных настроек: DNS/Маршрут/Скрипт перезагрузки и возвращение IP-адреса по умолчанию.
```
--- VARS ---
gw: (default: "192.168.88.2"; string)  
default_ip: (default: "192.168.88.1/24"; string)  
intf: (default: "bridge"; string)  
```
### Проблемы
1. При первичной настройке, адреса, по которым будет осуществляться SSH подключения, находятся в LAN сегменте (начиная с ether2). Менял их руками, выдавая  по порядку IP-адреса из 192.168.88.0/24  
2. После завершения плейбука №4, не разобрался как вернуть IP-адрес по умолчанию и при этом корректно завершить задачу Ansible. Сейчас получается так, что под конец плейбука, ip-адреса сбрасываются на всех девайсах, но они более недоступны по SSH, поэтому задача "зависает". Помогает только CTRL+C через пару секунд (в зависимости от количества устройств)  
