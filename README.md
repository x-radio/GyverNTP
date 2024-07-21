[![latest](https://img.shields.io/github/v/release/GyverLibs/GyverNTP.svg?color=brightgreen)](https://github.com/GyverLibs/GyverNTP/releases/latest/download/GyverNTP.zip)
[![PIO](https://badges.registry.platformio.org/packages/gyverlibs/library/GyverNTP.svg)](https://registry.platformio.org/libraries/gyverlibs/GyverNTP)
[![Foo](https://img.shields.io/badge/Website-AlexGyver.ru-blue.svg?style=flat-square)](https://alexgyver.ru/)
[![Foo](https://img.shields.io/badge/%E2%82%BD%24%E2%82%AC%20%D0%9F%D0%BE%D0%B4%D0%B4%D0%B5%D1%80%D0%B6%D0%B0%D1%82%D1%8C-%D0%B0%D0%B2%D1%82%D0%BE%D1%80%D0%B0-orange.svg?style=flat-square)](https://alexgyver.ru/support_alex/)
[![Foo](https://img.shields.io/badge/README-ENGLISH-blueviolet.svg?style=flat-square)](https://github-com.translate.goog/GyverLibs/GyverNTP?_x_tr_sl=ru&_x_tr_tl=en)  

[![Foo](https://img.shields.io/badge/ПОДПИСАТЬСЯ-НА%20ОБНОВЛЕНИЯ-brightgreen.svg?style=social&logo=telegram&color=blue)](https://t.me/GyverLibs)

# GyverNTP
Библиотека для получения точного времени с NTP сервера для esp8266/esp32
- Работает на стандартной библиотеке WiFiUdp.h
- Учёт времени ответа сервера и задержки соединения
- Получение времени с точностью до миллисекунд
- Интеграция с библиотекой [Stamp](https://github.com/GyverLibs/Stamp) для распаковки unix в часы, минуты итд.
- Автоматическая синхронизация
- Поддержание хода времени на базе millis() между синхронизациями
- Секундный таймер для удобства автоматизации
- Обработка ошибок
- Асинхронный режим

### Совместимость
esp8266, esp32

### Зависимости
- [Stamp](https://github.com/GyverLibs/Stamp)

## Содержание
- [Инициализация](#init)
- [Использование](#usage)
- [Пример](#example)
- [Версии](#versions)
- [Установка](#install)
- [Баги и обратная связь](#feedback)

<a id="init"></a>

## Инициализация
```cpp
GyverNTP ntp;                 // параметры по умолчанию (gmt 0, период 3600 секунд (1 час))
GyverNTP ntp(gmt);            // часовой пояс в часах (например Москва 3)
GyverNTP ntp(gmt, period);    // часовой пояс в часах и период обновления в секундах
```

<a id="usage"></a>

## Использование
```cpp
// Наследует StampTicker

// установить часовой пояс в часах или минутах
void setGMT(int16_t gmt);

// установить период обновления в секундах
void setPeriod(uint16_t prd);

// установить хост (умолч. "pool.ntp.org")
void setHost(const String& host);

// запустить
bool begin();

// остановить
void end();

// включить асинхронный режим (по умолч. true)
void asyncMode(bool async);

// получить пинг сервера, мс
int16_t ping();

// не учитывать пинг соединения (умолч. false)
void ignorePing(bool ignore);

// вернёт true, если tick ожидает ответа сервера в асинхронном режиме
bool busy();

// получить статус последнего действия
Status status();

// true - не было ошибок связи и есть соединение с Интернет
bool online();

// ============== ТИКЕР ===============
// тикер, обновляет время по своему таймеру. Вернёт true при смене статуса
bool tick();

// вручную запросить и обновить время с сервера. true при успехе
bool updateNow();
```

## Особенности
- Нужно вызывать `tick()` в главном цикле программы `loop()`, он синхронизирует время с сервера по своему таймеру и обеспечивает работу секундного таймера
- Если основной цикл программы сильно загружен, а время нужно получать с максимальной точностью (несколько мс), то можно выключить асинхронный режим `asyncMode(false)`
- Библиотека продолжает считать время после пропадания синхронизации. По моим тестам esp "уходит" на ~1.7 секунды за сутки, поэтому стандартный период синхронизации выбран 1 час
- Наследуется класс StampTicker, который обеспечивает счёт времени и работу секундного таймера

### Секундный таймер
Для удобства автоматизации событий по таймеру в библиотеку встроен секундный таймер, он срабатывает в 0 миллисекунд каждой секунды. По условию таймера NTP гарантированно синхронизирован и выдаёт корректное время:

```cpp
void loop() {
  ntp.tick();

  if (ntp) Serial.println(ntp.toString());  // короткая запись
  // if (ntp.newSecond()) Serial.println(ntp.toString()); // или так
}
```

### Рассинхронизация
Если период синхронизации очень большой или в системе надолго пропадает связь, часы рассинхронизируются и будут синхронизированы при следующем обращении к серверу. Если время ушло больше, чем на 1 секунду, то поведение будет следующим:
- Если внутренние часы "спешат" - секундный таймер перестанет срабатывать, пока реальное время не догонит внутреннее
- Если внутренние часы "отстают" - таймер будет вызываться каждую итерацию loop с прибавлением времени, пока внутреннее время не догонит реальное

Это сделано для того, чтобы при синхронизации не потерялись секунды - библиотека обработает каждую секунду и не будет повторяться, что очень важно для алгоритмов автоматизации.

### Проверка онлайна
NTP работает по UDP - очень легковесному и "дешёвому" протоколу связи, обращение к серверу правтически не занимает времени. Благодаря этому NTP можно использовать для проверки связи с Интернет - там, где стандартный TCP клиент зависнет на несколько секунд, NTP асинхронно сообщит о потере связи. В рамках GyverNTP это можно использовать так:
```cpp
GyverNTP ntp(3, 5000);  // синхронизация каждые 5 секунд

// ...

void loop() {
  // вернёт true при смене статуса
  if (ntp.tick()) {
    Serial.println(ntp.online());
    // здесь флаг online можно использовать для передачи в другие библиотеки
    // например FastBot2
    // bot.setOnline(ntp.online());
  }
}
```

### Получение времени
GyverNTP наследует [StampCore](https://github.com/GyverLibs/Stamp?tab=readme-ov-file#stamp-1), то есть получать время можно множеством способов:
```cpp
if (ntp) {
  Serial.println(ntp.toString());     // вывод даты и времени строкой
  Serial.println(ntp.dateToString()); // вывод даты строкой

  // можно сравнивать напрямую с unix
  if (ntp >= 12345) { }

  ntp.getUnix();  // unix
  ntp.second();   // секунды
  ntp.minute();   // минуты и так далее

  // но эффективнее использовать парсер Datime
  Datime dt(ntp);  // ntp само конвертируется в unix

  dt.year;
  dt.month;
  dt.day;
  dt.hour;
  dt.minute;
  dt.second;
  dt.weekDay;
  dt.yearDay;
}
```

### GMT
Часовой пояс задаётся для всех операций со Stamp/Datime в программе! Установка часового пояса в объекте ntp равносильна вызову `setStampZone()` - установка глобального часового пояса для библиотеки Stamp

<a id="example"></a>

## Пример
```cpp
// пример выводит время каждую секунду
// а также два раза в секунду мигает светодиодом
// можно прошить на несколько плат - они будут мигать синхронно

#include <GyverNTP.h>
GyverNTP ntp(3); // Москва, GMT+3

// список серверов, если "pool.ntp.org" не работает
//"ntp1.stratum2.ru"
//"ntp2.stratum2.ru"
//"ntp.msk-ix.ru"

void setup() {
  Serial.begin(115200);
  WiFi.begin("WIFI_SSID", "WIFI_PASS");
  while (WiFi.status() != WL_CONNECTED) delay(100);
  Serial.println("Connected");

  ntp.begin();
  //ntp.asyncMode(false);   // выключить асинхронный режим
  //ntp.ignorePing(true);   // не учитывать пинг до сервера
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  ntp.tick();
  
  if (ntp.ms() == 0) {
    delay(1);
    digitalWrite(LED_BUILTIN, 1);
  }
  if (ntp.ms() == 500) {
    delay(1);
    digitalWrite(LED_BUILTIN, 0);
    Serial.println(ntp.toString());
    Serial.println();
  }
}
```

<a id="versions"></a>

## Версии
- v1.0
- v1.1 - мелкие улучшения и gmt в минутах
- v1.2 - оптимизация, улучшена стабильность, добавлен асинхронный режим
- v1.2.1 - изменён стандартный период обновления
- v1.3 - ускорена синхронизация при запуске в асинхронном режиме
- v1.3.1 - заинклудил WiFi библиотеку в файл
- v2.0 - добавлена зависимость от Stamp, больше возможностей, проверка онлайна для других библиотек

<a id="install"></a>

## Установка
- Библиотеку можно найти по названию **GyverNTP** и установить через менеджер библиотек в:
    - Arduino IDE
    - Arduino IDE v2
    - PlatformIO
- [Скачать библиотеку](https://github.com/GyverLibs/GyverNTP/archive/refs/heads/main.zip) .zip архивом для ручной установки:
    - Распаковать и положить в *C:\Program Files (x86)\Arduino\libraries* (Windows x64)
    - Распаковать и положить в *C:\Program Files\Arduino\libraries* (Windows x32)
    - Распаковать и положить в *Документы/Arduino/libraries/*
    - (Arduino IDE) автоматическая установка из .zip: *Скетч/Подключить библиотеку/Добавить .ZIP библиотеку…* и указать скачанный архив
- Читай более подробную инструкцию по установке библиотек [здесь](https://alexgyver.ru/arduino-first/#%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0_%D0%B1%D0%B8%D0%B1%D0%BB%D0%B8%D0%BE%D1%82%D0%B5%D0%BA)
### Обновление
- Рекомендую всегда обновлять библиотеку: в новых версиях исправляются ошибки и баги, а также проводится оптимизация и добавляются новые фичи
- Через менеджер библиотек IDE: найти библиотеку как при установке и нажать "Обновить"
- Вручную: **удалить папку со старой версией**, а затем положить на её место новую. "Замену" делать нельзя: иногда в новых версиях удаляются файлы, которые останутся при замене и могут привести к ошибкам!

<a id="feedback"></a>

## Баги и обратная связь
При нахождении багов создавайте **Issue**, а лучше сразу пишите на почту [alex@alexgyver.ru](mailto:alex@alexgyver.ru)  
Библиотека открыта для доработки и ваших **Pull Request**'ов!


При сообщении о багах или некорректной работе библиотеки нужно обязательно указывать:
- Версия библиотеки
- Какой используется МК
- Версия SDK (для ESP)
- Версия Arduino IDE
- Корректно ли работают ли встроенные примеры, в которых используются функции и конструкции, приводящие к багу в вашем коде
- Какой код загружался, какая работа от него ожидалась и как он работает в реальности
- В идеале приложить минимальный код, в котором наблюдается баг. Не полотно из тысячи строк, а минимальный код