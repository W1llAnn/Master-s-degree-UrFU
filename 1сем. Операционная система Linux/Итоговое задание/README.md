# Итоговое задание


## ШАГ 1. Подготовка виртуальной машины

Я подняла на Oracle VM VirtualBox чистую систему Linux:

![VM VirtualBox Mint](VM_VirtualBox_Mint.png)

С оперативной памятью 2048 мб чтобы ускорить срабатывание OOM-killer

Устанавливаем необходимые пакеты:  

```
sudo apt update
sudo apt install -y htop stress-ng sysstat dmesg
```


## ШАГ 2. Имитация процесса с утечкой памяти

Создадим простой скрипт, который будет потреблять память без освобождения:

1. Файл 

```
nano ~/memory_leak.sh
```

2. Помещаем в него следующее:

```
i=0
while true; do
    dd if=/dev/zero of=/dev/shm/memleak_$i bs=10M count=1 2>/dev/null
    ((i++))
    sleep 2
done
```
Постепенно аллоцируем память по 10 МБ каждые 2 секунды

3. Делаем скрипт исполняемым и запускаем в фоне:

```
chmod +x ~/memory_leak.sh
nohup ~/memory_leak.sh > /dev/null 2>&1 &
```

Что происходит. Этот скрипт будет заполнять tmpfs (/dev/shm), что быстро исчерпает RAM и вызовет использование swap, а затем — OOM-killer.


## ШАГ 3. Настройка автоматического мониторинга (раз в 5 минут)

1. Создадим скрипт мониторинга и добавим его в cron

```
nano ~/monitor.sh
```

```
LOG_DIR="/var/log/system_monitor"
mkdir -p "$LOG_DIR"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")

echo "=== MEMORY ===" > "$LOG_DIR/monitor_$TIMESTAMP.log"
free -h >> "$LOG_DIR/monitor_$TIMESTAMP.log"

echo -e "\n=== CPU LOAD ===" >> "$LOG_DIR/monitor_$TIMESTAMP.log"
uptime >> "$LOG_DIR/monitor_$TIMESTAMP.log"
mpstat 1 1 >> "$LOG_DIR/monitor_$TIMESTAMP.log" 2>/dev/null || echo "mpstat not available" >> "$LOG_DIR/monitor_$TIMESTAMP.log"

echo -e "\n=== SWAP USAGE ===" >> "$LOG_DIR/monitor_$TIMESTAMP.log"
swapon --show >> "$LOG_DIR/monitor_$TIMESTAMP.log"
cat /proc/swaps >> "$LOG_DIR/monitor_$TIMESTAMP.log"

echo -e "\n=== CRITICAL KERNEL MESSAGES (last 50 lines) ===" >> "$LOG_DIR/monitor_$TIMESTAMP.log"
dmesg -T | tail -n 50 >> "$LOG_DIR/monitor_$TIMESTAMP.log"

echo -e "\n=== OOM-KILLER EVENTS ===" >> "$LOG_DIR/monitor_$TIMESTAMP.log"
dmesg -T | grep -i "oom\|killed process" >> "$LOG_DIR/monitor_$TIMESTAMP.log" || echo "No OOM events found" >> "$LOG_DIR/monitor_$TIMESTAMP.log"
```

2. Делаем скрипт исполняемым

```
chmod +x ~/monitor.sh
```

3. Добавляем в  в cron выполнение раз в 5 минут

```
crontab -e
```

И добавляем строку

```
*/5 * * * * /home/mint/monitor.sh
```


## ШАГ 4. Анализ событий ядра и OOM-killer

Помимо регулярного сбора через dmesg, мы докинем логирование ядра в journalctl или /var/log/kern.log

1. Проверяем наличие OOM-событий в реальном времени:

```
dmesg -T | grep -i "killed process"
```

2. Чтобы убедиться, что ядро логирует события

Но сначала установим rsyslog: 

```
sudo apt install rsyslog -y
sudo systemctl enable --now rsyslog
```

```
sudo cat /var/log/kern.log | grep -i "oom\|killed"
```


## ШАГ 5. Генерация диагностического отчёта

Создаем скрипт, собирающий все логи в один отчёт

```
nano ~/generate_report.sh
```

И его содержимое

```
REPORT="/home/$(whoami)/diagnostic_report_$(date +%Y%m%d_%H%M%S).txt"

echo "DIAGNOSTIC REPORT - $(date)" > "$REPORT"
echo "========================================" >> "$REPORT"

echo -e "\n[1] MEMORY STATUS" >> "$REPORT"
free -h >> "$REPORT"

echo -e "\n[2] CPU LOAD" >> "$REPORT"
uptime >> "$REPORT"
mpstat 1 1 >> "$REPORT" 2>/dev/null

echo -e "\n[3] SWAP STATUS" >> "$REPORT"
swapon --show >> "$REPORT"
cat /proc/swaps >> "$REPORT"

echo -e "\n[4] LAST 100 KERNEL MESSAGES" >> "$REPORT"
dmesg -T | tail -n 100 >> "$REPORT"

echo -e "\n[5] OOM-KILLER EVENTS (ALL)" >> "$REPORT"
dmesg -T | grep -i "killed process" >> "$REPORT"

echo -e "\n[6] MONITOR LOGS (last 3 entries)" >> "$REPORT"
ls -t /var/log/system_monitor/monitor_*.log | head -n 3 | xargs -I {} sh -c 'echo "=== {} ==="; cat {}' >> "$REPORT"

echo -e "\n[7] RUNNING PROCESSES (top 10 by memory)" >> "$REPORT"
ps aux --sort=-%mem | head -n 11 >> "$REPORT"

echo "Report saved to: $REPORT"
```

Делаем скрипт исполняемым и запускаем

```
chmod +x ~/generate_report.sh
~/generate_report.sh
```

## ШАГ 6. Останавливаем процесс утечки 

```
pkill -f memory_leak.sh
```