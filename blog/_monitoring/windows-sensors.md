---
layout: post
sitemap:
  lastmod: 2023-07-27
title: мониторинг - сенсоры windows
descr: сенсоры windows - температура, скорость вентилятора.
keywords: sensors, windows
---

#### Предистория

В данной статье речь пойдет о получении данных с сеносров в прометеии.
Есть [гуишная версия](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor).
Но прометей неможет "смотреть" на гуи.
Поэтому люди написали сервис [hardware-supervisor](https://github.com/darkbrain-fc/HardwareSupervisor).
Использует библиотеку с устройствами из гуишной версии и размещает данные в WMI.

#### Установка

Перед установкой нужно убедится, что `net frameworks 4.7.2` или более свежая версия уже установлены.
Для этого нужно скачать его и попробовать поставить.

Далее скачиваем сам `hardware-supervisor`. Открываем `cmd.exe` от администратора,
что позволит создать сервис от пользователя `система`
(сервис будет работать без входа пользователя, да и запрос некоторых сенсоров требует высоких привилегий),
и выполняем команду:
```
msiexec /i HardwareSupervisorSetup.msi INSTALLFOLDER="C:\hardware-supervisor"
```

где `INSTALLFOLDER` - указывает в какую директорию необходимо поставить программу.

#### Сбор данных из WMI

Примеры запросов в повершелле:
```
Get-WmiObject -namespace "root/HardwareSupervisor" `
		-Class Hardware -Filter "HardwareType = 'Storage'"
```
Вернёт названия жестких дисков.


```
Get-WmiObject -namespace "root/HardwareSupervisor" `
		-Class Sensor -Filter "SensorType = 'Temperature'"
```
Вернёт температуру устройств, включая CPU, HDD/SSD/NVME, а так же сенсоры с материнской платы

```
Get-WmiObject -namespace "root/HardwareSupervisor" `
		-Class Sensor -Filter "SensorType = 'Fan' AND Min > 0"
```
Вернет информацию о вентиляторах. По поводу `Min > 0` - на материнской плате архитектурно
заложено подключение нескольких вентиялторов, но не все из них подключены или даже распаяны.
Обычно когда поключен вентилятор, то `Min` становится 10.

