---
title: Настраиваем сервер исходящей почты для отправки уведомлений
slug: send-only-mail-server
date: 2024-05-18T17:05:01+03:00
---

Моя ситуация: есть домашний сервер, на котором настроено несколько десятков Prometheus-алертов + cron job'ы, рассылающие уведомления на почту. Хочется, чтобы все эти уведомления попадали на мою Gmail-почту и не помечались как спам.

Лет десять назад я себе это всё как-то настроил, но с тех пор прошло слишком много времени – у почтовых сервисов появился ряд требований, которым должен удовлетворять сервер исходящей почты, поэтому пришло время обновить свою конфигурацию, чтобы она соответствовала современным стандартам. Поделюсь набором шагов, как это можно сделать.

На всякий случай сразу оговорюсь, что почти всё написанное ниже подразумевает, что у вас есть свой личный домен, с которого вы будете отправлять почту, т. к. ни один уважающий себя почтовый сервис не будет серьёзно относиться к письмам, отправленным с условного localhost'а, и будет (хотя бы периодически) расценивать их как спам. Домен стоит не так уж и дорого – конкретно в моём случае это 250 руб. в год, что является вполне приемлемой суммой даже для домашнего сервера.

Настройка ниже предполагает, что мы хотим только отсылать почту, но не принимать её извне. В случае рассылки алертов это абсолютно не нужно, а получение почты извне добавляет массу ненужной сложности (к примеру, борьбу со спамом). И, как правило, если мы действительно хотим получать письма извне, то самым лучшим вариантом тут будет воспользоваться почтой для домена от одного из почтовых сервисов, т. к., как минимум, вместе с ней у вас будет удобный web-интерфейс и вполне сносная защита от спама. При этом такая почта для домена может без проблем сосуществовать с приведённой ниже конфигурацией: живые люди будут пользоваться её web-интерфейсом, а сервисы на вашем сервере – рассылать алерты через настроенный почтовый сервер.

Пример настройки будет описан для Ubuntu, но завязок на конкретный дистрибутив, как таковых, не будет. В качестве имени домена везде ниже буду использовать example.com.

## Postfix

В качестве почтового сервера будем использовать [Postfix](https://www.postfix.org/). Рекомендую почитать на сайте их документацию – она достаточно подробная и даёт понимание, как оно работает под капотом.

Итак, ставим пакет:
```bash
sudo apt install postfix
```
В случае с Debian/Ubuntu он запустит интерактивный скрипт `dpkg-preconfigure`, который сгенерит нам базовую конфигурацию. В отобразившемся диалоге выбираем "Internet Site" и вводим имя своего домена. Особого смысла в этой конфигурации для нас нет, т. к. мы всё равно ниже всё переконфигурим, и пожалуй единственное что от неё останется – это Debian-specific файл `/etc/mailname`, в который будет прописано имя нашего домена, чтобы его могли использовать почтовые клиенты.

У Postfix два основных конфигурационных файла:
* `/etc/postfix/master.cf`, который описывает, как именно будут запускаться различные демоны, из которых состоит почтовый сервер. Его лучше не трогать и менять только тогда, когда вы действительно знаете, что делаете.
* `/etc/postfix/main.cf` – в нём будут храниться все основные настройки.

Заменяем `/etc/postfix/main.cf` следующим содержимым:
```cfg
# man 5 postconf

# See http://www.postfix.org/COMPATIBILITY_README.html
# Restart postfix server and look into /var/log/mail.log for complains about new defaults
compatibility_level = 3.8

# Used in default values of many variables. Specifies what domain to use for outbound mail
myhostname = example.com
mydomain = $myhostname

# Domains which will be delivered locally instead of forwarding to another machine
mydestination = localhost $myhostname

# Rewrite (possibly local) sender domain to our external domain
sender_canonical_maps = static:@$myorigin

# Interfaces to listen on
inet_interfaces = loopback-only

# Forward mail from local host only
mynetworks_style = host

# Redefine the default to disable NIS support
alias_maps = hash:/etc/aliases

# A little hardening
allow_mail_to_files =
allow_mail_to_commands =

# Relay mail only via TLS. Since it's too strict setting for generic server which should be able to send mail to any
# server on the Internet, it's reasonable when we send messages only to well-known servers like Gmail.
smtp_tls_security_level = verify
smtp_tls_CApath=/etc/ssl/certs

# Generate delayed mail warnings
delay_warning_time = 4h

# Errors to notify postmaster@ about
notify_classes = 2bounce, data, delay, policy, protocol, resource, software
```

Здесь необходимо прописать в `myhostname` ваш домен, а также перечислить в `mydestination` все домены, с которых локальные сервисы могут прислать почту.

Логика тут следующая: если в `To:` письма указано только имя пользователя (скажем `dmitry`), то Postfix автоматом понимает, что это письмо предназначено для нашего домена (локальной доставки), но если в качестве имени пользователя указано, скажем, `dmitry@server`, то необходимо проинструктировать Postfix, что `server` – это на самом деле тоже наш домен. Необходимость в этом возникает, к примеру, когда hostname вашего сервера (значение в `/etc/hostname`) отличается от почтового домена, и неправильно настроенная программа отправки почты может генерировать `To`-адреса используя это имя сервера.

Далее заводим пользователя `postmaster`:
```bash
sudo useradd -d /nonexistent -s /usr/sbin/nologin -r postmaster
```
Это well-known имя пользователя, которому Postfix будет посылать сообщения о различных ошибках (например, доставки почты).

Заменяем `/etc/aliases` следущим содержимым:
```cfg
# man 5 aliases

root:   example@gmail.com
dmitry: example@gmail.com
```

Логика тут следующая: у нас есть два активных пользователя – `root` и `dmitry`, которым система может слать почтовые сообщения. В моём случае это преимущественно `cron`, который отсылает stdout/stderr своих джоб пользователю по почте, но, к примеру, также их шлёт `sudo`, если 3 раза неправильно ввести пароль. Данные директивы говорят Postfix о том, что необходимо принимать почту для этих двух пользователей не локально, а пересылать её на наш Gmail-ящик.

Если же Postfix не сможет доставить почту по данным адресам (Gmail отклонит её по какой-то причине), то информация об этом будет направлена пользователю `postmaster`.

Несмотря на то, что в `main.cf` у нас прописан `/etc/aliases`, Postfix будет считывать файл `/etc/aliases.db`, который является скомпилированным бинарным представлением этого конфига. Поэтому после каждой правки `/etc/aliases` необходимо запускать `sudo newaliases`, чтобы она создала `/etc/aliases.db`.

На этом базовая настройка закончена. Перезапускаем сервер и смотрим в его лог:
```bash
sudo systemctl restart postfix && sudo tail -f /var/log/mail.log
```

### Проверка пересылки

Проверяем:
```bash
sudo apt install mutt
mutt -s 'Test mail' "$(whoami)" <<< 'Message body'
```
– письмо должно уйти на наш Gmail-аккаунт.

### Проверка локальной доставки

Чтобы быть в курсе всех проблем с отправкой почты, я выставил пользователю `root` переменную окружения `MAIL=/var/mail/postmaster`: своего ящика у него всё равно больше нет (вся почта перенаправляется в Gmail), но зато теперь bash будет автоматически уведомлять меня о всех новых сообщениях пользователя `postmaster`.

Проверяем работу данной схемы:
1. Логинимся под `root`.
2. От любого пользователя запускаем `mutt -s 'Test mail' postmaster <<< 'Message body'`.
3. root'овый bash должен написать в терминал `You have mail in /var/mail/postmaster`.
4. Смотрим почту (к примеру, при помощи того же `mutt`).

## SPF, DKIM, DMARK

Базовая настройка завершена, почта успешно уходит в Gmail, но с большой вероятностью будет попадать в спам. Всё потому, что наш сервер не удовлетворяет [требованиям Gmail](https://support.google.com/a/answer/81126).

Рекомендую почитать какой-нибудь tutorial по SPF/DKIM/DMARK, о которых пойдёт речь ниже, чтобы было понимание, что именно мы будем делать. К примеру, вот [этот](https://dmarcly.com/blog/how-to-implement-dmarc-dkim-spf-to-stop-email-spoofing-phishing-the-definitive-guide).

### PTR

Первым делом стоит удостовериться, что у вашего сервера правильная [PTR-запись](https://en.wikipedia.org/wiki/Reverse_DNS_lookup). Не всегда есть возможность её поменять – у меня, к примеру, в силу определённых причин её нету, и мне (пока?) это не мешает, но почтовые сервисы на неё точно смотрят, и желательно, чтобы она указывала на ваш домен.

### SPF

С помощью [SPF](https://en.wikipedia.org/wiki/Sender_Policy_Framework) мы определим список IP, с которых можно осылать письма от имени нашего домена.

Добавляем следующую DNS-запись:
```
example.com. IN TXT "v=spf1 a ~all"
```
которая разрешает отправлять их только с тех IP, в которые резолвится наш `example.com`. Я использую `~all` вместо `-all`, т. к. в моём случае вряд ли кто-то будет заниматься спуфингом от моего имени, и в случае какой-то мисконфигурации я бы предпочёл получить свои письма в спам, чем не получить вовсе.

Если вам хочется задать другие правила, то можно воспользоваться [SPF-генератором](https://dmarcly.com/tools/spf-record-generator).

### DKIM

[DKIM](https://en.wikipedia.org/wiki/DomainKeys_Identified_Mail) – это механизм подписи всех исходящих писем, с помощью которого получающая сторона может убедиться, что данные письма были присланы именно нашим сервером.

Реализовывать его будем с помощью [OpenDKIM](http://www.opendkim.org/).

Ставим пакет:
```bash
sudo apt install opendkim
```

Генерируем ключи:
```bash
sudo opendkim-genkey --domain example.com --selector server --nosubdomains --restrict --directory /etc/dkimkeys
```

`server` здесь – это DKIM selector – произвольный идентификатор ключа. Нужен он потому, что их может быть несколько: у каждого сервера по ключу, либо в целях периодической ротации ключей.

Смотрим в `/etc/dkimkeys/server.txt` и прописываем получившуюся запись в DNS. Тут, правда, вас может ждать неудача: запись получается довольно длинная, и некоторые хостеры (к примеру, мой) просто отказываются её принимать. В таком случае можно сгенерировать чуть менее безопасный ключ, который будет меньшего размера:

```bash
sudo opendkim-genkey --domain example.com --selector server --bits 1024 --nosubdomains --directory /etc/dkimkeys
```

Проверяем, что всё прописалось правильно:
```bash
sudo opendkim-testkey -d example.com -s server -k /etc/dkimkeys/server.private
```

Меняем права на ключи:
```bash
sudo chown opendkim:opendkim /etc/dkimkeys/server.{private,txt}
```

и начинаем конфигурировать сам сервис.

В Debian/Ubuntu Postfix запускается в chroot'е, и если UNIX-сокет OpenDKIM-сервера будет находиться вне его, то они просто не смогут общаться между собой. Поэтому содаём следующую директорию:

```bash
sudo install -d -o opendkim -g postfix -m 710 /var/spool/postfix/opendkim
sudo chmod g+s /var/spool/postfix/opendkim
```

Создаём `/etc/systemd/system/opendkim.service` (конфигурация сервиса, которая поставляется с дистрибутивом, довольно странная – и лучше написать свою):
```systemd
[Unit]
Description=OpenDKIM Milter
Before=postfix.service

[Service]
User=opendkim
NoNewPrivileges=true
MemoryDenyWriteExecute=true
ProtectSystem=true
PrivateTmp=true
ExecStart=/usr/sbin/opendkim

[Install]
WantedBy=multi-user.target
```

Заменяем `/etc/opendkim.conf` на:
```cfg
Background no

Syslog yes
SyslogSuccess yes

Mode s
Domain example.com
Selector server
KeyFile /etc/dkimkeys/server.private

# Postfix chroots into /var/spool/postfix, so we have to create SGID directory there and use permissive umask
Socket local:/var/spool/postfix/opendkim/opendkim.sock
UMask 007
```

Добавляем в `/etc/postfix/main.cf`:
```cfg
# DKIM signing
smtpd_milters = unix:opendkim/opendkim.sock
non_smtpd_milters = unix:opendkim/opendkim.sock
```

Проверяем, работоспособность всей схемы:
```bash
sudo systemctl daemon-reload && sudo systemctl restart opendkim postfix && sudo tail -f /var/log/mail.log
mutt -s 'Test mail' "$(whoami)" <<< 'Message body'
```

### DMARK

Ну и последний штрих, который нам остался – это [DMARK](https://en.wikipedia.org/wiki/DMARC). Воспользуемся [DMARK-генератором](https://dmarcly.com/tools/dmarc-generator) и получим следующую DNS-запись:

```
_dmarc IN TXT "v=DMARC1; p=quarantine; sp=none; aspf=s; adkim=s;"
```

Как и с SPF, для наших целей лучше подойдёт `p=quarantine`.

Email для aggregate/forensic reports не указываем, т. к. это опять-таки не наш случай:
* Aggregate Reports – это ежедневная рассылка XML-файлов, которая имеет смысл только если вы – какая-то большая организация и перенаправляете их в соответствующий сервис для последующего анализа.
* Forensic Reports и вовсе не поддерживаются в Gmail.

## Заключительные шаги

После того, как мы всё настроили, самое время пойти посмотреть, что про наши письма думают почтовые сервисы:

1. Нажимаем на вертикальное троеточие в сообщении в Gmail и выбираем пункт "Show original" – там в самом верху будут статусы вида "SPF/DKIM/DMARC: PASS", а также проверяем, что само сообщение не помечается красным флажком "No encryption" (в настройках Postfix мы указали отправку только по защищённому каналу). Есть ещё [Google Postmaster Tools](https://postmaster.google.com/), но они выглядят какими-то полузаброшенными: дашборд с графиками объявлен устаревшим и упорно отказывается показывать хоть какие-то графики даже после того, как я в течение недели слал себе каждую минуту какие-то сообщения (утверждается, что они могут не отображаться при небольшом количестве сообщений); в Compliance status в итоге все зелёные кружочки у меня загорелись, но только спустя неделю.
2. Заходим на [mail-tester.com](https://www.mail-tester.com/) и отправляем сообщение на адрес, который он нам предложит, используя команду вида `mutt -s 'Test mail' test-m2rbwe4su@srv1.mail-tester.com <<< 'Message body'`. Высокого рейтинга нам в нём ждать не стоит: он проверяет содержимое сообщения на спам (и тогда надо отсылать что-то реальное, чтобы не поджечь эти лампочки), а также у нас нет `MX`-записи для входящей почты, что в его понимании выглядит крайне подозрительным. Но наша цель тут – не попытаться получить максимальный рейтинг по всем параметрам, а убедиться, что его устраивает всё, что касается только отправки писем.

В итоге, имеем конфигурацию, которой Gmail полностью доволен, и если раньше в случае резкого всплеска сообщений от какой-то сломавшейся cron'ячки он начинал периодически помещать их в спам, то теперь, сколько бы тестовых сообщений я не пытался отправить, такого не происходит.