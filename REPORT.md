## Проблема 1

### Что было обнаружено

Сервис постоянно перезапускался с ошибкой `203/EXEC`.

### Как проводилась диагностика

```bash
systemctl status homework-app.service
systemctl cat homework-app.service
systemctl show homework-app.service -p ExecStart
ls -l /opt/linux-devops-homework
```

В ExecStart был указан app.py, а в каталоге лежал только server.py.

### В чём была причина

Systemd пытался запустить несуществующий файл `/opt/linux-devops-homework/app.py`.

### Что было изменено

Сначала в `systemd/homework-app.service` указал найденный файл:

```ini
ExecStart=/usr/bin/python3 /opt/linux-devops-homework/server.py
```

Применение unit:

```bash
sudo cp systemd/homework-app.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение

Нужно было указать существующий файл приложения. 

### Как проверялся результат

```bash
systemctl status homework-app.service
```

Ошибка изменилась на `1/FAILURE`: питон запустился, но приложение падало уже по другой причине.

---

## Проблема 2

### Что было обнаружено

После исправления пути приложение не могло записать startup.log:

```text
PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'
```

### Как проводилась диагностика

```bash
journalctl -u homework-app.service -n 30
namei -l /var/lib/linux-devops-homework
ls -ld /var/lib/linux-devops-homework
id homework
```

Каталог принадлежал root:root и имел права 700. Родительские каталоги были доступны, но сам каталог данных - только для рута.

### В чём была причина

Сервис запускался от homework, поэтому не мог создать файл в чужом закрытом каталоге.

### Что было изменено

```bash
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 750 /var/lib/linux-devops-homework
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение

Владельцу нужна возможность создавать файлы в каталоге. homework:homework и права 750 прямо указаны в задании, открывать доступ всем не надл

### Как проверялся результат

```bash
ls -ld /var/lib/linux-devops-homework
ls -l /var/lib/linux-devops-homework/startup.log
systemctl status homework-app.service
```

Владелец изменился на homework, появился startup.log. Сервис стал `active (running)`.

---

## Проблема 3

### Что было обнаружено

check.sh выдал 6 успешных проверок из 10. 2 ошибки относились к пути запуска, хотя сервис уже работал. Ещё 2 к адресу прослушивания.

### Как проводилась диагностика

```bash
sudo ./scripts/check.sh
ls -l /opt/linux-devops-homework
grep server.py scripts/setup.sh scripts/check.sh
```

Установщик и проверка использовали разные пути к приложению.

### В чём была причина

setup.sh устанавливает файл в `/opt/linux-devops-homework/server.py`, а check.sh ожидает `/opt/linux-devops-homework/app/server.py`.

### Что было изменено

Создал каталог app и скопировал туда приложение:

```bash
sudo mkdir -p /opt/linux-devops-homework/app
sudo chmod 755 /opt/linux-devops-homework/app
sudo cp app/server.py /opt/linux-devops-homework/app/server.py
sudo chmod 755 /opt/linux-devops-homework/app/server.py
```

В `systemd/homework-app.service` записал окончательный путь:

```ini
ExecStart=/opt/linux-devops-homework/app/server.py
```

```bash
sudo cp systemd/homework-app.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение

Скрипты задания менять нельзя. Поэтому установил приложение по ожидаемому пути и указал его в unit

### Как проверялся результат

```bash
systemctl show homework-app.service -p ExecStart
systemctl status homework-app.service
sudo ./scripts/check.sh
```

Путь стал правильным, сервис запустился. Повторный запуск check.sh дал 8/10. Остались только две ошибки, связанные с адресом прослушивания.

---

## Проблема 4

### Что было обнаружено

Приложение работало, но слушало только `127.0.0.1:8080`.

### Как проводилась диагностика

```bash
ss -lntp
cat /etc/linux-devops-homework/app.conf
cat config/app.conf
```

В ss был адрес 127.0.0.1:8080, в обоих конфигах - `host = 127.0.0.1`.

### В чём была причина

Адрес в конфигурации ограничивал приложение loopback-интерфейсом. Успешный запрос к localhost не означал, что сервис слушает остальные интерфейсы.

### Что было изменено

В `config/app.conf` заменил адрес на `host = 0.0.0.0`. Порт оставил 8080.

```bash
sudo cp config/app.conf /etc/linux-devops-homework/
sudo systemctl restart homework-app.service
sudo systemctl enable homework-app.service
```

### Почему было выбрано это решение

127.0.0.1 доступен только с самой машины, а 0.0.0.0 означает прослушивание всех IPv4-интерфейсов, как требует задание

### Как проверялся результат

```bash
ss -lntp
curl -i http://127.0.0.1:8080/health
systemctl is-enabled homework-app.service
```

ss показал 0.0.0.0:8080, curl вернул HTTP 200 и `{"status": "ok"}`. Автозапуск включён.

---



## Итоговый ход диагностики

После новой установки сервис падал с 203/EXEC. Исправил путь, затем по ошибке в журнале исправил владельца и права каталога данных. Сервис заработал, но check.sh дал 6 из 10: не подходили путь и адрес прослушивания. Установил приложение по ожидаемому пути - стало 8 из 10. После изменения адреса на 0.0.0.0 прошли все 10 проверок

Unit прошёл systemd-analyze verify. Через cmp проверил совпадение итоговых unit и конфигурации с установленными в системе файлами. Каталог данных принадлежит homework:homework, права - 750. Сервис работает и включён в автозапуск.

## Итоговая проверка

```bash
sudo ./scripts/check.sh
```

```text
Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```
