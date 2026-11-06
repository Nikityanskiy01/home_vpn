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
