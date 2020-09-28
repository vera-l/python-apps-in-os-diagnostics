# Диагностика запущенных python-приложений

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

## Доп. литература
https://github.com/vera-l/python-resources#os



