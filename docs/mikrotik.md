ОС: chr-7.20.1
CPU: 1/1
RAM: 256MB
HD: 128 MB (vmdk)
Интерфейсы: ether1 (bridge) - 00:0C:29:B6:98:FC (192.168.3.200); ether2 (host-only) - 00:A0:A0:A0:A0:00 (172.10.10.2)

## Первоначальная настройка
- Авторизоваться под admin, без пароля
- Затем прочесть или не прочесть лицензию 
- Ввести новый пароль, в моем случае, для простоты, это `1111`
- `ip address/print` для того чтобы посмотреть адреса интерфейсов. В моем случае адреса по dhcp по какой-то причине получены не были
- `interface/print` для того чтобы проверить, что интерфейсы вообще подключены. Есть вывод для ether1, ether2, и lo интерфейсов.
- `ip address/add address=192.168.3.200/24 interface=ether1`, где 192.168.3.200 это адрес в моей домашней сети
- `ip route/add gateway=192.168.3.1`, где 192.168.3.1 это адрес шлюза внешней сети
- `ip address/print` - для проверки того, что адрес действительно применился
- `ping 192.168.3.1` - для проверки, что виртуальная машина имеет доступ к шлюзу внешней сети
- `ip address/add address=172.10.10.2/24 interface=ether2` - установить адрес для интерфейса внутренней сети. `172.10.10.2`, а не `172.10.10.1` по той причине, что первый занят хостовой машиной
- `ip address/print` - снова для поверки того, что адрес применился
- На хостовой машине `ping 172.10.10.2` для того чтобы убедиться, что с нее действительно есть доступ к шлюзу

## Настройка в качестве шлюза
- `ip firewall/nat/add chain=srcnat out-interface=ether1 action=masquerade` - Чтобы устройства из LAN выходили в интернет
- `ip dns/set allow-remote-requests=yes servers=8.8.8.8,1.1.1.1` - Включаем встроенный DNS и разрешаем клиентам использовать роутер как резолвер
- `ip pool/add name=lan-pool ranges=172.10.10.10-172.10.10.200` - пул адресов dhcp
- `ip dhcp-server/add name=lan-dhcp interface=ether2 address-pool=lan-pool lease-time=10m` - создание DHCP-сервера
- `ip dhcp-server/network/add address=172.10.10.0/24 gateway=172.10.10.2 dns-server=172.10.10.2` - параметры сети для DHCP

## Установка статического ip для хастов со стороны dhcp сервера
- `ip dhcp-server/lease/add address=172.10.10.10 mac-address=00:A0:A0:A0:A0:10 comment="nfs0.lab"` - установка статического ip, в данном случае для хоста [nfs0](./nfs-server.md)
- `ip dhcp-server/lease/add address=172.10.10.20 mac-address=00:A0:A0:A0:A0:20 comment="master0.lab"` - установка статического ip
- `ip dhcp-server/lease/add address=172.10.10.30 mac-address=00:A0:A0:A0:A0:30 comment="worker0.lab"` - установка статического ip

## Задание доменных имен машин
- `ip dns/static/add name=nfs0.lab address=172.10.10.10`
- `ip dns/static/add name=master0.lab address=172.10.10.20`
- `ip dns/static/add name=worker0.lab address=172.10.10.30`