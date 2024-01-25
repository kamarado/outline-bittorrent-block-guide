# Outline c блокировкой торрентов (и не только)


## Зачем?

Некоторые облачные платформы отправляют страйки и уведомления об абьюзе из-за торрентов. Вероятно, из-за жалоб правообладателей. То есть с самим протоколом все в порядке. Проблемы с контентом. Тем не менее, если не удается договориться со своими пользователями об организационном запрете закачек (что должно быть сделано в первую очередь), можно попытаться заблокировать их техническими средствами.


## Что будем делать?

Будем ставить сервер Outline и настроим его так, чтобы блокировать все, что не разрешено явно.
Блокировать будем порты при помощи `ufw`. В минимальном варианте оставим открытыми только
ssh, DNS, http, https и порты самого Outline. причем последние укажем при установке сервера.

По шагам:
 1. Раздобудем linux-сервер (протестировано на Ubuntu 20.04)
 2. Установим сервер `Outline`
    * Скачаем `Outline Manager` по ссылке https://getoutline.org/get-started/#step-1 и установим его
    * После запуска `Outline Manager`
        * Нажмем *Добавить сервер*
        * Выберем один из стандартных облачных сервисов или самостоятельно настроенную платформу (плитка "Настройте Outline где угодно")
        * Нажмем *Настроить*
        * Скопируем в буфер обмена команду из пункта *1* инструкции
        
        ```sh
        sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
        ```
        
        * Эту команду нужно модифицировать для установки Outline с заранее заданными портами. Для этого ее нужно вставить в любой редактор или непосредственно в терминал и дописать параметры. Должно получиться нечто такое:

        ```sh
        sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh) --api-port 41814 --keys-port 1814"
        ```

        * Выполняем и ждем завершения. При отсутствии ошибок Outline выдаст инструкции для дальнейших шагов. Например:

        ```sh
        To manage your Outline server, please copy the following line (including curly brackets) into Step 2 of the Outline Manager interface:
        
        {"apiUrl":"https://1.2.3.4:41814/abcdefg","certSha256":"0123456789ABCDEF"}
        
        If you have connection problems, it may be that your router or cloud provider blocks inbound connections, even though your machine seems to allow them.
        
        Make sure to open the following ports on your firewall, router or cloud provider:
        - Management port 41814, for TCP
        - Access key port 1814, for TCP and UDP
        ```
        * Скопируем параметры доступа `{"apiUrl":"", "certSha256":""}` в буфер обмена и вставим в форму *2* `Outline Manager`
        * Нажмем *Готово*
3. Настроим блокировку портов. Идея состоит в том, чтобы блокировать все, кроме необходимого минимума
    * Подключаемся к терминалу сервера и проверяем состояние файрвола. В идеале нужно получить нечто подобное примеру ниже, в котором разрешен доступ к управлению сервером по ssh (порт 22), разрешение имен DNS (53), http (80), https (443), выбранные при установке порты Outline: клиентский (1418) и управления сервером (41814). Подключения к остальным портам заблокированы. **Порядок правил имеет значение: применяется первое подходящее правило**, поэтому сначала мы разрешаем нужные порты, а затем ограничиваем все остальные. Если порты отличаются, вносим правки по ситуации
    
        ```sh
        user@host:~$ ufw status verbose

        Status: active

        To                         Action      From
        --                         ------      ----
        Status: active
        Logging: on (high)
        Default: deny (incoming), allow (outgoing), deny (routed)
        New profiles: skip

        To                         Action      From
        --                         ------      ----
        22                         ALLOW IN    Anywhere
        80/tcp                     ALLOW IN    Anywhere
        443/tcp                    ALLOW IN    Anywhere
        1814                       ALLOW IN    Anywhere
        41814                      ALLOW IN    Anywhere
        1:65535/tcp                REJECT IN   Anywhere
        1:65535/udp                REJECT IN   Anywhere
        80/tcp (v6)                ALLOW IN    Anywhere (v6)
        443/tcp (v6)               ALLOW IN    Anywhere (v6)
        1:65535/tcp (v6)           REJECT IN   Anywhere (v6)
        1:65535/udp (v6)           REJECT IN   Anywhere (v6)

        53                         ALLOW OUT   Anywhere
        80/tcp                     ALLOW OUT   Anywhere
        443/tcp                    ALLOW OUT   Anywhere
        1814                       ALLOW OUT   Anywhere
        1:65535/tcp                REJECT OUT  Anywhere
        1:65535/udp                REJECT OUT  Anywhere
        53 (v6)                    ALLOW OUT   Anywhere (v6)
        80/tcp (v6)                ALLOW OUT   Anywhere (v6)
        443/tcp (v6)               ALLOW OUT   Anywhere (v6)
        1:65535/tcp (v6)           REJECT OUT  Anywhere (v6)
        1:65535/udp (v6)           REJECT OUT  Anywhere (v6)
        ```
    
    * Проверяем, разрешено ли ufw управлять ipv6

        ```sh
        user@host:~$ cat /etc/default/ufw | grep IPV6
        IPV6=yes
        ```
        
    * Если в ответ получили `IPV6=no`, исправляем

        ```sh
        cat /etc/default/ufw | sudo sed 's/^IPV6=.*$/IPV6=yes/' > /etc/default/ufw
        ```

    * Если файрвол выключен или правила не соответствуют приведенным выше, выполняем команды
        
        ```sh
        sudo ufw limmit 22; \
        sudo ufw allow out 53; \
        sudo ufw allow 80/tcp; \
        sudo ufw allow out 80/tcp; \
        sudo ufw allow 443/tcp; \
        sudo ufw allow out 443/tcp; \
        sudo ufw allow 1814; \
        sudo ufw allow out 1814; \
        sudo ufw allow 41814; \
        sudo ufw reject 1:65535/tcp; \
        sudo ufw reject 1:65535/udp; \
        sudo ufw reject out 1:65535/tcp; \
        sudo ufw reject out 1:65535/udp

        ```

    * Запускаем / перезапускаем файрвол с новыми правилами. При этом команда выдаст предупреждение про возможное нарушение связи с сервером. **Внимательно проверяем порт SSH (по умолчанию 22), чтобы не заблокировать сервер** и соглашаемся

        ```sh
        sudo ufw disable; sudo ufw enable
        ```

    * Проверяем результат

        ```sh
        ufw status verbose
        ```
        или
        ```sh
        ufw status numbered
        ```


## Дальнейшее использование сервера

Для добавления разрешенных портов (например, может понадобиться пересылать почту через порты 25, 465, 110, 995, 143, 993) добавляем правила для этих портов вставкой в начало списка правил. Во всяком случае, разрешенные порты должны идти до заблокированных

```sh
sudo ufw insert 1 allow номер_порта
sudo ufw insert 1 allow out номер_порта
```


## Ограничения

Некоторые пиры все еще могут использовать для раздачи оставленные открытыми порты.


## Что еще можно сделать

### Установить лимит трафика для ключей

В `Outline Manager` выбираем сервер, вкладку *Настройки* -> *Лимит данных* -> *Включен* -> Задать *Лимит данных на ключ*. Если необходимо, можно настроить подобные лимиты для каждого ключа в отдельности: в списке ключей нажать на три точки.

### Часть bittorrent-трафика можно пытаться блокировать через iptables

Способ хотя и является самым правильным, но нетривиальный, до конца не отработан и в текущем виде помогает только отчасти.  

Нужно сделать следующее:
 * Установить дополнительные модули файрвола (иначе не будет работать ipp2p)

    ```sh
    sudo apt install xtables-addons-common
    ```

 * Дописать следующее в конце файла `/etc/ufw/before.rules`

    ```ini
    ## BitTorrent blocking Rules

    *mangle
    # Mark BitTorrent traffic while passing already marked packets through
    -A PREROUTING -j CONNMARK --restore-mark
    -A PREROUTING -m mark ! --mark 0 -j ACCEPT
    -A PREROUTING -m ipp2p --bit -j MARK --set-mark 6881
    -A PREROUTING -m mark --mark 6881 -j CONNMARK --save-mark

    -A OUTPUT -j CONNMARK --restore-mark
    -A OUTPUT -m mark ! --mark 0 -j ACCEPT
    -A OUTPUT -m ipp2p --bit -j MARK --set-mark 6882
    -A OUTPUT -m mark --mark 6882 -j CONNMARK --save-mark

    COMMIT

    *filter
    :LOGREJECT-INPUT - [0:0]
    :LOGREJECT-OUTPUT - [0:0]
    :LOGREJECT-MARKED - [0:0]

    -A LOGREJECT-INPUT -j LOG --log-prefix "LOGREJECT-INPUT "
    -A LOGREJECT-INPUT -j REJECT --reject-with icmp-proto-unreachable

    -A LOGREJECT-OUTPUT -j LOG --log-prefix "LOGREJECT-OUTPUT "
    -A LOGREJECT-OUTPUT -j REJECT --reject-with icmp-proto-unreachable

    -A LOGREJECT-MARKED -j LOG --log-prefix "LOGREJECT-MARKED "
    -A LOGREJECT-MARKED -j REJECT --reject-with icmp-proto-unreachable

    # Log+reject input packets with BitTorrent-like dataload
    -A ufw-before-input -m string --algo bm --string "BitTorrent" -j LOGREJECT-INPUT
    -A ufw-before-input -m string --algo bm --string "BitTorrent protocol" -j LOGREJECT-INPUT
    -A ufw-before-input -m string --algo bm --string "peer_id=" -j LOGREJECT-INPUT
    -A ufw-before-input -m string --algo bm --string ".torrent" -j LOGREJECT-INPUT
    -A ufw-before-input -m string --algo bm --string "announce.php?passkey=" -j LOGREJECT-INPUT
    -A ufw-before-input -m string --algo bm --string "torrent" -j LOGREJECT-INPUT
    -A ufw-before-input -m string --algo bm --string "announce" -j LOGREJECT-INPUT
    -A ufw-before-input -m string --algo bm --string "info_hash" -j LOGREJECT-INPUT

    # Log+reject input packets by DHT keywords
    -A ufw-before-input -m string --string "get_peers" --algo bm -j LOGREJECT-INPUT
    -A ufw-before-input -m string --string "announce_peer" --algo bm -j LOGREJECT-INPUT
    -A ufw-before-input -m string --string "find_node" --algo bm -j LOGREJECT-INPUT

    # Log+reject output packets with BitTorrent-like dataload
    -A ufw-before-output -m string --algo bm --string "BitTorrent" -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --algo bm --string "BitTorrent protocol" -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --algo bm --string "peer_id=" -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --algo bm --string ".torrent" -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --algo bm --string "announce.php?passkey=" -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --algo bm --string "torrent" -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --algo bm --string "announce" -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --algo bm --string "info_hash" -j LOGREJECT-OUTPUT

    # Log+reject output packets by DHT keywords
    -A ufw-before-output -m string --string "get_peers" --algo bm -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --string "announce_peer" --algo bm -j LOGREJECT-OUTPUT
    -A ufw-before-output -m string --string "find_node" --algo bm -j LOGREJECT-OUTPUT

    # Log+reject input packets marked as part of a BitTorrent connection
    -A ufw-before-input -m mark --mark 6881 -j LOGREJECT-MARKED

    # Log+reject output packets marked as part of a BitTorrent connection
    -A ufw-before-output -m mark --mark 6882 -j LOGREJECT-MARKED

    COMMIT

    ```

 * Если используется IPv6, дописать в конец файла `/etc/ufw/before6.rules`

    ```sh
    ## BitTorrent blocking Rules

    *filter
    :LOGREJECT6-INPUT - [0:0]
    :LOGREJECT6-OUTPUT - [0:0]

    -A LOGREJECT6-INPUT -j LOG --log-prefix "LOGREJECT6-INPUT "
    -A LOGREJECT6-INPUT -j REJECT

    -A LOGREJECT6-OUTPUT -j LOG --log-prefix "LOGREJECT6-OUTPUT "
    -A LOGREJECT6-OUTPUT -j REJECT

    # Log+reject input packets with BitTorrent-like dataload
    -A ufw6-before-input -m string --algo bm --string "BitTorrent" -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --algo bm --string "BitTorrent protocol" -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --algo bm --string "peer_id=" -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --algo bm --string ".torrent" -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --algo bm --string "announce.php?passkey=" -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --algo bm --string "torrent" -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --algo bm --string "announce" -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --algo bm --string "info_hash" -j LOGREJECT6-INPUT

    # Log+reject input packets by DHT keywords
    -A ufw6-before-input -m string --string "get_peers" --algo bm -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --string "announce_peer" --algo bm -j LOGREJECT6-INPUT
    -A ufw6-before-input -m string --string "find_node" --algo bm -j LOGREJECT6-INPUT

    # Log+reject output packets with BitTorrent-like dataload
    -A ufw6-before-output -m string --algo bm --string "BitTorrent" -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --algo bm --string "BitTorrent protocol" -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --algo bm --string "peer_id=" -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --algo bm --string ".torrent" -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --algo bm --string "announce.php?passkey=" -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --algo bm --string "torrent" -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --algo bm --string "announce" -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --algo bm --string "info_hash" -j LOGREJECT6-OUTPUT

    # Log+reject output packets by DHT keywords
    -A ufw6-before-output -m string --string "get_peers" --algo bm -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --string "announce_peer" --algo bm -j LOGREJECT6-OUTPUT
    -A ufw6-before-output -m string --string "find_node" --algo bm -j LOGREJECT6-OUTPUT

    COMMIT
    
    ```
    
    К можалению, модуль `ipp2p` реализован только для IPv4, поэтому мы не можем включить его в правила для IPv6


 * Перезапустить файрвол

    ```sh
    sudo ufw disable; sudo ufw enable
    ```

    или перезагрузить сервер

    ```sh
    sudo reboot
    ```

 * Для контроля перехвата трафика читать системный лог

    ```sh
    dmesg | grep LOGREJECT
    ```

    и смотреть первые две колонки статистики `iptables` командами

    ```sh
    user@host:~$ iptables -vL LOGREJECT-INPUT

    Chain LOGREJECT-INPUT (11 references)
    pkts bytes target     prot opt in     out     source               destination
    0     0 LOG        all  --  any    any     anywhere             anywhere             LOG level warning prefix "LOGREJECT-INPUT "
    0     0 REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-port-unreachable
    ```


## Как узнать, что все получилось

### Понаблюдать в `Outline Manager`
 * Создать ключ `Outline` и добавить его в клиент
 * Подключиться к серверу
 * Запустить закачку небольшого файла через торренты
 * В `Outline Manager` наблюдать за потреблением трафика у созданного ключа. Если закачка не идет или идет успешно, а потребление не растет, значит все настроено хорошо

### Перехватывать и анализировать пакеты при помощи связки rpcapd + Wireshark
 * Установить на сервере демон удаленного перехвата пакетов `rpcapd` (на примере Ubuntu Server 20.04)
    + На некоторых облачных платформах (например, у Hetzner) в репозитории может не быть исходников, поэтому добавляем альтернативное зеркало командой

    ```sh
    echo "deb-src https://mirror.yandex.ru/ubuntu focal main restricted" >> /etc/apt/sources.list
    ```

    +  Стоит иметь в виду, что эти изменения могут быть потеряны при перезагрузке, если облачная платформа управляет источниками. Не удивляйтесь! Если нужен постоянный доступ к исходникам, редактируйте `/etc/cloud/templates/sources.list.*.tmpl`

    + Выполняем `sudo apt update`
    + Устанавливаем инструменты `sudo apt install git checkinstall`
    + А также зависимости `sudo apt build-dep libpcap`
    + Скачиваем исходники демона `cd ~; git clone https://github.com/the-tcpdump-group/libpcap.git`
    + Переходим к исходникам `cd libpcap`
    + Выполняем `./autogen.sh`
    + Конфигурируем и собираем `./configure --enable-remote && make`
    + Выполняем `checkinstall` для создания и установки пакета в систему. Читаем инструкции и меняем рекомендуемые параметры. Можно, хотя и необязательно, переименовать пакет из `libpcap` в `libpcap-rpcapd`, чтобы отличать его от стандартного.
    + Итак, демон установлен. Проверить это можно командой `rpcapd -h`
* Скачать, запустить установщик `Wireshark` https://www.wireshark.org/download.html и на этапе выбора компонентов добавить пункт `Sshdump...`. Остальные компоненты по желанию. Затем продолжить установку
* Запустить `Wireshark`
* Нажать на шестеренку слева от пункта `SSH remote capture`
    + Во вкладке `Server` указываем адрес и SSH-порт (22) сервера, с которого будем снимать пакеты
    + Во вкладке `Auth...` указываем пользователя и пароль или путь к приватному SSH-ключу для доступа к серверу
    + Сохраняем настройки
* Начать захват пакетов двойным щелчком по пункту `SSH remote capture`. **Активный захват довольно быстро съедает ВСЁ доступное место на диске. За полчаса работы может легко уйти 10-20 Гб!**. Рекомендуется не оставлять его включенным надолго. Впрочем, переживать не стоит, потому что при перезапуске захвата и при выходе из `Wireshark` все занятое освобождается
* В строке фильтра (вверху) задаем фильтр `bittorrent || bt-dht || bt-tracker || bt-utp` и применяем его (Enter или стрелка справа от строки фильтра)
* Смотрим и пытаемся что-то понять
