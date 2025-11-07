Ниже — готовая «памятка» под твою задачу: домашний фото/видео-архив, macOS + Linux, большие объёмы на HDD, аккуратный прогресс и обязательная проверка целостности.

Я буду использовать пример каталога `202511` (фото/видео за ноябрь 2025).

---

## 0. Исходные условия и инструменты

**Цель:**
есть каталог `/SOURCE/202511`, его надо надёжно скопировать на `/DEST/202511` и иметь файл с хэшами для будущих проверок.

**CLI-инструменты, которые будем использовать:**

1. **rsync ≥ 3.x** — основное средство копирования, с однострочным прогрессом и возможностью контрольной проверки по хэшам. ([man7.org][1])
2. **hashdeep (из пакета md5deep)** — кроссплатформенный инструмент для рекурсивных хэшей и последующего аудита (выявляет изменённые/пропавшие/новые файлы). ([forensics.wiki][2])
3. **b3sum (BLAKE3)** — опционально, если захочешь сверхбыстрые хэши, но мы его оставим как бонус. ([GitHub][3])

### Установка (один раз)

**macOS (Homebrew):**

```bash
brew install rsync hashdeep b3sum
```

macOS по умолчанию ставит древний rsync 2.6.9, поэтому нужен установочный rsync через Homebrew; ты уже поставил себе 3.4.1 — отлично.([dribin.org][4])

**Linux (Debian/Ubuntu):**

```bash
sudo apt update
sudo apt install rsync hashdeep b3sum
```

hashdeep/md5deep и b3sum есть в репозиториях большинства дистрибутивов как кроссплатформенные утилиты для массовых хэшей. ([forensics.wiki][2])

---

## 1. Подготовка перед копированием

### 1.1 Проверить, где что лежит и хватит ли места

```bash
# macOS
diskutil list
df -h

# Linux
lsblk
df -h
```

Убедись, что:

* исходный каталог, например:
  `SRC=/Users/qbook/Photos/202511`
* целевой диск смонтирован, например:
  `DST=/Volumes/ARCH20TB/Photos/202511`

Проверь, что на `DST` есть запас по размеру (лучше +20–30% свободного места).

---

## 2. Копирование rsync’ом с аккуратным прогрессом

### 2.1 Базовая команда: «тихий» progress-bar в одну строку

```bash
rsync -aHAX --human-readable \
      --info=progress2,name0 --no-inc-recursive \
      --partial --inplace \
      "$SRC/"  "$DST/"
```

**Разбор ключей:**

* `-aHAX` — архивный режим: рекурсия, права, время, hard-links, ACL, xattrs. ([man7.org][1])
* `--human-readable` / `-h` — размеры и скорость в GiB/MiB. ([ittavern.com][5])
* `--info=progress2` — общий прогресс по всему объёму (одна строка: общий % / скорость / ETA). ([Ask Ubuntu][6])
* `name0` — отключает вывод имён файлов → не засоряем терминал. ([man7.org][1])
* `--no-inc-recursive` (`--no-i-r`) — сначала строит полный список файлов, потом копирует; благодаря этому `%` и ETA точные и не «скачут». ([Server Fault][7])
* `--partial --inplace` — если копирование прервётся, повторный запуск будет докачивать файлы, а не начинать их сначала. ([man7.org][1])

> Главное: **не добавляй `-v`**, иначе rsync начнёт печатать имя каждого файла, и прогресс перестанет быть «тихим». ([man7.org][1])

### 2.2 Ограничить нагрузку на диск (скорость) — `--bwlimit`

Чтобы не жарить HDD на полную, можно ограничить среднюю скорость чтения/записи:

```bash
rsync -aHAX --human-readable \
      --info=progress2,name0 --no-inc-recursive \
      --partial --inplace \
      --bwlimit=40m \
      "$SRC/" "$DST/"
```

* `--bwlimit=40m` — ограничивает среднюю скорость примерно до 40 MiB/s.
* Опция действует и для локальных копий, ограничивая именно I/O, а не только сетевой трафик. ([Unix & Linux Stack Exchange][8])

### 2.3 Делать «перерывы» для охлаждения диска — `--stop-after`

Современный rsync умеет **сам отключаться** по таймеру:

```bash
rsync -aHAX --human-readable \
      --info=progress2,name0 --no-inc-recursive \
      --partial --inplace \
      --stop-after=60 \
      "$SRC/" "$DST/"
```

* `--stop-after=60` — через 60 минут rsync аккуратно завершится. ([download.samba.org][9])

Дальше можно:

1. Подождать 15–30 минут, пока диск остынет.
2. Запустить **точно ту же команду** ещё раз.
   rsync пропустит уже скопированные файлы и продолжит с места остановки — это как «пошаговое копирование» с передышками.

Если удобнее остановиться по конкретному времени (например, «копировать до 23:00»), есть опция `--stop-at=2025-11-07T23:00`. ([download.samba.org][9])

---

## 3. Проверка целостности средствами rsync

После первого копирования можно сделать **«проверочный проход»**, который ничего не изменяет, но сравнивает содержимое файлов.

### 3.1 Dry-run + checksum

```bash
rsync -aHAX --human-readable \
      --info=progress2,name0 --no-inc-recursive \
      --checksum \
      --dry-run \
      "$SRC/" "$DST/"
```

* `--checksum` (`-c`) — rsync читает оба файла и сравнивает хэш содержимого, а не только время/размер. ([man7.org][1])
* `--dry-run` — **ничего не копирует**, только сообщает, есть ли отличия. ([ittavern.com][5])

Если команда ничего не выводит (или только итог `--stats`), значит файловое дерево на `SRC` и `DST` идентично побайтно.

> Важно: `--checksum` сильно замедляет прогон (нужно прочитать всё с обоих дисков). Это нормально и делается один раз — для первоначальной валидации большого архива. ([ArchWiki][10])

---

## 4. Отдельный файл с хэшами для каталога (hashdeep)

Для долгосрочного домашнего архива лучше иметь **отдельные манифесты хэшей** — по одному на месяц.
Это стандартный приём в цифровой археологии: один раз считаем хэши, файл с манифестом храним рядом и периодически проверяем. ([LFCS Certification Prep eBook][11])

### 4.1 Генерация манифеста SHA-256 для `202511`

Предположим, каталог уже скопирован на архивный HDD и лежит как:

```bash
DST=/Volumes/ARCH20TB/Photos/202511
```

Чтобы пути в манифесте были **относительными**, а не абсолютными:

```bash
cd /Volumes/ARCH20TB/Photos

hashdeep -r -l -c sha256 202511 > 202511.sha256.manifest
```

Разбор:

* `-r` — рекурсивно по подкаталогам. ([Alois Kraus][12])
* `-l` — писать **относительные пути** (от текущего каталога). ([Alois Kraus][12])
* `-c sha256` — считать только SHA-256 (без MD5), современный и достаточно быстрый выбор. ([CSDN Blog][13])

Файл `202511.sha256.manifest` лучше оставить **в том же каталоге, где лежит `202511`**, чтобы любая future-проверка была «самодостаточной».

Формат манифеста hashdeep: заголовок с типами хэшей и дальше строки вида
`<size>,<md5>,<sha256>,<path>`. ([git-annex.branchable.com][14])

### 4.2 Проверка по манифесту через год

Например, через год ты хочешь убедиться, что файлы в `202511` не побились:

```bash
cd /Volumes/ARCH20TB/Photos

hashdeep -a -k 202511.sha256.manifest -r -l 202511
```

* `-a` — **audit-режим**, сверяет текущие файлы с известным списком хэшей.
* `-k manifest` — файл с известными хэшами.
* `-r -l` — рекурсивно и с относительными путями, совместимыми с манифестом. ([Alois Kraus][12])

В случае проблем hashdeep покажет:

* отсутствующие файлы,
* новые файлы,
* файлы, содержимое которых изменилось.

Это как раз то, что нужно для периодической проверки домашнего архива. ([LFCS Certification Prep eBook][11])

---

## 5. Быстрый вариант: BLAKE3 (b3sum) вместо hashdeep

Если когда-нибудь захочешь **максимальную скорость**, можно использовать `b3sum`:

### 5.1 Создать манифест

```bash
cd /Volumes/ARCH20TB/Photos
b3sum -r 202511 > 202511.b3.manifest
```

### 5.2 Проверить

```bash
cd /Volumes/ARCH20TB/Photos
b3sum -c 202511.b3.manifest
```

BLAKE3 сильно быстрее MD5/SHA-2 и криптографически устойчив, а `b3sum` сделан по интерфейсу наподобие `sha256sum`/`md5sum`. ([GitHub][3])

Минус: формат чуть менее «форензик-орiented», чем hashdeep, и нет встроенного аудита «новые / пропавшие / переехавшие» файлы.

---

## 6. Альтернатива rsync: rclone (по желанию)

Если захочешь поэкспериментировать с альтернативой:

* **rclone** — кроссплатформенный CLI-инструмент, умеет копировать локально и на кучу облаков.
* `rclone copy` по умолчанию сравнивает размер+mtime, а на некоторых хранилищах ещё и MD5. ([Rclone][15])
* Есть удобный прогресс-индикатор `-P`, отдельная команда `rclone check` для пост-валидации, и возможность принудительно использовать `--checksum`. ([Rclone][15])

Но под твою задачу (локальные HDD, macOS+Linux) **rsync остаётся более классическим и прозрачным выбором**, а rclone можно держать в уме для сложных сценариев (облако, S3 и т.д.).

---

## 7. Мини-шпаргалка

### 7.1 Копировать каталог месяца с прогрессом

```bash
SRC=/Users/qbook/Photos/202511
DST=/Volumes/ARCH20TB/Photos/202511

rsync -aHAX --human-readable \
      --info=progress2,name0 --no-inc-recursive \
      --partial --inplace \
      --bwlimit=40m \
      "$SRC/" "$DST/"
```

### 7.2 Проверка копии (rsync)

```bash
rsync -aHAX --human-readable \
      --info=progress2,name0 --no-inc-recursive \
      --checksum --dry-run \
      "$SRC/" "$DST/"
```

### 7.3 Создать манифест SHA-256 для каталога

```bash
cd /Volumes/ARCH20TB/Photos
hashdeep -r -l -c sha256 202511 > 202511.sha256.manifest
```

### 7.4 Проверить каталог через год

```bash
cd /Volumes/ARCH20TB/Photos
hashdeep -a -k 202511.sha256.manifest -r -l 202511
```

### 7.5 Копировать «партиями» с паузами

```bash
SRC=/Users/qbook/Photos/202511
DST=/Volumes/ARCH20TB/Photos/202511

# Копировать сессиями по 60 минут, потом rsync сам завершится.
rsync -aHAX --human-readable \
      --info=progress2,name0 --no-inc-recursive \
      --partial --inplace \
      --stop-after=60 \
      "$SRC/" "$DST/"
# подождать пока диск остынет и повторить ту же команду
```

---

Если хочешь, в следующем шаге могу собрать для тебя буквально один-два bash-скрипта вида `backup_month.sh 202511`, которые автоматизируют всё: копирование, проверку, генерацию манифеста и логирование, под твой конкретный layout дисков.

[1]: https://www.man7.org/linux/man-pages/man1/rsync.1.html?utm_source=chatgpt.com "rsync(1) - Linux manual page"
[2]: https://forensics.wiki/md5deep/?utm_source=chatgpt.com "Md5deep -"
[3]: https://github.com/BLAKE3-team/BLAKE3?utm_source=chatgpt.com "the official Rust and C implementations of the BLAKE3 ..."
[4]: https://www.dribin.org/dave/blog/archives/2024/01/21/rsync-overall-progress/?utm_source=chatgpt.com "Overall Progress with rsync | Dave Dribin's Blog"
[5]: https://ittavern.com/getting-started-with-rsync-comprehensive-guide/?utm_source=chatgpt.com "Getting started with rsync - Comprehensive Guide - Ittavern.com"
[6]: https://askubuntu.com/questions/609303/how-can-i-view-a-progress-bar-when-running-rsync?utm_source=chatgpt.com "How can I view a progress bar when running rsync?"
[7]: https://serverfault.com/questions/219013/showing-total-progress-in-rsync-is-it-possible?utm_source=chatgpt.com "Showing total progress in rsync: is it possible?"
[8]: https://unix.stackexchange.com/questions/333144/how-can-i-slow-down-rsync?utm_source=chatgpt.com "How can I slow down rsync?"
[9]: https://download.samba.org/pub/rsync/rsync.1?utm_source=chatgpt.com "rsync(1) manpage"
[10]: https://wiki.archlinux.org/title/Rsync?utm_source=chatgpt.com "rsync - ArchWiki"
[11]: https://www.tecmint.com/hashdeep-file-integrity-checker/?utm_source=chatgpt.com "Hashdeep: A Tool for File Integrity and Forensics"
[12]: https://aloiskraus.wordpress.com/2020/09/05/when-things-get-really-bad-ntfs-file-system-corruption-war-story/?utm_source=chatgpt.com "When Things Get Really Bad – NTFS File System Corruption ..."
[13]: https://blog.csdn.net/weixin_42514750/article/details/151894876?utm_source=chatgpt.com "高效数据完整性验证工具chechsum实战指南原创"
[14]: https://git-annex.branchable.com/tips/hashdeep_integration/?utm_source=chatgpt.com "hashdeep integration - git-annex - Branchable"
[15]: https://rclone.org/commands/rclone_copy/?utm_source=chatgpt.com "rclone copy"
