# Диагностика запущенных в ОС (mac, linux) python-приложений

<a name="index"></a>
* [Информация о процессах](#processes)
([pstree](#pstree), [pgrep](#pgrep), [pidof](#pidof), [pidstat](#pidstat), [ps](#ps), [top](#top), [atop](#atop), [htop](#htop))
* [Системные и библиотечные вызовы](#syscalls)
([strace](#strace), [ltrace](#ltrace), [dtruss](#dtruss))
* [Профилирование](#profiling)
([perf](#perf), [py-spy](#py-spy))
* [Сбор метрик из приложения для отображения на графиках](#metrics)
* [Доп. литература](#resources)

<a name="processes"></a>
## Информация о процессах [^](#index "к оглавлению")

Узнаем о состоянии процесса и всей системы при помощи нескольких утилит. Для проведения диагностики нам понадобится узнать pid процесса. Также, в системе могут присутствовать и другие приложения, которые могут оказывать влияние на наше. Для каждой программы подробная справка с дополнительными опциями доступна по `man <name>`

<a name="pstree"></a>
### pstree [^](#index "к оглавлению")
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

<a name="pgrep"></a>
### pgrep [^](#index "к оглавлению")
Поиск процесса по имени и некоторым атрибутам. Аналогичная `pkill` - посылает найденным процессам сигнал.
```console
vera@vera$ pgrep -a nginx
669640 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
669641 nginx: worker process 
```

<a name="pidof"></a>
### pidof [^](#index "к оглавлению")
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

<a name="pidstat"></a>
### pidstat [^](#index "к оглавлению")
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

<a name="ps"></a>
### ps [^](#index "к оглавлению")
Отображает снимок процессов **на данный момент** с подробной информацией. Имеет много опций и возможностей (подробнее `man ps`).
`ps aux` - Отображает снимок всех запущенных в системе процессов на данный момент, с подробной информацией. 
Добавим `--sort=-%cpu` или `--sort=-%mem` - сортировка по использованию cpu или памяти. Например,
```console
vera@vera:$ ps aux --sort=-%mem | head
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         218  0.0  7.4 137748 75164 ?        S<s  Aug26  28:21 /lib/systemd/systemd-journald
dd-agent  668713  0.6  5.8 950228 58368 ?        Ssl  Aug29 282:44 /opt/datadog-agent/bin/agent/agent 
vera     1062819  0.0  5.1 591588 52168 ?        Ssl  13:22   0:09 python3 app.py
mongodb  2282713  0.3  4.0 984948 40588 ?        Ssl  Sep18  51:34 /usr/bin/mongod 
root     1271512  0.0  1.5 646168 15804 ?        Ssl  Sep15   1:57 /usr/lib/snapd/snapd
root           1  0.0  0.9 168400  9732 ?        Ss   Aug26   4:22 /sbin/init
root      427317  0.0  0.9 435428  9648 ?        Ss   Sep12   0:01 /usr/bin/python3 /usr/bin/glances
```
`ps axjf` - отображение процессов в виде дерева
```console
vera@vera$ ps axjf
PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
      1  669640  669640  664640 ?             -1 Ss       0   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
 669640  669641  669640  664640 ?             -1 S       33   4:04  \_ nginx: worker process   
...
```

<a name="top"></a>
### top [^](#index "к оглавлению")

<a name="atop"></a>
### atop [^](#index "к оглавлению")

<a name="htop"></a>
### htop [^](#index "к оглавлению")



<a name="syscalls"></a>
## Системные и библиотечные вызовы [^](#index "к оглавлению")

<a name="strace"></a>
### strace (🐧 only) [^](#index "к оглавлению")
Выводит системные вызовы и сиглалы конкретного процесса
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
Посмотреть, что означает тот или иной системный вызов:
```console
vera@vera$ man 2 epoll_wait

EPOLL_WAIT(2)                         Linux Programmer's Manual                         EPOLL_WAIT(2)

NAME
       epoll_wait, epoll_pwait - wait for an I/O event on an epoll file descriptor
...
```
Кейс: в логах была нечитаемая информация. Смотрим вывод `strace` - там много вызовов `write('a')` с одним символом. Значит, в лог пишут несколько процессов по одному символу. Уже стало понятнее, куда дальше копать.

<a name="ltrace"></a>
### ltrace (🐧 only) [^](#index "к оглавлению")
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

<a name="dtruss"></a>
### dtruss ( only) [^](#index "к оглавлению")
Отображает системные вызовы. Использует Dtrace
```console
vera@~/dev/dev/dev$ sudo dtruss -p 9616
dtrace: system integrity protection is on, some features will not be available

SYSCALL(args) 		 = return
gettimeofday(0x7000039E1740, 0x0, 0x0)		 = 0 0
psynch_cvsignal(0x1089C7620, 0x2C0000002D00, 0x2C00)		 = 257 0
psynch_cvwait(0x1089C7620, 0x2C0100002D00, 0x2C00)		 = 0 0
kqueue(0x0, 0x0, 0x0)		 = 17 0
kevent(0x11, 0x700003EE40D8, 0x1)		 = 0 0
socketpair(0x1, 0x1, 0x0)		 = 0 0
gettimeofday(0x7000039E1760, 0x0, 0x0)		 = 0 0
setsockopt(0x12, 0xFFFF, 0x1100)		 = 0 0
sendto_nocancel(0x10, 0x7FCC5BA27E80, 0x32)		 = 50 0
sendmsg_nocancel(0x10, 0x700003EE3BD0, 0x0)		 = 1 0
close_nocancel(0x13)		 = 0 0
select_nocancel(0x13, 0x700003EE3BD0, 0x0)		 = 1 0
recvfrom_nocancel(0x12, 0x700003EE3BA0, 0x4)		 = 4 0
close_nocancel(0x12)		 = 0 0
kevent(0x11, 0x700003EE40D8, 0x1)		 = 0 0
```

<a name="profiling"></a>
## Профилирование [^](#index "к оглавлению")

<a name="perf"></a>
### perf [^](#index "к оглавлению")

<a name="py-spy"></a>
### py-spy [^](#index "к оглавлению")

<a name="ebpf"></a>
## eBPF [^](#index "к оглавлению")


```
...
#ifndef LATENCY
  BPF_HASH(counts, struct method_t, u64);            // number of calls
  #ifdef SYSCALLS
    BPF_HASH(syscounts, u64, u64);                   // number of calls per IP
  #endif  // SYSCALLS
#else
  BPF_HASH(times, struct method_t, struct info_t);
  BPF_HASH(entry, struct entry_t, u64);              // timestamp at entry
  #ifdef SYSCALLS
    BPF_HASH(systimes, u64, struct info_t);          // latency per IP
    BPF_HASH(sysentry, u64, struct syscall_entry_t); // ts + IP at entry
  #endif  // SYSCALLS
#endif
...
```

## bpftrace [^](#index "к оглавлению")

```console
vera@vera$ sudo bpftrace -e 'usdt:/usr/bin/python3.8:function__entry {printf("%d: %s:%d %s() \n", pid, str(arg0), arg2, str(arg1))}' -p $(pidof python3)
Attaching 1 probe...
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/periodic_:115 __should_stop() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_cli:731 target() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_cli:1747 _process_periodic_tasks() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_cli:1723 _process_kill_cursors() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/topology.:433 update_pool() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py:1128 remove_stale_sockets() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py:384 max_idle_time_seconds() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py:377 min_pool_size() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/periodic_:115 __should_stop() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_cli:731 target() 
1062829: /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_cli:1747 _process_periodic_tasks() 
```

## Утилиты из пакета `bpfcc-tools` для питона [^](#index "к оглавлению")
* `pythoncalls-bpfcc` - Суммирует вызовы функций питон приложения (также есть подобные для некоторых других высокоуровневых языков) и системные вызовы. Записывает количество вызовов и время выполнения в сумме. 

```console
vera@vera$ sudo pythoncalls-bpfcc -L -S 1042829
Attached kernel tracepoints for syscall tracing.
Tracing calls in process 1062829 (language: python)... Ctrl-C to quit.
^C
METHOD                                              # CALLS TIME (us)

local/lib/python3.8/site-packages/httpx/models.py.aread       36 121236.02
/usr/lib/python3.8/traceback.py.extract_stack           165 122949.36
/home/vera/.local/lib/python3.8/site-packages/httpx/client.py.send_single_request       30 125382.13
/home/vera/.local/lib/python3.8/site-packages/httpx/client.py.send_handling_auth       30 125848.94
/home/vera/.local/lib/python3.8/site-packages/httpx/client.py.send_handling_redirects       30 126382.88
/home/vera/.local/lib/python3.8/site-packages/httpx/dispatch/http11.py._receive_event       85 134719.61
/usr/lib/python3.8/asyncio/tasks.py.wait_for            108 167032.10
/usr/lib/python3.8/traceback.py.extract                 568 219331.55
/home/vera/.local/lib/python3.8/site-packages/httpx/client.py.send       60 249747.39
/home/vera/.local/lib/python3.8/site-packages/httpx/client.py.request       60 275263.45
/home/vera/.local/lib/python3.8/site-packages/httpx/client.py.get       60 275729.60
epoll_wait                                              199 2291913.30
/usr/lib/python3.8/asyncio/streams.py.open_connection       14 2422572.04
/usr/lib/python3.8/asyncio/streams.py.open_connection        5 2461725.92
futex                                                    59 7306215.55
select                                                   22 11012235.26
Detaching kernel probes, please wait...
```

* `pythonflow-bpfcc` - записывает цепочку выполнения методов, есть возможность фильтрации по классу или методу

```console
vera@vera$ sudo pythonflow-bpfcc 1082829
Tracing method calls in python process 1062829... Ctrl-C to quit.
CPU PID    TID    TIME(us) METHOD
0   1062829 1062901 0.244    -> /home/vera/.local/lib/python3.8/site-packages/pymongo/periodic_executor.py.__should_stop
0   1062829 1062901 0.244    <- /home/vera/.local/lib/python3.8/site-packages/pymongo/periodic_executor.py.__should_stop
0   1062829 1062901 0.244    -> /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_client.py.target
0   1062829 1062901 0.245      -> /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_client.py._process_periodic_tasks
0   1062829 1062901 0.245        -> /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_client.py._process_kill_cursors
0   1062829 1062901 0.245        <- /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_client.py._process_kill_cursors
0   1062829 1062901 0.245        -> /home/vera/.local/lib/python3.8/site-packages/pymongo/topology.py.update_pool
0   1062829 1062901 0.245          -> /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py.remove_stale_sockets
0   1062829 1062901 0.245            -> /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py.max_idle_time_seconds
0   1062829 1062901 0.245            <- /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py.max_idle_time_seconds
0   1062829 1062901 0.245            -> /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py.min_pool_size
0   1062829 1062901 0.245            <- /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py.min_pool_size
0   1062829 1062901 0.246          <- /home/vera/.local/lib/python3.8/site-packages/pymongo/pool.py.remove_stale_sockets
0   1062829 1062901 0.246        <- /home/vera/.local/lib/python3.8/site-packages/pymongo/topology.py.update_pool
0   1062829 1062901 0.246      <- /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_client.py._process_periodic_tasks
0   1062829 1062901 0.246    <- /home/vera/.local/lib/python3.8/site-packages/pymongo/mongo_client.py.target

```
       
* `pythongc-bpfcc` - Записывает события сборки мусора. Можно задать порог длительности сборки и фильтр по коллекции.

```console
vera@vera:/var/www/sanc$ sudo pythongc-bpfcc 1142829
        
Tracing garbage collections in python process 1142829... Ctrl-C to quit.
START    TIME (us) DESCRIPTION                             
10.804   835.96   None
22.757   999.44   None
27.725   826.92   None
```

* `pythonstat-bpfcc` - Собирает статистику по приложению (события сборки мусора, исключения, создание потоков, выделение памяти, вызовы методов и т.д.). Отображаются в виде отсортированной таблицы.

```console
vera@vera$ sudo pythonstat-bpfcc 1032829
17:48:42 loadavg: 0.00 0.00 0.00 1/158 1122088

PID    CMDLINE              METHOD/s   GC/s   OBJNEW/s   CLOAD/s  EXC/s  THR/s 
Detaching...
```

Другие утилиты из пакета https://packages.ubuntu.com/ru/bionic/all/bpfcc-tools/filelist. Дока будет доступна `man ...` после установки пакета для всех утилит. 


<a name="metrics"></a>
## Сбор метрик из приложения для отображения на графиках [^](#index "к оглавлению")

### Время выполнения нескольких типовых операций [^](#index "к оглавлению")

Типовая операция - это что-то постоянное, например конвертация одного какого-то юнита или обработка данных одного объема.
На графиках это время выполения будет увеличиваться в часы нагрузки и уменьшаться ночью.
Если выходит какой-то плохой релиз, влияющий на производительность - на этом графике будет заметен скачок, и наоборот.
Например, при переходе с питона 3.5 на 3.7 на графиках было заметно уменьшение времени выполнения этих типовых операций на ~20%. 

```python
...
start = time.time()
typical_operation()
duration_ms = (time.time() - self.start) * 1000

my_metrics.send(f'Typical operation has taken {duration_ms} ms')
```

<a name="resources"></a>
## Доп. литература [^](#index "к оглавлению")
https://github.com/vera-l/python-resources#os
