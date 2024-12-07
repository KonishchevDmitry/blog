---
title: Cloudflare, ECH, или почему в последнее время у вас может не открываться часть зарубежных сайтов
slug: cloudflare-ech
date: 2024-12-07T16:16:27+03:00
---

На прошлой неделе я заметил, что у меня ни с того ни с сего перестала открываться в браузере часть зарубежных сайтов. Поначалу это не выглядело как массовое явление: хотел зайти на [prometheus.io/docs/prometheus](https://prometheus.io/docs/prometheus/) – а он не открывается по таймауту. Ну ладно, думаю – прилёг, с кем не бывает. День лежит, второй лежит – уже выглядит подозрительно. При этом в процессе гугления каких-то совсем других вещей заметил, что также подвисают чьи-то небольшие блоги.

Посмотрел повнимательнее – curl открывает сайты бодро, Firefox – тоже, а вот в Яндекс Браузере по прежнему таймаут соединения без каких-либо объяснений причин в сетевой вкладке Developer Tools.

Ну что ж, Developer Tools ничего не говорят – пойдёмте смотреть в [Wireshark](https://www.wireshark.org/), что там такого интересного происходит...


## SNI, ESNI, ECH, WTF?

Для начала краткий ликбез: когда мы устанавливаем HTTPS-соединение, то, несмотря на то, что весь HTTP-запрос у нас шифруется, имя домена в TLS handshake идёт plain text'ом, т. к. оно необходимо для того, чтобы сообщить балансировщику, какой именно сайт вы хотите открыть (и для какого домена балансировщик, терминирующий TLS, должен выдать вам сертификат). Данное расширение TLS, в рамках которого передаётся имя домена, называется [SNI (Server Name Indication)](https://www.cloudflare.com/learning/ssl/what-is-sni/).

Именно благодаря SNI РКН может селективно блокировать HTTPS-ресурсы не по IP-адресам (задевая при этом ещё кучу других сайтов), а по доменным именам.

Так вот, давайте посмотрим, что нам расскажет Wireshark, когда мы набираем [prometheus.io](https://prometheus.io/docs/prometheus/) в адресной строке браузера.

Видим два DNS-запроса:

```
DNS Standard query 0x6d3e A prometheus.io
DNS Standard query 0x169c HTTPS prometheus.io
```

1. Стандартная `A`-запись – тут всё понятно и никаких сюрпризов;
2. А вот вторая необычная – `HTTPS`-запись. Я о такой даже и не слышал – как интересно. :)

Вот ответ DNS-сервера:
```
prometheus.io: type HTTPS, class IN
    Name: prometheus.io
    Type: HTTPS (65) (HTTPS Specific Service Endpoints)
    Class: IN (0x0001)
    Time to live: 277 (4 minutes, 37 seconds)
    Data length: 136
    SvcPriority: 1
    TargetName: <Root>
    SvcParam: alpn=h3,h2
        SvcParamKey: alpn (1)
        SvcParamValue length: 6
        ALPN length: 2
        ALPN: h3
        ALPN length: 2
        ALPN: h2
    SvcParam: ipv4hint=104.21.60.220,172.67.201.240
        SvcParamKey: ipv4hint (4)
        SvcParamValue length: 8
        IP: 104.21.60.220
        IP: 172.67.201.240
    SvcParam: ech
        SvcParamKey: ech (5)
        SvcParamValue length: 71
        ECHConfigList length: 69
        ECHConfig: id=131 cloudflare-ech.com
    SvcParam: ipv6hint=2606:4700:3030::6815:3cdc,2606:4700:3030::ac43:c9f0
        SvcParamKey: ipv6hint (6)
        SvcParamValue length: 32
        IP: 2606:4700:3030::6815:3cdc
        IP: 2606:4700:3030::ac43:c9f0
```

А вот наш TLS handshake с балансером, за которым находится [prometheus.io](https://prometheus.io/):
```
TLSv1 Record Layer: Handshake Protocol: Client Hello
    Content Type: Handshake (22)
    Version: TLS 1.0 (0x0301)
    Length: 512
    Handshake Protocol: Client Hello
        Handshake Type: Client Hello (1)
        Length: 508
        Version: TLS 1.2 (0x0303)
        ...
        Extension: server_name (len=23) name=cloudflare-ech.com
            Type: server_name (0)
            Length: 23
            Server Name Indication extension
                Server Name list length: 21
                Server Name Type: host_name (0)
                Server Name length: 18
                Server Name: cloudflare-ech.com
        ...
```

Ого, всё ещё интереснее: несмотря на то, что мы идём на [https://prometheus.io/](https://prometheus.io/), самому серверу мы в SNI указываем какой-то cloudflare-ech.com, который как раз нам выдал `HTTPS` DNS-запрос.

Идём гуглить – и узнаём, что [HTTPS](https://blog.cloudflare.com/speeding-up-https-and-http-3-negotiation-with-dns/) DNS-запись призвана:
1. Уменьшить latency загрузки сайта при первичном заходе, т. к. сразу предоставляет `A`, `AAAA`-записи, а также список поддерживаемых протоколов (HTTP/2, HTTP/3).
2. Увеличивает безопасность соединения (при первичном открытии сайта, когда в браузере ещё нет закешированного [HSTS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)).
3. ECH stands for Encrypted Client Hello.

Когда-то давно я читал про [ESNI (Encrypted Server Name Indication)](https://www.cloudflare.com/learning/ssl/what-is-encrypted-sni/), который призван решить проблему plain text'овости домена в TLS. И даже ни раз приходилось слышать, что Роскомнадзор блокирует ESNI как раз по той причине, то он не позволяет фильтровать HTTPS-трафик. Но вот [ECH (Encrypted Client Hello)](https://blog.cloudflare.com/encrypted-client-hello/), как-то проходил мимо меня, как что-то ещё из неопределённого будущего.

Почитав про [ECH](https://blog.cloudflare.com/encrypted-client-hello/), мы узнаём, что схема работает следующим образом:
1. Браузер запрашивает [HTTPS](https://blog.cloudflare.com/speeding-up-https-and-http-3-negotiation-with-dns/) DNS-запись.
2. Данная запись помимо IP-адресов возвращает публичный домен-заглушку cloudflare-ech.com + открытый ключ.
3. При установке соединения с сайтом браузер в [SNI](https://www.cloudflare.com/learning/ssl/what-is-sni/) (в открытой части Client Hello) передаёт ни о чём не говорящий cloudflare-ech.com, а в закрытой – уже в зашифрованном виде, настоящий домен.
4. РКН видит это, расстраивается, что ему не видно конечный домен – и [блокирует соединение](https://habr.com/ru/news/856342/).

Ну а, собственно, проблемы начались из-за того, что Cloudflare, за которым находится просто бесчисленное множество сайтов, недавно [начал массово включать ECH](https://github.com/net4people/bbs/issues/393) по умолчанию для всех сайтов.


## Как обойти

Если хочется сохранить доступ к сайтам, находящимся за Cloudflare, ~~не привлекая внимания санитаров~~ не заворачивая куда-либо абсолютно весь трафик до него, то первая мысль, которая приходит в голову – это пойти и отключить ECH в настройках браузера. Но браузеров много, устройств много, а помимо браузеров есть множество других программ, которые рано или поздно тоже начнут поддерживать ECH.

Если же хочется какого-то более универсального решения, то достаточно вспомнить о том, с чего всё началось, а именно – с HTTPS-записи. Поэтому решение, которое выбрал я (по крайней мере пока) – это просто взять и заблокировать `HTTPS` DNS-запись на своём роутере. В моём случае это решается добавлением `filter-rr=HTTPS` в конфиг [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html).

Имея такую конфигурацию, наш DNS-сервер, находящийся на роутере, не будет отвечать на запросы `HTTPS`-записей, и браузер будет получать только обычные `A` и `AAAA`-записи, что вынудит его работать по старинке без всяких ECH. РКН будет видеть, что вы действительно заходите на незаблокированный ресурс и не будет резать соединение.