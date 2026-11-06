# 3. OpenWrt и firewall

## Первый вход и WAN

Загрузите VM, войдите через консоль Proxmox и сначала определите реальные интерфейсы:

```sh
ip -br link
ubus call system board
```

Порядок `eth0/eth1/...` не гарантирован. Подключайте интерфейсы по MAC-адресам, видимым в Proxmox, а не по догадке.

В LuCI создайте WAN со статическими значениями из вашей карточки: IP OpenWrt, маска `/24`, шлюз главного роутера. Проверьте с OpenWrt:

```sh
ping -c 3 YOUR_ROUTER_IP
nslookup openwrt.org
```

Должен отвечать ping и разрешаться имя. Если нет — проверьте MAC/NIC VM, IP, маску и gateway. Не устанавливайте Nikki, пока WAN не работает.

В LuCI откройте **Network → Interfaces**. У WAN нажмите **Edit**, выберите **Protocol: Static address** и введите:

```text
IPv4 address: 192.168.1.250       # либо YOUR_OPENWRT_WAN_IP
IPv4 netmask: 255.255.255.0       # для /24
IPv4 gateway: 192.168.1.1         # либо YOUR_ROUTER_IP
```

Для management нажмите **Add new interface**:

```text
Name:          management
Protocol:      Static address
Device:        NIC, подключённый к vmbr2
IPv4 address:  10.77.0.1
IPv4 netmask:  255.255.255.0
IPv4 gateway:  оставить пустым
```

С Proxmox выполните `ping 10.77.0.1`: это независимый путь управления OpenWrt.

## Сети

Создайте management (`10.77.0.1/24`), SMART (`10.77.20.1/24`) и FULL (`10.77.30.1/24`). Для SMART/FULL включите DHCPv4; в исходной схеме планировался диапазон `.100–.250`.

Для VLAN trunk создайте устройства `802.1q` с tag `20` и `30` поверх проверенного родительского интерфейса, затем привяжите их к SMART/FULL. В LuCI это **Network → Devices → Add device configuration → VLAN (802.1q)**. Сохраните и переподключите тестовый клиент.

ПРОВЕРИТЬ: DSA vs. swconfig в вашей сборке OpenWrt. Интерфейс LuCI и имя родительского устройства различаются.

В **Network → Devices → Add device configuration → VLAN (802.1q)** создайте два устройства:

```text
Parent device: YOUR_TRUNK_DEVICE
VLAN ID:       20                 # SMART
```

и

```text
Parent device: YOUR_TRUNK_DEVICE
VLAN ID:       30                 # FULL
```

В **Network → Interfaces → Add new interface** создайте `smart` на устройстве `.20` и `full` на `.30`:

| Поле | smart | full |
|---|---|---|
| Protocol | Static address | Static address |
| IPv4 address | `10.77.20.1` | `10.77.30.1` |
| Netmask | `255.255.255.0` | `255.255.255.0` |
| Gateway | пусто | пусто |

В **Network → DHCP and DNS** включите DHCP на обоих интерфейсах. Исходный пример: start `100`, limit `151` (адреса около `.100–.250`). Проверка: SMART-клиент получает `10.77.20.x`, FULL-клиент — `10.77.30.x`; gateway у обоих — `.1` своей сети.

## Firewall: минимальная политика

Создайте отдельные зоны `smart` и `full`. Разрешайте DHCP/DNS от клиентов к самому OpenWrt в input. SMART получает доступ к механизму прозрачного прокси по настройке Nikki.

FULL обязан быть fail-closed:

```text
full → WAN: НЕ создавать forwarding
full → LAN / smart / management: НЕ создавать forwarding без осознанной причины
input: только DHCP/DNS/нужное управление
output: ACCEPT
forward: REJECT
```

Это не заменяет правила Nikki/Mihomo: firewall предотвращает обычный обход через WAN, а Mihomo направляет разрешённый трафик к VPN.

В LuCI откройте **Network → Firewall → Zones → Add**. Для `full` выберите сеть `full` и задайте:

```text
Input:       REJECT
Output:      ACCEPT
Forward:     REJECT
Forwardings: не создавать full → wan
```

В **Traffic Rules** добавьте только исключения, необходимые клиенту для получения IP и DNS:

| Имя | Source zone | Protocol | Destination port | Action |
|---|---|---|---|---|
| Allow-FULL-DHCP | full | UDP | `67-68` | ACCEPT |
| Allow-FULL-DNS | full | TCP+UDP | `53` | ACCEPT |

До настройки Mihomo у FULL не должно быть интернета. Это правильный промежуточный результат; не «исправляйте» его forwarding `full → wan`.

## IPv6

До отдельной проверенной настройки IPv6 через VPN отключите RA и DHCPv6 для FULL. Иначе возможен обход: IPv4 идёт через VPN, IPv6 — напрямую. После изменений переподключите FULL-клиент и проверьте, что он не получил глобальный IPv6-маршрут.
