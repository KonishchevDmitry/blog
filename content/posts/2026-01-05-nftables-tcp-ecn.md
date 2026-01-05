---
title: nftables и TCP Explicit Congestion Notification – или как роботы Яндекса внезапно потеряли доступ к моему блогу
slug: nftables-tcp-ecn
date: 2026-01-05T20:56:25+03:00
---

Пару недель назад мне пришло письмо от [Яндекс Вебмастера](https://webmaster.yandex.ru/) с уведомлением о том, что мой блог стал недоступен. Каких-либо подробностей ни в письме, ни в UI в таких случаях, к сожалению, не предоставляется, а инструменты анализа robots.txt и sitemap.xml и вовсе вводили в заблуждение словами "Server returns HTTP code 502 (expected code 200)" – хотя я явно не видел никаких обращений роботов в логах web-сервера. При этом, судя по тем же логам, пользователи как приходили раньше, так и продолжали приходить + [Google Search Console](https://search.google.com/search-console) тоже не видел никаких проблем.

Поддержка Вебмастера ответила, что запросы роботов блокируются с моей стороны и посоветовала обратиться к хостинг-провайдеру.

## Поиск причины

Первая мысль – "неужели где-то перемудрил с [fail2ban](https://github.com/fail2ban/fail2ban)'ом?", но его отключение ни к чему не привело. К счастью, поддержка подсказала номер [AS](https://en.wikipedia.org/wiki/Autonomous_system_(Internet)), из которой могут исходить запросы роботов Яндекса – `AS13238`, что сильно облегчило поиск причины на своей стороне.

Получил список сетей Яндекса с помощью `bgpq3 -4 -F '%n/%l, ' AS13238` и добавил следующее правило в nftables поближе к самому началу обработки всех новых соединений:
```
ip saddr {5.45.192.0/18, 5.45.202.0/24, 5.45.205.0/24, 5.45.215.0/24, 5.255.192.0/18, 5.255.197.0/24, 5.255.255.0/24, 37.9.64.0/18, 37.9.64.0/24, 37.9.87.0/24, 37.9.112.0/24, 37.140.128.0/18, 77.88.0.0/18, 77.88.8.0/24, 77.88.44.0/24, 77.88.55.0/24, 84.252.160.0/19, 87.250.224.0/19, 87.250.247.0/24, 90.156.179.0/24, 90.156.180.0/24, 90.156.181.0/24, 90.156.184.0/24, 90.156.185.0/24, 92.255.112.0/20, 93.158.128.0/18, 95.108.128.0/17, 141.8.128.0/18, 178.154.128.0/19, 178.154.131.0/24, 178.154.160.0/19, 185.32.187.0/24, 213.180.192.0/19, 213.180.199.0/24} meta nftrace set 1
```

Запустил `nft monitor trace` и, воспользовавшись тулзой по анализу robots.txt, тут же получил следующее:
```
trace id f15d4066 inet mangle PREROUTING packet: iif "isp" ether saddr 3c:c7:86:12:89:7a ether daddr 08:f1:db:e6:ac:3b ip saddr 5.255.253.45 ip daddr 10.217.4.5 ip dscp cs0 ip ecn ect0 ip ttl 50 ip id 3342 ip protocol tcp ip length 60 tcp sport 61996 tcp dport 443 tcp flags == 0xc2 tcp window 42300
...
trace id f15d4066 inet filter check_packets rule ct state new tcp flags != syn counter packets 0 bytes 0 goto bad_tcp_new_packet (verdict goto bad_tcp_new_packet)
trace id f15d4066 inet filter bad_tcp_new_packet rule meta l4proto tcp counter packets 0 bytes 0 reject with tcp reset comment "Invalid TCP packets" (verdict drop)
```

Ого, это интересно. Мы что-то дропнули вот тут:
```
# New TCP connections must be started with SYN packets.
#
# NEW but not SYN is the only invalid TCP flag not covered by the
# INVALID state. The reason is because they are rarely malicious packets,
# and they should not just be dropped, but might be an error or attack.
#
# For example, this may be just connections forgotten by conntrack module.
ct state new tcp flags != syn counter goto bad_tcp_new_packet
```

Пробую убрать это правило – и теперь действительно всё работает. Ещё интереснее. :)

## Распространённая ошибка

Итак – проблема действительно на моей стороне. И это замечательно – меньше всего хотелось бы дебажить подобные вещи через поддержку своего провайдера.

Но вот только пока не очень понятно, в чём именно проблема: это ведь довольно стандартная рекомендация – reject'ить все новые TCP-соединения, у которых первый пакет не является SYN-пакетом. Да и что же тогда нам Яндексовый робот такое прислал, если это не SYN-пакет?..

Смотрим более внимательно в нашу трассировку – и видим там следующие TCP-флаги: `tcp flags == 0xc2`. `0xc2` – это `11000010`, что, исходя из [структуры TCP-пакета](https://en.wikipedia.org/wiki/Transmission_Control_Protocol), является `SYN + ECE + CWR`. Отлично, теперь всё понятно – это вполне себе SYN-пакет, но только какой-то необычный, а правило у нас написано довольно тупо (`tcp flags != syn`) и совершенно на такой случай не рассчитано. При этом, если для iptables обычно даётся рекомендация вида `iptables -t mangle -A PREROUTING -p tcp ! --syn -m conntrack --ctstate NEW -j DROP`, где `--syn` на самом деле является шорткатом для `--tcp-flags SYN,RST,ACK,FIN SYN`, то для nftables (помимо в целом в разы более куцей информации по его конфигурации) даже [официальная документация](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_headers) предлагает `nft add rule filter input tcp flags != syn counter` в качестве примера правила "to count packets that are not SYN ones" (как и во многих статьях в блогах, в которых правило представлено ровно в том виде, в котором оно используется у меня) – в результате чего возникает ощущение, что я могу быть далеко не единственным, кто столкнётся с этой проблемой (и именно это побудило меня написать данную статью).

Поэтому `tcp flags != syn` правильнее заменить на `tcp flags & (syn|ack|rst|fin) != syn`.

## Explicit Congestion Notification

А что же всё-таки за такой необычный пакет к нам пришёл? `ECE` и `CWR`-флаги ведут нас к [Explicit Congestion Notification](https://en.wikipedia.org/wiki/Explicit_Congestion_Notification) – хм, интересно: про [TCP congestion control](https://en.wikipedia.org/wiki/TCP_congestion_control) я в курсе, а вот Explicit Congestion Notification – это для меня что-то новое. Отлично – значит день уже прожит не зря. :)

Вкратце, суть его заключается в следующем: если традиционно в TCP текущая загруженность канала определялась неявно через отслеживание количества пакетов, которые потерялись по дороге к адресату (были дропнуты роутером где-то в середине пути), то ECN добавляет возможность роутеру специальным образом маркировать пакеты, тем самым заранее уведомляя участников TCP-соединения о том, что канал перегружен, и он скоро начнёт дропать пакеты. Но, само собой, рекомендую почитать [более полное описание](https://en.wikipedia.org/wiki/Explicit_Congestion_Notification).

Поэтому, видимо, дело в том, что недавно в Яндексе поменялась конфигурация серверов, на которых располагаются поисковые боты, и они стали устанавливать соединения с включенным ECN.

## В итоге

Если вы, как и во многих руководствах по nftables, дропаете/reject'ите пакеты по правилу `ct state new tcp flags != syn`, то его необходимо заменить на `ct state new tcp flags & (syn|ack|rst|fin) != syn` – иначе когда-нибудь эта довольно подлая бомба замедленного действия сработает и у вас.