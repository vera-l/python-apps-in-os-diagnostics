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

## Доп. литература
https://github.com/vera-l/python-resources#os



