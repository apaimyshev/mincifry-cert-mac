# Если не заработало

## Политики нет в `chrome://policy`

Проверьте по порядку:

1. **Chrome был закрыт полностью?** Именно ⌘Q, а не крестик в окне. Закрытое окно — это ещё
   работающий Chrome со старыми политиками.
2. **Файл на месте и читается?**

   ```bash
   ls -l "/Library/Managed Preferences/com.google.Chrome.plist"
   plutil -lint "/Library/Managed Preferences/com.google.Chrome.plist"
   ```

   Права должны быть `-rw-r--r--`, владелец `root:wheel`.
3. **Версия Chrome 132 или новее?** Смотрите `chrome://version`. На более старых политика
   игнорируется молча — никакой ошибки вы не увидите.
4. **Кеш настроек не сбросился.** `sudo killall cfprefsd`, затем снова ⌘Q и запуск.
5. На самой странице `chrome://policy` нажмите **«Перезагрузить политики»**.

## Chrome написал «Управляется вашей организацией»

Так и должно быть: этот значок включает любая политика такого рода. Ничего лишнего у вас
не появилось — весь список применённых политик виден там же, на `chrome://policy`.

## Политика есть, статус OK, но сайт всё равно ругается

**Скорее всего, сертификат сайта выпущен не Минцифрой.** Проверьте:

```bash
openssl s_client -connect example.ru:443 -servername example.ru </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer
```

Если в `issuer` нет `Russian Trusted`, причина другая и список доменов ни при чём.

Другие варианты:

- **Домен указан не того уровня.** В списке должен быть домен второго уровня: `tbank.ru`.
  Поддомены подхватятся сами. А вот `www.tbank.ru` в списке **не** покроет `tbank.ru`.
- **Сайт на домене, которого нет в списке.** Посмотрите текущий список:
  `./scripts/add-mincifry --list`.
- **Другая ошибка сертификата.** `ERR_CERT_DATE_INVALID` — просрочен; `ERR_CERT_REVOKED` — отозван;
  `ERR_CERT_COMMON_NAME_INVALID` — имя не совпало. Доверие к корню тут не поможет.

### Как достать сертификат сайта из Chrome

На странице с ошибкой: значок слева от адреса → **Сведения о сертификате** → вкладка
**Details** → **Export**. Или прямо из терминала:

```bash
openssl s_client -connect example.ru:443 -servername example.ru -showcerts </dev/null
```

## Файл политики исчезает сам

На рабочем Mac с MDM каталогом `/Library/Managed Preferences` распоряжается система управления,
и ваш файл может затереться при очередной синхронизации. Проверить, есть ли MDM:

```bash
sudo profiles list -all
```

Если Mac корпоративный, идите в ИТ-отдел: ту же политику они раскатают профилем на всех сразу.

## Всё сломалось, хочу как было

```bash
sudo ./scripts/uninstall-mincifry
```

Резервные копии лежат рядом с файлом политики:

```bash
ls -l "/Library/Managed Preferences/"*.backup-*
```

Вернуть конкретную:

```bash
sudo cp "/Library/Managed Preferences/com.google.Chrome.plist.backup-ГГГГММДД-ЧЧММСС" \
        "/Library/Managed Preferences/com.google.Chrome.plist"
sudo killall cfprefsd
```

## Ничего из этого не помогло

Заведите issue и приложите:

- вывод `sw_vers` и версию браузера с `chrome://version`;
- `plutil -p "/Library/Managed Preferences/com.google.Chrome.plist"` — **без строки `certificate`**,
  она длинная и бесполезная для диагностики;
- точный текст ошибки из браузера (`NET::ERR_…`);
- `issuer` сертификата проблемного сайта (команда выше).
