# Лабораторная 13: Пошаговый сценарий (VirtualBox + Ubuntu + PHP + MySQL)

Этот файл сделан как чеклист: идешь сверху вниз и отмечаешь выполненные пункты.

---

## 1) Что нужно заранее

- [ ] Установлен `VirtualBox`
- [ ] Скачан ISO-образ Ubuntu (например, `ubuntu-24.04.4-live-server-amd64.iso`)
- [ ] Есть интернет в виртуальной машине

---

## 2) Создание виртуальной машины

- [ ] Открыть VirtualBox -> `New`
- [ ] Имя: `Lab13-Ubuntu`
- [ ] RAM: `4096-8192 MB` (если ПК слабый, ставь 4096)
- [ ] CPU: `2`
- [ ] Disk: `25-30 GB`, формат `VDI`, `Dynamically allocated`
- [ ] Подключить ISO Ubuntu
- [ ] Сеть: `NAT` (проще всего для начала)

---

## 3) Установка Ubuntu

- [ ] Язык: `English`
- [ ] Keyboard: `English (US)`
- [ ] Разметка: автоматическая (чтобы не усложнять)
- [ ] Дождаться завершения установки и перезагрузки
- [ ] Войти под своим пользователем

---

## 4) Обновление системы

Выполнить в терминале:

```bash
sudo apt update && sudo apt upgrade -y
```

- [ ] Обновление завершилось без критических ошибок

---

## 5) Установка LAMP (Apache + PHP + MySQL)

```bash
sudo apt install -y lamp-server^
```

Проверка:

```bash
php -v
mysql --version
systemctl status apache2 --no-pager
systemctl status mysql --no-pager
```

- [ ] Apache запущен
- [ ] MySQL запущен
- [ ] PHP доступен

---

## 6) Создание базы данных и пользователя (основной вариант: с доступом из Workbench на Windows)

Запуск MySQL:

```bash
sudo mysql
```

Внутри MySQL выполнить:

```sql
CREATE DATABASE lab13_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'lab13_user'@'%' IDENTIFIED BY 'Lab13Pass!123';
GRANT ALL PRIVILEGES ON lab13_db.* TO 'lab13_user'@'%';
FLUSH PRIVILEGES;
USE lab13_db;

CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

EXIT;
```

Примечание:

- `%` в `@'%'` разрешает подключение с других хостов (например, Workbench на Windows).
- Если нужен только локальный доступ внутри Ubuntu, можно использовать `@'localhost'`.

- [ ] База `lab13_db` создана
- [ ] Таблица `users` создана

### 6A) То же самое через MySQL Workbench (визуально)

Если удобнее, БД и таблицу можно создать полностью мышкой.

1. Открыть подключение в Workbench.
2. Слева `Schemas` -> ПКМ -> `Create Schema...`
3. Имя схемы: `lab13_db` -> `Apply` -> `Apply` -> `Finish`.
4. Открыть `lab13_db` -> `Tables` -> ПКМ -> `Create Table...`
5. Имя таблицы: `users`.
6. Добавить поля:
   - `id` -> тип `INT`, включить `PK`, `NN`, `AI`
   - `username` -> тип `VARCHAR(50)`, включить `NN`, `UQ`
   - `password` -> тип `VARCHAR(255)`, включить `NN`
   - `created_at` -> тип `TIMESTAMP`, Default/Expression: `CURRENT_TIMESTAMP`
7. Нажать `Apply` -> Workbench покажет SQL -> снова `Apply`.

Как скопировать SQL для отчета:

- в окне `Apply SQL Script to Database` скопировать скрипт перед применением;
- или ПКМ по таблице `users` -> `Copy to Clipboard` -> `Create Statement`.

Примечание:

- пользователя БД (`CREATE USER` и `GRANT`) удобнее создать SQL-командами в Query Tab, как в шаге 6.

---

## 7) Подготовка папки проекта

```bash
sudo mkdir -p /var/www/html/lab13
sudo chown -R $USER:$USER /var/www/html/lab13
```

- [ ] Папка проекта создана

---

## 8) Создание файлов проекта

Создай 4 файла в `/var/www/html/lab13`:

- [ ] `config.php`
- [ ] `register.php`
- [ ] `login.php`
- [ ] `index.php`

### `config.php`

```php
<?php
$host = 'localhost';
$db   = 'lab13_db';
$user = 'lab13_user';
$pass = 'Lab13Pass!123';
$charset = 'utf8mb4';

$dsn = "mysql:host=$host;dbname=$db;charset=$charset";

try {
    $pdo = new PDO($dsn, $user, $pass, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION
    ]);
} catch (PDOException $e) {
    die("Ошибка подключения к БД: " . $e->getMessage());
}
```

### `register.php`

```php
<?php
require 'config.php';

$message = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username = trim($_POST['username'] ?? '');
    $password = trim($_POST['password'] ?? '');

    if ($username === '' || $password === '') {
        $message = "Заполните все поля.";
    } else {
        $hash = password_hash($password, PASSWORD_DEFAULT);
        $stmt = $pdo->prepare("INSERT INTO users (username, password) VALUES (?, ?)");
        try {
            $stmt->execute([$username, $hash]);
            $message = "Регистрация успешна.";
        } catch (PDOException $e) {
            $message = "Пользователь уже существует или ошибка БД.";
        }
    }
}
?>
<!doctype html>
<html lang="ru">
<head><meta charset="UTF-8"><title>Регистрация</title></head>
<body>
<h2>Регистрация</h2>
<form method="post">
  <input name="username" placeholder="Логин"><br><br>
  <input name="password" type="password" placeholder="Пароль"><br><br>
  <button type="submit">Зарегистрироваться</button>
</form>
<p><?= htmlspecialchars($message) ?></p>
<p><a href="login.php">Перейти ко входу</a></p>
</body>
</html>
```

### `login.php`

```php
<?php
require 'config.php';
session_start();

$message = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username = trim($_POST['username'] ?? '');
    $password = trim($_POST['password'] ?? '');

    $stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
    $stmt->execute([$username]);
    $user = $stmt->fetch(PDO::FETCH_ASSOC);

    if ($user && password_verify($password, $user['password'])) {
        $_SESSION['username'] = $user['username'];
        header("Location: index.php");
        exit;
    } else {
        $message = "Неверный логин или пароль.";
    }
}
?>
<!doctype html>
<html lang="ru">
<head><meta charset="UTF-8"><title>Вход</title></head>
<body>
<h2>Вход</h2>
<form method="post">
  <input name="username" placeholder="Логин"><br><br>
  <input name="password" type="password" placeholder="Пароль"><br><br>
  <button type="submit">Войти</button>
</form>
<p><?= htmlspecialchars($message) ?></p>
<p><a href="register.php">Регистрация</a></p>
</body>
</html>
```

### `index.php`

```php
<?php
session_start();
if (!isset($_SESSION['username'])) {
    header("Location: login.php");
    exit;
}
?>
<!doctype html>
<html lang="ru">
<head><meta charset="UTF-8"><title>Главная</title></head>
<body>
<h2>Добро пожаловать, <?= htmlspecialchars($_SESSION['username']) ?>!</h2>
<p>Вы успешно вошли в систему.</p>
</body>
</html>
```

---

## 9) Проверка работы приложения

Открыть в браузере:

- [ ] `http://localhost/lab13/register.php`
- [ ] Зарегистрировать пользователя
- [ ] Войти через `login.php`
- [ ] Увидеть приветствие на `index.php`

---

## 10) Проверка данных в БД

```bash
mysql -u lab13_user -p
```

Пароль: `Lab13Pass!123`

```sql
USE lab13_db;
SELECT id, username, created_at FROM users;
```

- [ ] Пользователь появился в таблице `users`

---

## 11) Что показать на защите (быстрый план)

1. Параметры ВМ в VirtualBox (RAM/CPU/Disk/Network)
2. Ubuntu запущена, показать IP (`hostname -I`)
3. Проверка сервисов (`apache2`, `mysql`)
4. Регистрация пользователя в веб-форме
5. Вход в систему
6. Показ записи в таблице `users`

---

## 12) Типичные проблемы и решения

### Ошибка подключения к БД

- Проверь `config.php` (имя БД, логин, пароль)
- Проверь, запущен ли MySQL:

```bash
systemctl status mysql --no-pager
```

### Белая страница в PHP

- Проверь синтаксис файлов
- Проверь права доступа к папке проекта

### Apache не открывает страницу

- Проверка:

```bash
systemctl status apache2 --no-pager
```

- Перезапуск:

```bash
sudo systemctl restart apache2
```

---

## 13) Workbench с Windows не подключается к MySQL в Ubuntu (важно)

Если в Workbench ошибка вида `Unable to connect to <IP>:3306`, проверь 4 вещи ниже.

### 13.1 Почему SQL-команды не меняют `mysqld.cnf`

Команды внутри `mysql>`:

- `CREATE USER`
- `GRANT`
- `CREATE TABLE`

меняют объекты внутри БД (пользователи, права, таблицы), но не меняют системный конфиг сервера MySQL.

Файл `mysqld.cnf` меняется только вручную (через редактор) и отвечает за сетевое поведение сервера.

### 13.2 Разреши MySQL слушать внешние подключения

Открыть конфиг:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Найти и изменить:

```ini
bind-address = 127.0.0.1
mysqlx-bind-address = 127.0.0.1
```

на:

```ini
bind-address = 0.0.0.0
mysqlx-bind-address = 0.0.0.0
```

Перезапуск:

```bash
sudo systemctl restart mysql
sudo systemctl status mysql --no-pager
```

### 13.3 Создай пользователя для подключения по сети

Пользователь вида `'name'@'localhost'` работает только внутри Ubuntu.

Для подключения из Workbench (Windows) нужен пользователь, например:

```sql
CREATE USER 'leh'@'%' IDENTIFIED BY 'leh123_!';
GRANT ALL PRIVILEGES ON demodb.* TO 'leh'@'%';
FLUSH PRIVILEGES;
```

### 13.4 Проверь firewall (если включен UFW)

```bash
sudo ufw status
sudo ufw allow 3306/tcp
```

### 13.5 Правильные поля в Workbench

- `Hostname`: IP Ubuntu (например, `192.168.0.107`)
- `Port`: `3306`
- `Username`: пользователь из п. 13.3 (например, `leh`)
- `Password`: пароль этого пользователя
- `Default Schema`: `demodb` (можно оставить пусто)

---

## 14) Мини-шпаргалка команд

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y lamp-server^
php -v
mysql --version
systemctl status apache2 --no-pager
systemctl status mysql --no-pager
sudo mysql
```

---

Готово. Если хочешь, следующим шагом сделаю второй файл: короткий "сценарий ответа на защите" на 1-2 минуты.
