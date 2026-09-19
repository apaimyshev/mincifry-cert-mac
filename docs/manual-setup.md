# Ручная установка, без скриптов

Если не хочется запускать чужие скрипты под `sudo` — вот то же самое руками.
Заодно понятнее, что вообще происходит.

## 1. Возьмите корневой сертификат

Он лежит в репозитории: `certs/russian_trusted_root_ca.pem`. Официальный источник —
Госуслуги, https://www.gosuslugi.ru/crt (файл `russian_trusted_root_ca.cer`).

Сверьте отпечаток — он должен быть таким:

```bash
openssl x509 -in russian_trusted_root_ca.pem -noout -fingerprint -sha256
# SHA256 Fingerprint=D2:6D:2D:02:31:B7:C3:9F:92:CC:73:85:12:BA:54:10:35:19:E4:40:5D:68:B5:BD:70:3E:97:88:CA:8E:CF:31
```

Если файл скачался в формате DER (`.cer`, `.crt`), переведите в PEM:

```bash
openssl x509 -inform DER -in russian_trusted_root_ca.cer -out russian_trusted_root_ca.pem
```

## 2. Получите сертификат в base64

Политика хранит сертификат как base64 от DER, одной строкой:

```bash
openssl x509 -in russian_trusted_root_ca.pem -outform DER | base64 | tr -d '\n'
```

## 3. Соберите файл политики

Создайте `com.google.Chrome.plist` такого вида (`<САМАЯ_ДЛИННАЯ_СТРОКА>` — вывод из шага 2,
домены — свои):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CACertificatesWithConstraints</key>
  <array>
    <dict>
      <key>certificate</key>
      <string>MIIFwjCCA6qgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAw…</string>
      <key>constraints</key>
      <dict>
        <key>permitted_dns_names</key>
        <array>
          <string>sberbank.ru</string>
          <string>tochka.com</string>
          <string>gosuslugi.ru</string>
        </array>
      </dict>
    </dict>
  </array>
</dict>
</plist>
```

Проверьте синтаксис:

```bash
plutil -lint com.google.Chrome.plist
```

## 4. Положите на место

```bash
sudo install -d -o root -g wheel -m 755 "/Library/Managed Preferences"
sudo install -o root -g wheel -m 644 com.google.Chrome.plist "/Library/Managed Preferences/com.google.Chrome.plist"
sudo killall cfprefsd
```

> **Осторожно:** если файл `/Library/Managed Preferences/com.google.Chrome.plist` уже существует,
> вы затрёте политики, которые там были. Сначала посмотрите, что внутри
> (`plutil -p "/Library/Managed Preferences/com.google.Chrome.plist"`), и при необходимости
> допишите свой ключ в существующий файл, а не заменяйте его целиком.

Это каталог «управляемых настроек» macOS. Обычно его наполняет MDM, но система читает оттуда
всё, что положено root, — отдельный MDM-сервер для этого не нужен.

## 5. Перезапустите браузер

⌘Q, затем запустить заново. Без полного выхода политика не перечитается.

Проверьте на `chrome://policy`: строка `CACertificatesWithConstraints`, источник **Platform**,
статус **OK**.

## Добавить домен потом

```bash
sudo plutil -insert CACertificatesWithConstraints.0.constraints.permitted_dns_names.0 \
  -string "ozonbank.ru" "/Library/Managed Preferences/com.google.Chrome.plist" && sudo killall cfprefsd
```

Посмотреть текущий список:

```bash
plutil -extract CACertificatesWithConstraints.0.constraints.permitted_dns_names json -o - \
  "/Library/Managed Preferences/com.google.Chrome.plist"
```

## Удалить всё

```bash
sudo rm "/Library/Managed Preferences/com.google.Chrome.plist" && sudo killall cfprefsd
```

Если в файле были и другие политики — удаляйте только свой ключ:

```bash
sudo plutil -remove CACertificatesWithConstraints "/Library/Managed Preferences/com.google.Chrome.plist"
```
