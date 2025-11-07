Ниже — готовая «операционная карта» на весь цикл работы с внешним ZFS-диском на Intel-Mac (у тебя 2.6 GHz 6-Core i7, macOS 10.15+ и OpenZFS 2.3.0).

Я исхожу из того, что OpenZFS установлен через Homebrew/cask (`brew install --cask openzfs`), он ставит kext в `/Library/Extensions` и требует разрешения в *Privacy & Security*. ([Homebrew Formulae][1])

---

## 0. Быстрый чек-лист «подключил → поработал → выключил»

**Подключение (всё хорошо и по-взрослому)**

```bash
diskutil list                        # 1. Убедиться, что диск виден
kextstat | grep -i lundman          # 2. ZFS-модуль загружен?
sudo kextload /Library/Extensions/zfs.kext   # 2a. Загрузить, если нужно (Intel / 10.15)
# или на новых системах:
# sudo kmutil load -p /Library/Extensions/zfs.kext

sudo zpool import                    # 3. Посмотреть доступные пулы
sudo zpool import -R /Volumes/zfsdisk <POOL> # 4. Импорт с altroot
sudo zfs mount -a                    # 5. Смонтировать все ФС
zpool status                         # 6. Проверить состояние пула
zfs list                             # 7. Проверить список ФС
df -h | grep /Volumes/zfsdisk        # 8. Убедиться, что точки монтирования есть
```

**Отключение (чистый выход и безопасное питание)**

```bash
zpool status -x                      # 1. Убедиться, что ошибок нет
sudo lsof /Volumes/zfsdisk          # 2. Кто держит файлы? (должно быть пусто)
sudo zpool sync <POOL>              # 3. Сбросить кеш на диск
sync                                 # 3a. Доп. страховка

sudo zpool export <POOL>            # 4. Экспорт пула
# при необходимости:
# sudo zpool export -f <POOL>

diskutil eject /dev/diskX           # 5. «Выбросить» устройство
diskutil list                        # 6. Проверить, что diskX исчез

# после этого — отключать питание дока / выдёргивать USB
```

Дальше — то же самое, но с пояснениями и «полезными» командами вокруг.

---

## 1. Первичная диагностика после подключения диска

### 1.1. Проверяем, видит ли macOS физический диск

```bash
diskutil list
```

В выводе ищешь устройство на ~22 TB (`/dev/diskX` с размером 22 TB). Если его **нет**, проблема не в ZFS, а в железе/доке/кабеле. Apple-доки и статьи по `diskutil` как раз показывают, что даже неизвестная ФС всё равно должна появляться как устройство. ([OS X Daily][2])

Если Mac видит диск, но ёмкость не 22 TB, а например 2.2 TB — косяк USB-SATA моста (старый 32-битный LBA). Такое регулярно всплывает в обсуждениях внешних больших дисков. ([The FreeBSD Forums][3])

### 1.2. Проверяем, загружены ли ZFS-модули

```bash
kextstat | grep -i lundman
```

Ищем строки вроде `net.lundman.zfs` и `net.lundman.spl` (это OpenZFS on OS X). ([openzfsonosx.org][4])

Если пусто — модуль не загружен.

#### Загрузка kext (Intel-Mac)

*Для Catalina и в целом Intel-систем с kext-расширениями:*

```bash
sudo kextload /Library/Extensions/zfs.kext
# при необходимости:
sudo kextload /Library/Extensions/spl.kext
```

Такую схему загрузки kext (через `kextload` и/или скрипт `load.sh`) описывают в документации OpenZFS On OS X. ([openzfsonosx.org][4])

*Для Big Sur и новее* обычно используют:

```bash
sudo kmutil load -p /Library/Extensions/zfs.kext
```

`kmutil` — новый рекомендуемый инструмент загрузки kext, о чём пишут и разработчики OpenZFS, и Apple. ([GitHub][5])

Если загрузка не удаётся, часто требуется разрешить kext в **System Settings → Privacy & Security** (для openzfs это явно указано и в cask-формуле Homebrew). ([Homebrew Formulae][1])

---

## 2. Импорт пула и монтирование файловых систем

### 2.1. Находим пул

После того как kext загружены:

```bash
sudo zpool import
```

Команда покажет список найденных пулов (имя, размер, состояние). Такой подход описан и в официальной wiki по `zpool` для O3X. ([openzfsonosx.org][6])

Если пул не найден:

```bash
sudo zpool import -d /dev
```

`-d` позволяет явно указать каталог/устройство, где искать vdev’ы. ([Super User][7])

### 2.2. Импорт пула с указанием корня монтирования

Допустим, пул называется `tank`, а ты хочешь, чтобы всё монтировалось под `/Volumes/tank`:

```bash
sudo mkdir -p /Volumes/tank
sudo zpool import -R /Volumes/tank tank
# при необходимости, если пул «с другого хоста»:
# sudo zpool import -f -R /Volumes/tank tank
```

Флаг `-R` задаёт altroot — префикс ко всем `mountpoint` пула. ([justinscholz.de][8])

Флаг `-f` нужен, если ZFS считает, что пул всё ещё активен на другой машине (другой hostid). Это обычная практика при переносе пулов между системами. ([The FreeBSD Forums][9])

### 2.3. Монтируем все файловые системы пула

Обычно при импорте ZFS сам монтирует все datasets с `mountpoint` ≠ `legacy`. Если какие-то не смонтировались:

```bash
sudo zfs mount -a        # смонтировать все
# или выборочно
sudo zfs mount tank/media
```

`zfs mount -a` — стандартный «поднять всё, что должно быть смонтировано» и широко используется в гайдах. ([Apple Support Community][10])

---

## 3. Проверка, что всё ок

Полезный мини-набор:

```bash
zpool status
zpool status -x           # «-x» показывает только проблемные пулы
zpool list

zfs list                  # список всех dataset’ов
df -h | egrep 'tank|zfs'  # убедиться в точках монтирования
```

* `zpool status` покажет состояние диска, ошибки I/O, DEGRADED и т.п. ([openzfsonosx.org][6])
* `zpool status -x` вернёт «all pools are healthy», если всё в порядке. ([openzfsonosx.org][11])

Если делал большие записи, можно раз в какое-то время запускать:

```bash
sudo zpool scrub tank
```

`zpool scrub` — фоновая проверка целостности; её рекомендуют для периодической проверки пула, особенно на внешних дисках. ([Practical ZFS][12])

---

## 4. Подготовка к отключению: проверка и «очистка»

### 4.1. Проверяем состояние и открытые файлы

```bash
zpool status -x
```

Если всё хорошо — «no pools available to import» или «all pools are healthy» / аналогичное сообщение без ошибок. ([Unix & Linux Stack Exchange][13])

Затем убеждаемся, что никто не держит файлы на пуле:

```bash
sudo lsof /Volumes/tank
# или по конкретному dataset:
sudo lsof /Volumes/tank/media
```

`lsof` по пути к тому же советуют и в Apple-дискуссиях / блогах, когда нужно понять, кто мешает извлечению внешнего диска. ([Reddit][14])

Если там висят `mds`/Spotlight или Finder — либо ждём, либо аккуратно закрываем / отключаем индексацию именно для этого диска (по желанию).

### 4.2. Сбрасываем кеш на диск

```bash
sudo zpool sync tank
sync
```

`zpool sync` принудительно сбрасывает все «грязные» данные из кеша ARC/ZIL на диск, это рекомендуемый способ убедиться, что все операции завершены перед экспортом. ([ryan.himmelwright.net][15])

Обычный `sync` — общесистемная страховка.

---

## 5. Размонтирование и экспорт пула

### 5.1. Опционально: размонтировать ФС ZFS

Это не обязательно (экспорт сам по себе размонтирует), но иногда удобно:

```bash
sudo zfs unmount -a           # размонтировать все ZFS ФС
# или точечно:
sudo zfs unmount /Volumes/tank/media
```

Если ФС занята, можно добавить `-f`, но лучше сначала разобраться, кто её держит через `lsof`. ([Server Fault][16])

### 5.2. Экспорт пула

```bash
sudo zpool export tank
```

Это **главный ZFS-шаг** перед физическим отключением: экспорт помечает пул как «аккуратно закрытый» и готовый к переносу или отключению. После `zpool export` ZFS больше не использует этот диск, данные считаются безопасно записанными. ([Reddit][17])

Если видишь сообщение `cannot export 'tank': pool is busy` — кто-то ещё открывает файлы:

```bash
sudo lsof /Volumes/tank
# закрыть процессы → повторить export
# крайний вариант:
sudo zpool export -f tank
```

`-f` форсирует экспорт пула, хотя обычно это не рекомендуется без понимания, какие процессы оторвёшь. ([Server Fault][18])

---

## 6. Финальный шаг: безопасное отключение устройства

ZFS-сторона закончилась, теперь вырубить железо по-маковски.

### 6.1. «Выбросить» диск на уровне macOS

Сначала определяем нужное устройство:

```bash
diskutil list
# допустим это /dev/disk3
```

Затем:

```bash
diskutil eject /dev/disk3
```

`diskutil eject` не просто размонтирует тома, но и «отцепит» устройство так, что его `/dev/disk3` исчезнет — это рекомендуют именно для USB-дисков. ([Ask Different][19])

Если по какой-то причине хочется сначала размонтировать все разделы диска:

```bash
diskutil unmountDisk /dev/disk3
diskutil eject /dev/disk3
```

В случаях, когда обычный `eject` не удаётся, на форумах Apple советуют использовать `sudo diskutil umount force /dev/disk3s1` (и аналоги для других разделов). ([Apple Support Community][10])

### 6.2. Контрольный просмотр и питание

```bash
diskutil list   # убеждаемся что /dev/disk3 больше нет
```

Когда устройство исчезло, можно:

* дождаться, пока индикатор на доке перестанет мигать,
* отключить питание дока **или** вытащить USB-кабель.

С точки зрения ZFS, после `zpool export` данные уже безопасны; это подтверждают и обсуждения на Unix.SE и Reddit — ОС может ещё что-то логировать, но ZFS-пул уже закрыт. ([Unix & Linux Stack Exchange][13])

---

## 7. Что ещё полезно не забыть

### 7.1. Регулярный scrub

Раз в N недель:

```bash
sudo zpool scrub tank
zpool status tank   # после завершения: убедиться, что ошибок нет
```

Так делают многие, кто возит ZFS-пулы на внешних дисках; это позволяет вовремя обнаружить битые сектора, усиливая стандартную защиту ZFS. ([Practical ZFS][12])

### 7.2. Просмотр истории операций

Иногда удобно понять, что вообще делали с пулом:

```bash
zpool history tank
```

Показывает историю `zpool`/`zfs`-команд, полезно при отладке или если что-то пошло не так. ([openzfsonosx.org][6])

### 7.3. Если диск «выдернули» без экспорта

В мире O3X/FreeBSD/TrueNAS при внезапном отключении USB-ZFS-диска зачастую советуют **перезагрузку**, особенно если `zpool status` показывает `pool I/O is currently suspended` или похожие ошибки. ([GitHub][20])

Минимальный набор действий в таком случае:

```bash
zpool status tank          # понять, что случилось
zpool clear tank           # попытаться сбросить ошибки
# если диск всё ещё отсутствует / всё плохо:
# 1) выключить Mac
# 2) подключить диск
# 3) включить Mac → zpool import
```

---

## 8. Итоговая «шпаргалка» в одну кучу

**Подключение:**

```bash
diskutil list
kextstat | grep -i lundman
sudo kextload /Library/Extensions/zfs.kext    # при необходимости

sudo zpool import
sudo zpool import -R /Volumes/tank tank
sudo zfs mount -a

zpool status
zfs list
df -h | grep /Volumes/tank
```

**Отключение:**

```bash
zpool status -x
sudo lsof /Volumes/tank
sudo zpool sync tank
sync

sudo zpool export tank
diskutil eject /dev/disk3
diskutil list
```

Если пройдёшь все шаги сверху вниз и снизу вверх — цепочка «подключил → поработал → отключил» будет максимально безопасной и для данных, и для железа.

[1]: https://formulae.brew.sh/cask/openzfs?utm_source=chatgpt.com "openzfs — Homebrew Formulae"
[2]: https://osxdaily.com/2013/05/13/mount-unmount-drives-from-the-command-line-in-mac-os-x/?utm_source=chatgpt.com "Mount & Unmount Drives from the Command Line in Mac ..."
[3]: https://forums.freebsd.org/threads/setup-zfs-on-occasionally-detached-external-drive.94444/?utm_source=chatgpt.com "Setup ZFS on occasionally detached external drive?"
[4]: https://openzfsonosx.org/wiki/Install?utm_source=chatgpt.com "Install"
[5]: https://github.com/openzfsonosx/openzfs/issues/8?utm_source=chatgpt.com "kext does not load on 10.16 Big Sur · Issue #8"
[6]: https://openzfsonosx.org/wiki/Zpool?utm_source=chatgpt.com "Zpool"
[7]: https://superuser.com/questions/1359140/beginner-issues-with-zfs-on-os-x-how-to-re-mount?utm_source=chatgpt.com "macos - beginner issues with ZFS on OS X: how to re-mount?"
[8]: https://justinscholz.de/2018/06/15/an-extensive-zfs-setup-on-macos/?utm_source=chatgpt.com "An extensive ZFS setup on MacOS - Yep"
[9]: https://forums.freebsd.org/threads/correctly-using-external-zfs-hdd-between-multiple-pc-s.94639/?utm_source=chatgpt.com "correctly using external zfs hdd between multiple pc`s"
[10]: https://discussions.apple.com/thread/251284296?utm_source=chatgpt.com "Unable to Eject Disk in Terminal"
[11]: https://openzfsonosx.org/forum/viewtopic.php?f=26&t=3319&utm_source=chatgpt.com "View topic - Install error Catalina 10.15.1"
[12]: https://discourse.practicalzfs.com/t/shipping-my-nas-overseas-whats-the-best-way-to-maintain-access-to-my-data-in-the-meantime/713?utm_source=chatgpt.com "Shipping my NAS overseas, what's the best way to ..."
[13]: https://unix.stackexchange.com/questions/624612/what-exactly-does-zpool-export-do?utm_source=chatgpt.com "What exactly does zpool export do?"
[14]: https://www.reddit.com/r/MacOS/comments/14qhrfy/why_cant_macos_just_tell_me_which_program_is/?utm_source=chatgpt.com "Why can't MacOS just TELL ME which program is using the ..."
[15]: https://ryan.himmelwright.net/post/zfS-Backups-To-LUKS-External/?utm_source=chatgpt.com "ZFS Snapshot Backups to an External Drive with LUKS"
[16]: https://serverfault.com/questions/159422/os-x-determine-which-application-is-accessing-a-hdd-and-preventing-ejection?utm_source=chatgpt.com "OS X determine which application is accessing a HDD and ..."
[17]: https://www.reddit.com/r/zfs/comments/5y3nn2/if_i_zpool_export_my_external_hard_drive_is_it/?utm_source=chatgpt.com "If I \"zpool export\" my external hard drive, is it now safe to ..."
[18]: https://serverfault.com/questions/954479/always-do-zpool-export-for-easier-and-or-more-reliable-recovery?utm_source=chatgpt.com "Always do \"zpool export\" for easier and/or more reliable ..."
[19]: https://apple.stackexchange.com/questions/370406/how-to-eject-all-drives-from-the-command-line?utm_source=chatgpt.com "How to eject all drives from the command-line - Ask Different"
[20]: https://github.com/openzfsonosx/zfs/issues/104?utm_source=chatgpt.com "zpool: pool I/O is currently suspended · Issue #104"
