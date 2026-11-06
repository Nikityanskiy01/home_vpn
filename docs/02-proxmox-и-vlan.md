# 2. Proxmox, bridges и VLAN

## Логика интерфейсов VM

Создайте OpenWrt VM (в исходной схеме — ID 110, 2 vCPU, 2 GB RAM) и добавьте отдельные виртуальные NIC:

| Назначение | Куда подключить | IP/роль |
|---|---|---|
| WAN | bridge домашней LAN (`vmbr0` в примере) | статический `192.168.1.250/24`, шлюз Keenetic |
| management | изолированный bridge (`vmbr2` в примере) | `10.77.0.1/24` |
| VPN trunk | VLAN-aware bridge (`vmbr1` в примере) | теги 20 и 30 |

Названия `vmbr0/vmbr1/vmbr2` — пример. До изменений посмотрите текущий файл:

```bash
cat /etc/network/interfaces
ip -br link
```

Не вставляйте целиком чужой `/etc/network/interfaces`: имена физических адаптеров и bridge у каждого сервера свои. В Proxmox GUI откройте **Node → System → Network** и создайте/проверьте:

1. WAN bridge привязан к физической LAN и имеет IP Proxmox, если это текущий management bridge.
2. Отдельный изолированный management bridge без физического порта. IP `10.77.0.2/24` допускается только для управления; default gateway там не указывайте.
3. Bridge для VLAN-trunk: включите `VLAN aware`. Привяжите к физическому порту только если этот порт действительно подключён к trunk-коммутатору.

### Пример фрагмента (не готовая конфигурация)

```ini
auto vmbr1
iface vmbr1 inet manual
    bridge-ports YOUR_TRUNK_NIC
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
```

`YOUR_TRUNK_NIC` требует проверки. Неправильное имя может отрезать сервер от сети.

## VLAN на VM и оборудовании

Для trunk NIC VM разрешите VLAN 20 и 30. На коммутаторе порт к Proxmox должен быть tagged trunk; порт к точке доступа — тоже tagged trunk, если она раздаёт оба SSID. Порты обычных клиентов делайте untagged ровно в одном VLAN.

ПРОВЕРИТЬ: конкретный режим VLAN и названия меню у модели Keenetic/коммутатора/AP. Схема требует именно того, чтобы VLAN 20 и 30 доходили до OpenWrt, а не только существовали в Proxmox.

Включите **Start at boot** для VM. После первоначальной настройки создайте backup VM через Proxmox (например, vzdump).

## Что нажимать в Proxmox

### Создание VM

Нажмите **Create VM**. В исходной схеме: VM ID `110`, 2 CPU cores, 2048 MiB RAM. После создания откройте VM → **Hardware** → **Add → Network Device**. Нужны отдельные NIC для WAN, management и VLAN trunk. Запишите MAC-адрес каждого NIC: в OpenWrt по нему станет понятно, какой интерфейс куда подключён.

### Management bridge

Откройте **Node → System → Network → Create → Linux Bridge** и для примера исходной схемы введите:

```text
Name:          vmbr2             # пример; убедитесь, что имя свободно
IPv4/CIDR:     10.77.0.2/24
Gateway:       оставить пустым
Bridge ports:  оставить пустым
```

Подключите management NIC VM к `vmbr2`. Это изолированная сеть только Proxmox ↔ OpenWrt; второй default gateway на Proxmox здесь запрещён.

### VLAN trunk bridge

Создайте/проверьте bridge, подключённый к trunk:

```text
Name:          vmbr1             # пример
Bridge ports:  YOUR_TRUNK_NIC
VLAN aware:    включить
IPv4/CIDR:     оставить пустым
Gateway:       оставить пустым
```

Узнайте `YOUR_TRUNK_NIC` командами из начала документа. Перед **Apply Configuration** оставьте открытой консоль/ILO: ошибочный порт способен оборвать SSH.

| Где | SMART | FULL |
|---|---|---|
| NIC OpenWrt на trunk bridge | VLAN `20` | VLAN `30` |
| Порт Proxmox на коммутаторе | tagged `20` | tagged `30` |
| Порт AP, если он раздаёт SSID | tagged `20` | tagged `30` |
| Проводной клиентский порт | untagged `20` | untagged `30` |

**Проверка:** Proxmox после применения всё ещё открывается, а DIRECT-клиент не потерял интернет. Иначе верните последнее изменение bridge/порта и не настраивайте OpenWrt дальше.
