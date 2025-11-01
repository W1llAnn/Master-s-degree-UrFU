# SSH Setup Lab

## Выполненные шаги

1. **Установка SSH-сервера**  

Обновить список пакетов  
```
sudo apt update
```

Установить OpenSSH-сервер  
```
sudo apt install -y openssh-server
```

Запустить службу SSH (если не запущена автоматически)  
```
sudo systemctl start ssh
```

Включить автозапуск при загрузке  
```
sudo systemctl enable ssh
```

Проверить статус службы  
```
sudo systemctl status ssh
```

2. **Подключение по SSH**  
```
ssh -p PORT anna@95.64.227.126
```
Далее пароль

3. **Копирование файлов через SCP**  
Создаем текстовый файл  
```
echo "Hello from local" > local_file.txt
```

Копируем на сервер  
```
scp local_file.txt anna@95.64.227.126:/home/anna/
```

4. **Настройка SSH-ключей**  
Остальное я напишу как примеры команд так как ВМ корпоративная  

```
ssh-copy-id -f -p 22 root@192.168.1.30
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
ssh-copy-id user@192.168.1.100
```

Потом небольшая возня с правами у пользователя на сервере...


Проверка безпарольного входа
```
ssh -p PORT anna@95.64.227.126
```
Пароль не запрашивается и выводится терминал