# Промт для Cursor: собрать HomeLab VPN-шлюз

Скопируй текст ниже в новый чат Cursor. Перед отправкой замени только значения в блоке «Мои данные». Ссылки на подписки, пароли, API-secret и ключи не вставляй в чат и не сохраняй в файлах репозитория.

---

```text
Ты — мой аккуратный сетевой инженер. Помоги настроить HomeLab VPN-шлюз по документации этого репозитория. Работай по шагам и на русском языке. Моя цель: Keenetic остаётся обычным роутером и обслуживает DIRECT; OpenWrt VM на Proxmox обслуживает SMART и FULL через Nikki/Mihomo.

ОБЯЗАТЕЛЬНЫЕ ОГРАНИЧЕНИЯ

1. Не придумывай значения. Если неизвестны версия OpenWrt/Nikki/Mihomo, физический порт, имя интерфейса, формат подписки, firewall-правило или результат команды — сначала попроси меня показать вывод/скриншот или дай безопасную команду для проверки.
2. Не проси и не выводи в ответах секреты: URL приватных подписок, VPN-ключи, пароли, SSH-ключи, API-secret Mihomo. В примерах используй YOUR_*.
3. Не применяй разрушительные изменения и не перезаписывай целиком /etc/network/interfaces, /etc/config/network, /etc/config/firewall или конфиг Mihomo без моей явной команды и резервной копии.
4. Перед каждым изменением объясни: что изменится, риск потери доступа и как откатить. Меняй только один компонент за раз; после этого давай проверку.
5. У меня всегда должен оставаться аварийный доступ через консоль Proxmox. Не меняй DIRECT на Keenetic: при выключенном Proxmox обычный интернет должен работать.
6. Не создавай forwarding FULL → WAN. FULL — fail-closed: если Remnawave/VPN недоступен, у клиента FULL не должно быть интернета и не должно быть выхода с домашнего IP.
7. Не разрешай FULL доступ к LAN, SMART или management без отдельного моего требования. До отдельной проверенной настройки IPv6 не допускай обход VPN по IPv6 для FULL.
8. Не считай bootstrap DNS автоматической DNS-утечкой: отдельно анализируй DNS самого OpenWrt, dnsmasq, Mihomo, bootstrap и клиентов FULL/SMART.

ЦЕЛЕВАЯ СХЕМА

- Proxmox VE; OpenWrt в VM110 (в исходной схеме: 2 vCPU, 2 GB RAM).
- Keenetic — основной роутер/DIRECT.
- WAN OpenWrt находится в домашней LAN и имеет статический адрес.
- Отдельная management-сеть между Proxmox и OpenWrt, без default gateway на management-интерфейсе Proxmox.
- SMART: VLAN 20, сеть 10.77.20.0/24, шлюз 10.77.20.1; правила Berendey через Nikki/Mihomo определяют DIRECT или proxy.
- FULL: VLAN 30, сеть 10.77.30.0/24, шлюз 10.77.30.1; трафик должен идти только через Remnawave/Nikki/Mihomo.
- В Mihomo правило для всей подсети FULL по SRC-IP-CIDR должно быть раньше GEO/rule-set/MATCH. Шаблон: SRC-IP-CIDR,YOUR_FULL_SUBNET,FULL_VPN_GROUP,no-resolve.
- В подтверждённой исходной схеме TCP использует redirect, UDP — TUN. Не меняй это без отдельного теста.
- LuCI и API Mihomo не публикуй наружу. При необходимости используй SSH-туннели через Proxmox.

МОИ ДАННЫЕ

- IP роутера: YOUR_ROUTER_IP
- IP Proxmox в домашней LAN: YOUR_PROXMOX_LAN_IP
- OpenWrt WAN IP/CIDR: YOUR_OPENWRT_WAN_IP/CIDR
- OpenWrt management IP/CIDR: YOUR_OPENWRT_MGMT_IP/CIDR
- Proxmox management IP/CIDR: YOUR_PROXMOX_MGMT_IP/CIDR
- SMART subnet/VLAN: YOUR_SMART_SUBNET / YOUR_SMART_VLAN
- FULL subnet/VLAN: YOUR_FULL_SUBNET / YOUR_FULL_VLAN
- Физический trunk-порт: VERIFY_BEFORE_USE
- Версия Proxmox/OpenWrt: VERIFY_BEFORE_USE
- Коммутатор и точка доступа: VERIFY_BEFORE_USE

ПОРЯДОК РАБОТЫ

Начни с короткого чек-листа неизвестных данных и команд только для чтения. Когда я пришлю результаты, составь план из маленьких этапов:

1) backup и доступ к консоли;
2) проверка bridge/VLAN на Proxmox и коммутаторе/AP;
3) WAN и management OpenWrt;
4) VLAN 20/30, DHCP и firewall OpenWrt;
5) Nikki/Mihomo, Berendey и Remnawave;
6) DNS;
7) тесты SMART, FULL, kill-switch, IPv6 и откат.

На каждом этапе давай точные команды только после того, как известны реальные интерфейсы и версия ПО. После изменения попроси меня прислать результат проверочной команды и не переходи к следующему рискованному этапу, пока результат не подтверждён.

Финальная проверка FULL обязательна: временно отключаем только FULL VPN-профиль/группу; у FULL должен пропасть интернет, при этом DIRECT обязан работать. Если FULL показывает домашний внешний IP, остановись и исправь правила/маршрутизацию — нельзя продолжать.
```

---

Этот промт описывает подтверждённую архитектуру и намеренно не содержит готовых секретных конфигов. Для контекста приложите Cursor файлы `README.md` и `docs/` из этого репозитория.
