# mincifry-cert-mac

Российские банки и госсервисы в Chrome на macOS — не меняя браузер и не доверяя
корню Минцифры весь остальной интернет.

## Проблема

Банк или Госуслуги не открываются: `NET::ERR_CERT_AUTHORITY_INVALID`. Сертификат выпущен
УЦ Минцифры (`Russian Trusted Root CA`), а его нет ни в Chrome Root Store, ни в системном
хранилище macOS.

Проверить, ваш ли это случай:

```bash
openssl s_client -connect ваш-банк.ru:443 -servername ваш-банк.ru </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer
```

Если в выводе есть `Russian Trusted` — да, ваш.

## Варианты

| | Яндекс.Браузер | Корень в keychain | Этот репозиторий |
|---|---|---|---|
| Где действует доверие | на всех сайтах | на всех сайтах | только на ваших доменах |
| Работает в Chrome | — | да | да |
| Работает в Safari | — | да | нет |
| Надо менять браузер | да | нет | нет |
| Как откатить | удалить браузер | удалить из keychain | удалить один файл |

Здесь работает политика Chromium [`CACertificatesWithConstraints`](https://learn.microsoft.com/en-us/deployedge/microsoft-edge-browser-policies/cacertificateswithconstraints)
(Chrome 132+, Edge 133+): браузер доверяет корню только на перечисленных доменах.
Системное хранилище macOS при этом не меняется.

## Установка

```bash
git clone https://github.com/apaimyshev/mincifry-cert-mac.git
cd mincifry-cert-mac
sudo ./scripts/install-mincifry
```

Потом **⌘Q в Chrome** (именно выход, а не закрытие окна) и запустить заново.

- `--dry-run` — показать, что будет сделано, ничего не меняя
- `--browser edge|brave|chromium|all` — другие браузеры
- `--domains свой.txt` — свой список доменов

Скрипт проверяет SHA-256 отпечаток сертификата. Если файл политики уже есть — сохраняет копию
и правит только свой ключ, остальные политики не трогает. В сеть не ходит, в keychain не лезет.

## Проверка

Откройте `chrome://policy` — там должна быть строка `CACertificatesWithConstraints`,
источник **Platform**, статус **OK**.

Надпись «Управляется вашей организацией» — это нормально, её показывает любая политика такого рода.

## Домены

По умолчанию их 33: Сбер, ВТБ, Альфа, Точка, Т-Банк, ГПБ, Райф, РСХБ, ПСБ, Открытие, МКБ,
Совкомбанк, Уралсиб, Росбанк, ДОМ.РФ, Почта Банк, Госуслуги, ФНС, ЦБ, СФР, Казначейство.
Весь список — в [`domains.txt`](domains.txt).

```bash
sudo ./scripts/add-mincifry ozonbank.ru          # добавить
     ./scripts/add-mincifry --list               # посмотреть
sudo ./scripts/add-mincifry --remove tbank.ru    # убрать
```

Можно вставлять прямо из адресной строки: `HTTPS://WWW.OzonBank.ru/x?y=1` превратится
в `ozonbank.ru`. Поддомены подхватываются сами — `sberbank.ru` закрывает и `online.sberbank.ru`.

Не добавляйте домены на всякий случай, только когда реально упёрлись в ошибку. Чем короче
список, тем меньше лишнего вы разрешили — ради этого всё и затевалось.

## Что не работает

**Safari** доверяет только системному хранилищу, а там нельзя ограничить доверие по доменам.
**Firefox** держит свои корни, политики Chromium на него не действуют. **На iPhone** нужен
профиль конфигурации. **curl и wget** ходят со своими наборами корней — разово лечится
флагом `--cacert`.

Подробнее — [docs/browsers.md](docs/browsers.md).

## Удаление

```bash
sudo ./scripts/uninstall-mincifry
```

Удаляет только свой ключ, остальные политики в файле остаются. Резервные копии лежат рядом.

## Безопасность

Вы разрешаете УЦ Минцифры выпускать сертификаты для доменов из списка. Свои домены
туда не добавляйте.

Сертификат лежит в `certs/russian_trusted_root_ca.pem`, его SHA-256 —
`D2:6D:2D:02:31:B7:C3:9F:92:CC:73:85:12:BA:54:10:35:19:E4:40:5D:68:B5:BD:70:3E:97:88:CA:8E:CF:31`.
Тот же файл лежит на [Госуслугах](https://www.gosuslugi.ru/crt). Проверить самому:

```bash
openssl x509 -in certs/russian_trusted_root_ca.pem -noout -fingerprint -sha256
```

И не запускайте чужие скрипты под `sudo`, не прочитав их. Эти тоже: здесь три файла на shell,
читаются за десять минут.

## Документация

- [docs/manual-setup.md](docs/manual-setup.md) — то же самое руками, без скриптов
- [docs/browsers.md](docs/browsers.md) — Safari, Firefox, Яндекс, Edge, терминал
- [docs/troubleshooting.md](docs/troubleshooting.md) — политика не подхватилась, сайт всё равно ругается
- [docs/ai-agents.md](docs/ai-agents.md) — если настройку делает ИИ-агент: где он ошибается и что с него спросить

На Windows та же политика живёт в реестре (`HKLM\SOFTWARE\Policies\Google\Chrome`),
на Linux — в JSON в `/etc/opt/chrome/policies/managed/`. Формат значения тот же.

## Лицензия

[MIT](LICENSE). Антон Паймышев — [Everypay](https://everypay.io),
[@chistogipoteticheski](https://t.me/chistogipoteticheski).

Не хватает домена в списке или что-то отвалилось на вашей версии macOS — заводите issue.
