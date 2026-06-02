# Домашнее задание "Управление процессами"

## Задание

Реализация аналога ps ax

- Создайте скрипт, который получает информацию о процессах через файловую систему /proc.
- Реализуйте вывод не менее следующих полей: PID, PPID, состояние процесса, имя или команда запуска.
- Проверьте работу скрипта на запущенной системе.
- Зафиксируйте пример результата работы.
---

## Выполнение задания

создаем скрипт, делаем исполняемым, проверяем:
```bash
root@client:/home/user# touch ps_ax.sh
root@client:/home/user# chmod +x ps_ax.sh
```
<details> 
<summary>скрипт ps_ax.sh</summary>

```bash
#!/bin/bash

print_header() {                                                          # Функция печати заголовка
    printf "%-8s %-8s %-5s %s\n" "PID" "PPID" "STATE" "COMMAND"           # Форматированный вывод
}

print_header                                                              # Вызов функции

for pid_dir in /proc/[0-9]*; do                                           # Цикл по всем PID в /proc
    pid="${pid_dir##*/}"                                                  # Извлекаем PID из пути
    status_file="/proc/$pid/status"                                       # Путь к status
    [[ ! -r "$status_file" ]] && continue                                 # Пропускаем, если status не читается
    ppid=$(awk -F':[ \t]+' '/^PPid:/ {print $2}' "$status_file" 2>/dev/null)  # Извлекаем PPID
    state=$(awk -F':[ \t]+' '/^State:/ {print $2}' "$status_file" 2>/dev/null | cut -d' ' -f1)  # Извлекаем состояние (букву)
    name=$(awk -F':[ \t]+' '/^Name:/ {print $2}' "$status_file" 2>/dev/null)  # Извлекаем имя процесса
    [[ -z "$name" ]] && name="-"                                          # Если имя пусто, ставим прочерк
    printf "%-8s %-8s %-5s %s\n" "$pid" "$ppid" "$state" "$name"          # Выводим строку процесса
done | sort -n                                                            # Сортируем по PID
```
</details>

Проверяем:
```bash
sudo ./ps_ax.sh
```

<details> 
<summary>вывод скрипта</summary>

```bash
root@client:/home/user# ./ps_ax.sh 
PID      PPID     STATE COMMAND
1        0        S     systemd
2        0        S     kthreadd
3        2        S     pool_workqueue_release
4        2        I     kworker/R-rcu_g
5        2        I     kworker/R-rcu_p
6        2        I     kworker/R-slub_
7        2        I     kworker/R-netns
8        2        I     kworker/0:0-events
12       2        I     kworker/R-mm_pe
13       2        I     rcu_tasks_kthread
14       2        I     rcu_tasks_rude_kthread
15       2        I     rcu_tasks_trace_kthread
16       2        S     ksoftirqd/0
17       2        I     rcu_preempt
18       2        S     migration/0
19       2        S     idle_inject/0
20       2        S     cpuhp/0
21       2        S     kdevtmpfs
22       2        I     kworker/R-inet_
23       2        S     kauditd
24       2        S     khungtaskd
26       2        S     oom_reaper
28       2        I     kworker/R-write
29       2        S     kcompactd0
30       2        S     ksmd
31       2        S     khugepaged
32       2        I     kworker/R-kinte
33       2        I     kworker/R-kbloc
34       2        I     kworker/R-blkcg
35       2        S     irq/9-acpi
36       2        I     kworker/R-tpm_d
37       2        I     kworker/R-ata_s
38       2        I     kworker/R-md
39       2        I     kworker/R-md_bi
40       2        I     kworker/R-edac-
41       2        I     kworker/R-devfr
42       2        S     watchdogd
43       2        I     kworker/R-quota
44       2        I     kworker/0:1H-kblockd
45       2        S     kswapd0
46       2        S     ecryptfs-kthread
47       2        I     kworker/R-kthro
48       2        I     kworker/R-acpi_
49       2        S     scsi_eh_0
50       2        I     kworker/R-scsi_
51       2        S     scsi_eh_1
52       2        I     kworker/R-scsi_
56       2        I     kworker/R-mld
57       2        I     kworker/R-ipv6_
65       2        I     kworker/R-kstrp
67       2        I     kworker/u3:0
72       2        I     kworker/R-crypt
82       2        I     kworker/R-charg
124      2        S     scsi_eh_2
125      2        I     kworker/R-scsi_
126      2        S     scsi_eh_3
127      2        I     kworker/R-scsi_
128      2        S     scsi_eh_4
130      2        I     kworker/R-scsi_
131      2        S     scsi_eh_5
137      2        I     kworker/R-scsi_
138      2        S     scsi_eh_6
139      2        I     kworker/R-scsi_
140      2        S     scsi_eh_7
141      2        I     kworker/R-scsi_
142      2        S     scsi_eh_8
143      2        I     kworker/R-scsi_
144      2        S     scsi_eh_9
148      2        I     kworker/R-scsi_
149      2        S     scsi_eh_10
150      2        I     kworker/R-scsi_
154      2        I     kworker/u2:6-events_power_efficient
155      2        I     kworker/u2:7-events_power_efficient
239      2        I     kworker/R-kdmfl
268      2        I     kworker/R-raid5
307      2        S     jbd2/dm-0-8
308      2        I     kworker/R-ext4-
381      1        S     systemd-journal
406      2        I     kworker/R-kmpat
407      2        I     kworker/R-kmpat
432      1        S     multipathd
465      1        S     systemd-udevd
466      2        S     psimon
542      2        I     kworker/0:2H-kblockd
694      2        S     jbd2/sda2-8
695      2        I     kworker/R-ext4-
701      2        S     spl_system_task
702      2        S     spl_delay_taskq
703      2        S     spl_dynamic_tas
704      2        S     spl_kmem_cache
730      2        S     zvol
731      2        S     arc_prune
732      2        S     arc_evict
733      2        S     arc_reap
734      2        S     dbu_evict
735      2        S     dbuf_evict
736      2        S     z_vdev_file
739      2        S     irq/18-vmwgfx
740      2        I     kworker/R-ttm
761      2        S     l2arc_feed
895      2        S     z_vdev_file
896      2        S     z_vdev_file
897      2        S     z_vdev_file
898      2        S     z_vdev_file
899      2        S     z_vdev_file
903      2        S     z_vdev_file
904      2        S     z_vdev_file
905      2        S     z_vdev_file
1065     2        S     z_null_iss
1066     2        S     z_null_int
1067     2        S     z_rd_iss
1068     2        S     z_rd_int
1069     2        S     z_wr_iss
1070     2        S     z_wr_iss_h
1071     2        S     z_wr_int
1072     2        S     z_wr_int_h
1073     2        S     z_fr_iss
1074     2        S     z_fr_int
1075     2        S     z_cl_iss
1076     2        S     z_cl_int
1077     2        S     z_ioctl_iss
1078     2        S     z_ioctl_int
1079     2        S     z_trim_iss
1080     2        S     z_trim_int
1081     2        S     z_zvol
1082     2        S     z_metaslab
1083     2        S     z_prefetch
1084     2        S     z_upgrade
1091     2        S     z_vdev_file
1092     2        S     z_vdev_file
1093     2        S     z_vdev_file
1094     2        S     z_vdev_file
1095     2        S     z_vdev_file
1096     2        S     z_vdev_file
1097     2        S     z_vdev_file
1098     2        S     dp_sync_taskq
1099     2        S     dp_zil_clean_ta
1100     2        S     z_zrele
1101     2        S     z_unlinked_drai
1133     2        S     txg_quiesce
1134     2        S     txg_sync
1135     2        S     mmp
1136     2        S     z_indirect_cond
1137     2        S     z_livelist_dest
1138     2        S     z_livelist_cond
1139     2        S     z_checkpoint_di
1141     2        S     z_null_iss
1142     2        S     z_null_int
1143     2        S     z_rd_iss
1144     2        S     z_rd_int
1145     2        S     z_wr_iss
1146     2        S     z_wr_iss_h
1147     2        S     z_wr_int
1148     2        S     z_wr_int_h
1149     2        S     z_fr_iss
1150     2        S     z_fr_int
1151     2        S     z_cl_iss
1152     2        S     z_cl_int
1153     2        S     z_ioctl_iss
1154     2        S     z_ioctl_int
1155     2        S     z_trim_iss
1156     2        S     z_trim_int
1157     2        S     z_zvol
1158     2        S     z_metaslab
1159     2        S     z_prefetch
1160     2        S     z_upgrade
1167     2        S     dp_sync_taskq
1168     2        S     dp_zil_clean_ta
1169     2        S     z_zrele
1170     2        S     z_unlinked_drai
1202     2        S     txg_quiesce
1203     2        S     txg_sync
1204     2        S     mmp
1205     2        S     z_indirect_cond
1206     2        S     z_livelist_dest
1207     2        S     z_livelist_cond
1208     2        S     z_checkpoint_di
1216     2        S     z_null_iss
1217     2        S     z_null_int
1218     2        S     z_rd_iss
1219     2        S     z_rd_int
1220     2        S     z_wr_iss
1221     2        S     z_wr_iss_h
1222     2        S     z_wr_int
1223     2        S     z_wr_int_h
1224     2        S     z_fr_iss
1225     2        S     z_fr_int
1226     2        S     z_cl_iss
1227     2        S     z_cl_int
1228     2        S     z_ioctl_iss
1229     2        S     z_ioctl_int
1230     2        S     z_trim_iss
1231     2        S     z_trim_int
1232     2        S     z_zvol
1233     2        S     z_metaslab
1234     2        S     z_prefetch
1235     2        S     z_upgrade
1242     2        S     dp_sync_taskq
1243     2        S     dp_zil_clean_ta
1244     2        S     z_zrele
1245     2        S     z_unlinked_drai
1277     2        S     txg_quiesce
1278     2        S     txg_sync
1279     2        S     mmp
1280     2        S     z_indirect_cond
1281     2        S     z_livelist_dest
1282     2        S     z_livelist_cond
1283     2        S     z_checkpoint_di
1291     2        S     z_null_iss
1292     2        S     z_null_int
1293     2        S     z_rd_iss
1294     2        S     z_rd_int
1295     2        S     z_wr_iss
1296     2        S     z_wr_iss_h
1297     2        S     z_wr_int
1298     2        S     z_wr_int_h
1299     2        S     z_fr_iss
1300     2        S     z_fr_int
1301     2        S     z_cl_iss
1302     2        S     z_cl_int
1303     2        S     z_ioctl_iss
1304     2        S     z_ioctl_int
1305     2        S     z_trim_iss
1306     2        S     z_trim_int
1307     2        S     z_zvol
1308     2        S     z_metaslab
1309     2        S     z_prefetch
1310     2        S     z_upgrade
1317     2        S     dp_sync_taskq
1318     2        S     dp_zil_clean_ta
1319     2        S     z_zrele
1320     2        S     z_unlinked_drai
1352     2        S     txg_quiesce
1353     2        S     txg_sync
1354     2        S     mmp
1355     2        S     z_indirect_cond
1356     2        S     z_livelist_dest
1357     2        S     z_livelist_cond
1358     2        S     z_checkpoint_di
1362     2        S     z_null_iss
1363     2        S     z_null_int
1364     2        S     z_rd_iss
1365     2        S     z_rd_int
1366     2        S     z_wr_iss
1367     2        S     z_wr_iss_h
1368     2        S     z_wr_int
1369     2        S     z_wr_int_h
1370     2        S     z_fr_iss
1371     2        S     z_fr_int
1372     2        S     z_cl_iss
1373     2        S     z_cl_int
1374     2        S     z_ioctl_iss
1376     2        S     z_ioctl_int
1377     2        S     z_trim_iss
1378     2        S     z_trim_int
1379     2        S     z_zvol
1380     2        S     z_metaslab
1381     2        S     z_prefetch
1382     2        S     z_upgrade
1392     2        S     dp_sync_taskq
1393     2        S     dp_zil_clean_ta
1394     2        S     z_zrele
1395     2        S     z_unlinked_drai
1427     2        S     txg_quiesce
1428     2        S     txg_sync
1429     2        S     mmp
1430     2        S     z_indirect_cond
1431     2        S     z_livelist_dest
1432     2        S     z_livelist_cond
1433     2        S     z_checkpoint_di
1495     1        S     systemd-network
1512     1        S     rpcbind
1517     1        S     systemd-resolve
1522     1        S     systemd-timesyn
1611     2        I     kworker/R-rpcio
1612     2        I     kworker/R-xprti
1617     2        I     kworker/R-cfg80
1622     1        S     dbus-daemon
1639     1        S     polkitd
1676     1        S     systemd-logind
1678     1        S     udisksd
1691     1        S     zed
1796     1        S     rsyslogd
1847     1        S     cron
1900     1        S     unattended-upgr
1935     1        S     nginx
1937     1935     S     nginx
1945     1        S     ModemManager
1992     1        S     agetty
2162     1        S     sshd
2163     2162     S     sshd
2168     2        S     psimon
2170     1        S     systemd
2171     2170     S     (sd-pam)
2280     2163     S     sshd
2281     2280     S     bash
2324     2        I     kworker/R-tls-s
2326     2281     S     sudo
2327     2326     S     sudo
2328     2327     S     bash
10719    1        S     fwupd
10726    1        S     upowerd
15137    2        I     kworker/0:2-cgwb_release
15138    2        I     kworker/u2:0-flush-252:0
21666    2        I     kworker/u2:1-events_power_efficient
23846    2328     S     ps_ax.sh
23847    23846    S     ps_ax.sh
23848    23846    S     sort
```

</details>