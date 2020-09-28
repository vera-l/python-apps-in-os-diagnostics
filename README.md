# Диагностика запущенных python-приложений

Узнаем о состоянии процесса и всей системы при помощи нескольких утилит. 

## Список процессов

### pstree
Отображает дерево процессов. Доп опции `pt` включат отображение pid и потоков.

```console
vera@vera$ pstree -pt
systemd(1)─┬─accounts-daemon(323)─┬─{gdbus}(332)
           │                      └─{gmain}(338)
           ├─cron(325)
           ├─nginx(669630)───nginx(669631)
           ├─python3(1062829)─┬─{python3}(1062341)
           │                  ├─{python3}(1062342)
           │                  ├─{python3}(1062343)
           │                  ├─{python3}(1062344)
           │                  ├─{python3}(1062397)
           │                  ├─{python3}(1062398)
           │                  └─{python3}(1062301)
           ├─rsyslogd(339)─┬─{in:imklog}(337)
           │               ├─{in:imuxsock}(336)
           │               └─{rs:main Q:Reg}(338)
           ├─systemd(1041938)───(sd-pam)(1041935)
           ├─systemd-journal(238)
           └─systemd-udevd(235)
```

### pgrep
Поиск процесса по имени и некоторым атрибутам. Аналогичная `pkill` - посылает найденным процессам сигнал.
```console
vera@vera$ pgrep -a nginx
669640 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
669641 nginx: worker process 
```

### pidof
Аналог `pgrep`. Выводит список pid процессов, имя которых совпадает с заданным
```console
vera@vera$ pidof python3
1062859 427357 357
```
и действительно
```console
vera@vera$ ps aux | grep 'python'
root         357  0.0  0.4  34900  4832 ?        Ss   Aug26   0:00 /usr/bin/python3 /usr/bin/networkd-dispatcher 
root      427357  0.0  0.9 434428  9648 ?        Ss   Sep12   0:01 /usr/bin/python3 /usr/bin/glances
vera     1062859  0.0  4.8 545896 48272 ?        Ssl  13:22   0:07 python3 app.py
```


### pidstat
Статистика по процессам. Добавим `-p <PID> 1` - по процессу с pid каждую секунду. На 2-7 секундах подавалась нагрузка.
```console
vera@vera$ pidstat -p 1062829 1
Linux 5.4.0-29-generic (vera) 	09/28/20 	_x86_64_	(1 CPU)

14:52:31      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
14:52:32     1000   1062829    0.00    0.00    0.00    0.00    0.00     0  python3
14:52:33     1000   1062829    2.00    1.00    0.00    0.00    3.00     0  python3
14:52:34     1000   1062829    3.00    0.00    0.00    0.00    3.00     0  python3
14:52:35     1000   1062829    3.00    1.00    0.00    1.00    4.00     0  python3
14:52:36     1000   1062829    3.00    0.00    0.00    0.00    3.00     0  python3
14:52:37     1000   1062829    4.00    0.00    0.00    0.00    4.00     0  python3
14:52:38     1000   1062829    0.00    0.00    0.00    0.00    0.00     0  python3
14:52:39     1000   1062829    0.00    1.00    0.00    0.00    1.00     0  python3
^C
Average:     1000   1062829    1.50    0.30    0.00    0.10    1.80     -  python3
```

## Системные и библиотечные вызовы

### strace
Выводит системные вызовы и сиглалы конкретно процесса
```console
vera@vera$ sudo strace -p 1032829 
strace: Process 1032829 attached
epoll_wait(3, [{EPOLLIN, {u32=11, u64=11}}], 1024, -1) = 1
accept4(11, NULL, NULL, SOCK_CLOEXEC|SOCK_NONBLOCK) = 13
getpid()                                = 1032829
stat("app.py", {st_mode=S_IFREG|0664, st_size=1679, ...}) = 0
setsockopt(13, SOL_TCP, TCP_NODELAY, [1], 4) = 0
stat("app.py", {st_mode=S_IFREG|0664, st_size=1679, ...}) = 0
accept4(11, NULL, NULL, SOCK_CLOEXEC|SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)
getsockname(13, {sa_family=AF_INET, sin_port=htons(1845), sin_addr=inet_addr("74.443.134.152")}, [128->16]) = 0
getpeername(13, {sa_family=AF_INET, sin_port=htons(13825), sin_addr=inet_addr("169.252.119.327")}, [128->16]) = 0
stat("/home/vera/.local/lib/python3.8/site-packages/sanic/server.py", {st_mode=S_IFREG|0664, st_size=36448, ...}) = 0
stat("app.py", {st_mode=S_IFREG|0664, st_size=1679, ...}) = 0
stat("/home/vera/.local/lib/python3.8/site-packages/sanic/server.py", {st_mode=S_IFREG|0664, st_size=36448, ...}) = 0
```
Посмотреть, что означает тот или иной состемный вызов:
```console
vera@vera$ man 2 epoll_wait

EPOLL_WAIT(2)                         Linux Programmer's Manual                         EPOLL_WAIT(2)

NAME
       epoll_wait, epoll_pwait - wait for an I/O event on an epoll file descriptor
...
```
Кейс: в логах была нечитаемая информация. Смотрим вывод `strace` - там много вызовов `write('a')` с одним символом. Значит, в лог пишут несколько процессов по одному символу. Уже стало понятнее, куда дальше копать.

### ltrace
Выводит список библиотечных вызовов для данного процесса. 
```console
vera@vera$ sudo ltrace -p 1032829 
pthread_mutex_lock(0x934b70, 0, 0, 0xc279f9b)                   = 0
pthread_mutex_lock(0x934bc8, 0, 0, 0)                           = 0
pthread_cond_signal(0x934b98, 0, 0, 0)                          = 0
pthread_mutex_unlock(0x934bc8, 0, 0, 0)                         = 0
pthread_mutex_unlock(0x934b70, 0, 0, 0)                         = 0
clock_gettime(1, 0x62d2650, 0xf8003950, 0)                      = 0
ceil(3, 0x98bc270, 3, 0x3b9aca00)                               = 0
clock_gettime(1, 0x62d2600, 0, 28)                              = 0
^C
```
Посмотреть, что означает тот или иной библиотечный вызов:
```console
vera@vera$ man 3 pthread_cond_signal

PTHREAD_COND_SIGNAL(3)   BSD Library Functions Manual   PTHREAD_COND_SIGNAL(3)

NAME
     pthread_cond_signal -- unblock a thread waiting for a condition variable
...
```

## Профилирование

### perf







## Доп. литература
https://github.com/vera-l/python-resources#os



