# demo2024

## Таблица
| Имя устройства | Интерфейс | IPv4/IPv6     | Маска/Префикс | Шлюз          |
| -------------  | --------- | ------------- | ------------- | ------------- | 
| ISP            | ens ---   | DHCP          | /24           | DHCP          |
|                | ens ---   | 192.168.0.165 | /30           |               |
|                | ens ---   | 192.168.0.161 | /30           |               |
| HQ-RTR         | ens ---   | 192.168.0.129 | /             |               |
|                | ens ---   | 192.168.0.1   | /             |               |
|                | ens   --- | 192.168.0.    | /             |               |
| BR-RTR         | ens ---   | 192.168.0.    | /             |               |
|                | ens ---   | 192.168.0.    | /             |               |
| HQ-SRV         | ens ---   | 192.168.0.    | /             |               |
| HQ-CLI         | ens ---   | 192.168.0.130 | /             |               |
| BR-SRV         | ens ---   | 192.168.0.130 | /             |               |
