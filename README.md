

# Лабораторная работа 6: Файловые системы FUSE

Этот репозиторий содержит реализацию трёх FUSE файловых систем на основе одного бинарника:

- **Passthrough FS** — прозрачное проксирование реальной директории (Задание A).
- **ROT13 FS** — шифрует содержимое текстовых файлов ROT13 на диске (Задание B).
- **Uppercase FS** — при чтении отображает содержимое файлов в верхнем регистре (Задание C).

Режим выбирается переменной окружения `MYFUSE_MODE`.

---

## Структура проекта

```text
lab6/
├── src/
│   ├── myfuse.c        # Точка входа, парсинг аргументов, выбор режима
│   ├── operations.c    # Реализация file operations FUSE
│   ├── operations.h    # Общие структуры и объявления
│   └── utils.c         # Вспомогательные функции: логирование, построение путей, ROT13/uppercase
├── Makefile
├── README.md
└── REPORT.md           # Отчёт по лабораторной работе
````

---

## Зависимости

Проект проверен в среде:

* **ОС:** Ubuntu в VirtualBox
* **Ядро:** `Linux 6.14.0-35-generic`
* **ФС для `/tmp`:** `tmpfs`

Необходимые пакеты (Ubuntu/Debian):

```bash
sudo apt update
sudo apt install -y pkg-config libfuse3-dev
```

---

## Сборка

В корне проекта:

```bash
cd lab6
make
```

В результате будет собран бинарник:

```text
./myfuse
```

Очистка:

```bash
make clean
```

---

## Подготовка директорий

Реальная директория-источник и точка монтирования:

```bash
sudo mkdir -p /mnt/fuse
sudo chown "$USER":"$USER" /mnt/fuse
mkdir -p /tmp/source
```

---

## Режимы работы

Один и тот же бинарник используется для всех трёх заданий.
Режим задаётся переменной окружения `MYFUSE_MODE`:

* не задана (`unset`) — **Passthrough FS** (Задание A)
* `MYFUSE_MODE=rot13` — **ROT13 FS** (Задание B)
* `MYFUSE_MODE=upper` — **Uppercase FS** (Задание C)

Общий формат запуска:

```bash
./myfuse <source_dir> <mountpoint>
# пример:
./myfuse /tmp/source /mnt/fuse
```

---

## Задание A: Passthrough FUSE filesystem

### Запуск

```bash
cd lab6
./myfuse /tmp/source /mnt/fuse
```

(Процесс демонезируется через FUSE, поэтому приглашение shell может вернуться сразу.)

### Пример тестирования

```bash
echo "test" > /mnt/fuse/file.txt
cat /mnt/fuse/file.txt
ls -l /mnt/fuse
ls -l /tmp/source
rm /mnt/fuse/file.txt
ls -l /mnt/fuse
ls -l /tmp/source
```

Ожидаемое поведение:

* Файл `file.txt` создаётся и виден как в `/mnt/fuse`, так и в `/tmp/source`.
* Содержимое одинаково при чтении через FUSE и напрямую.

---

## Задание B: ROT13 Encryption Filesystem

### Запуск

```bash
cd lab6
MYFUSE_MODE=rot13 ./myfuse /tmp/source /mnt/fuse
```

### Пример тестирования

```bash
echo "Hello World" > /mnt/fuse/secret.txt

echo "Реальный файл (на /tmp/source):"
cat /tmp/source/secret.txt

echo "Через FUSE:"
cat /mnt/fuse/secret.txt
```

Фактический результат испытаний:

```text
Реальный файл (на /tmp/source):
Uryyb Jbeyq
Через FUSE:
Hello World
```

* На **реальной ФС** (`/tmp/source`) файл хранится в виде ROT13 (`Uryyb Jbeyq`).
* При чтении через FUSE (`/mnt/fuse`) содержимое автоматически расшифровывается.

---

## Задание C: Uppercase Filesystem

### Запуск

```bash
cd lab6
MYFUSE_MODE=upper ./myfuse /tmp/source /mnt/fuse
```

### Пример тестирования

```bash
echo "hello world" > /tmp/source/test_upper.txt

echo "Обычное чтение:"
cat /tmp/source/test_upper.txt

echo "Через FUSE:"
cat /mnt/fuse/test_upper.txt
```

Фактический результат:

```text
Обычное чтение:
hello world
Через FUSE:
HELLO WORLD
```

* На диске данные хранятся в исходном виде.
* При чтении через FUSE всё содержимое переводится в верхний регистр.

---

## Нагрузочное тестирование (кратко)

Бенчмарки проводились в режиме **passthrough** (`MYFUSE_MODE` не задана), сравнивались:

* `/mnt/fuse` — FUSE FS
* `/tmp/source` — реальная FS (в данном стенде `tmpfs`)

Примеры использованных команд:

```bash
# Latency (read 4KB x 50000)
 /usr/bin/time -p dd if=/mnt/fuse/testfile of=/dev/null bs=4096 count=50000
 /usr.bin/time -p dd if=/tmp/source/testfile of=/dev/null bs=4096 count=50000

# Throughput (1MB, 10MB, 100MB)
 /usr/bin/time -p dd if=/dev/zero of=/mnt/fuse/throughput_100M_fuse bs=1M count=100
 /usr/bin/time -p dd if=/dev/zero of=/tmp/source/throughput_100M_ext4 bs=1M count=100

# IOPS (1000 маленьких файлов)
 /usr/bin/time -p bash -c 'for i in $(seq 1 1000); do echo "data" > /mnt/fuse/iops_fuse/file_$i; done'
 /usr/bin/time -p bash -c 'for i in $(seq 1 1000); do echo "data" > /tmp/source/iops_ext4/file_$i; done'
```

Подробные результаты и анализ приведены в REPORT.md

---

## Размонтирование

После завершения работы:

```bash
fusermount3 -u /mnt/fuse 2>/dev/null || fusermount -u /mnt/fuse 2>/dev/null || sudo umount /mnt/fuse
```

---

## Отчёт

Подробное описание реализации, хода выполнения, результатов тестирования и ответы на контрольные вопросы находятся в файле:

* REPORT.md
