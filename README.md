# 🌍 RouterOS 'RU' Address List

[![Auto-updated](https://img.shields.io/badge/обновляется-ежедневно-brightgreen)]()
[![Countries](https://img.shields.io/badge/страны-RU-blue)](https://github.com/pavmiro/routeros-ru-address-list/tree/main/RU)
[![Source](https://img.shields.io/badge/источник-RIPE%20NCC-orange)](https://ftp.ripe.net/ripe/stats/)

Автоматически формируемый список IP‑адресов с выборкой 'RU' для **MikroTik RouterOS**.
Готовые файлы `.rsc` можно сразу импортировать и использовать в firewall, маршрутизации, QoS и других сценариях.

---

## ⚙️ Как это работает

1. **Источник данных** – официальная статистика RIPE NCC:  
   [ftp://ftp.ripe.net/ripe/stats/](ftp://ftp.ripe.net/ripe/stats/)

2. **Фильтрация** – выбираются только IPv4‑блоки, принадлежащие стране `RU`.

3. **Вычисление CIDR** – на основе диапазона и количества адресов вычисляется префикс:  
   `префикс = 32 - log₂(количество_адресов)` с округлением вниз (`int`).  

Используется следующая bash‑команда (запускается из crontab):

```bash
#!/bin/bash

inputFile="/dev/shm/delegated-ripencc-yesterday"
outputFile="/dev/shm/RIPE-RU-$(date -d "yesterday" +%d-%b-%Y).rsc"

wget -c -O - https://ftp.ripe.net/ripe/stats/`date -d "yesterday" "+%Y"`/delegated-ripencc-`date -d "yesterday" "+%Y%m%d"`.bz2 -T 10 |  bzcat | grep -i   "|RU|ipv4|" | awk -F '|' '{print $4 "/" (32 - log($5)/log(2))}' > "$inputFile"

if [ ! -s "$inputFile" ]; then
    exit 1
fi

content="/ip firewall address-list"

while IFS= read -r line; do
    if [[ $line =~ ^([^/]+)/([0-9]+) ]]; then
        ip="${BASH_REMATCH[1]}"
        mask="${BASH_REMATCH[2]}"       
        mask_floor=${mask%.*}
        content+=$'\n'"add address=${ip}/${mask_floor} list=RU timeout=35w"
    fi
done < "$inputFile"

printf "%s" "$content" > "$outputFile"

# Публикация $outputFile в GitHub

rm -f "$inputFile"
rm -f "$outputFile"
```
## 💾 Динамические адрес-листы и долговечность Flash

По умолчанию команды `/ip firewall address-list add ...` создают **статические** записи.  
Каждое такое добавление немедленно сохраняется на flash‑память роутера.  
Для больших списков это вызывает интенсивную запись и может сократить срок службы накопителя.

**Решение – динамические адрес-листы с параметром `timeout`.**  
Динамические записи хранятся только в оперативной памяти и не пишутся во flash, пока не истечёт таймаут.  
Максимально возможное значение `timeout` в RouterOS – **35 недель** (`35w`), что составляет около 8 месяцев.  
Этого более чем достаточно для ежедневной работы. Запись можно обновлять самостоятельно или автоматически при перезагрузке роутера.

## 🚀 Пример использования

Удаление старого списка, загрузка и импортно нового списка:
```routeros
/ip firewall address-list remove [find where list=RU]; \
/tool fetch url=https://raw.githubusercontent.com/pavmiro/routeros-ru-address-list/refs/heads/main/RIPE-RU-latest.rsc check-certificate=yes mode=https; \
/import RIPE-RU-latest.rsc
```
Автоматическое скачивание и импорт списка при загрузке роутера:
```routeros
/system scheduler
add name=schedule_RU start-time=startup on-event="/delay 15\r\
    \n/tool fetch url=https://raw.githubusercontent.com/pavmiro/routeros-ru-address-list/refs/heads/main/RIPE-RU-latest.rsc check-certificate=yes mode=https\r\
    \n/import RIPE-RU-latest.rsc"
```
