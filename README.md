# BashTimedFlashOps_v2

## EN

### Description
Version 2 of the Bash script. Improved logic and added new features.
The script checks the percentage of free space and starts deleting files if it falls bellow a set threshold. Deletion is based on filename pattern, starting with the oldest files, 10 at a time. Both scripts are designed to be scheduled via `cron`.

[**Ссылка на первую версию Bash-скрипта**](https://github.com/DShJr/BashTimedFlashOps)

**The repository contains two main scripts:**

* One script: `generation_files.sh` - **creates Bash files** on the flash drive at a specified time using the **`cron` scheduler**.
* The other script: `smart_cleanup_files.sh` - **deletes these files**, also on a `cron` schedule, eliminating the need for manual cleanup.

This is a simplified version of a homework assignment, showcasing basic skills in working with Bash scripts and `cron` for automating routine tasks.

### Assignment Execution Process

  This section details the setup and scripts used for the assignment.

  **We will use:**
  
1. **Flash Drive Path:**
 
   `/media/user/SMARTBUY`
   
   `SMARTBUY` - is the name of my flash drive.
   
2. **Files Folder:**
  `test_files_generation`

   This folder will be created on the flash drive.
   
   The script will create files for deletion in this folder, and another script will delete them. All operations will be scheduled using `cron`.
   
3. **File Name Prefix:**
   `dummy_file_`
   
   This will be the naming pattern for the files.

#### Шаг 1

**Скрипт Bash #1.**

**Назначение:** Только `создание` файлов.

**Имя скрипта:** `generation_files.sh`

**Описание скрипта:** Он будет просто создавать определённое количество файлов (в нашем случае, 200), каждый размером в 10МБ.

**Код скрипта:**

```Bash
#!/bin/bash

# --- SETTINGS ---

FLASH_DRIVE_MOUNT_POINT="/media/dima/SMARTBUY"
FILES_FOLDER="test_files_generation"
FULL_PATH_TO_FOLDER="$FLASH_DRIVE_MOUNT_POINT/$FILES_FOLDER"

FILES_SIZE_MB=10
NUMBER_OF_FILES=200
FILENAME_PREFIX="dummy_file_"

# ---/SETTINGS ---

# Проверка, на монтирование флэшки
if ! mountpoint -q "$FLASH_DRIVE_MOUNT_POINT"; then
    echo "ERROR: Флэшка '$FLASH_DRIVE_MOUNT_POINT' не примантирована или не доступна. Проверь, подключена ли флэшка."
    exit 1
fi

# Проверим, Существует ли папка, и создадим, если её нет
if [ ! -d "$FULL_PATH_TO_FOLDER" ]; then
    mkdir -p "$FULL_PATH_TO_FOLDER"
    echo "Папка '$FULL_PATH_TO_FOLDER' создана."
fi

# Генерация файлов
for ((i=1; i<="$NUMBER_OF_FILES"; i++)); do
    # Форматирование имени файла с нулями
    FILENAME=$(printf "%s%03d.bin" "$FILENAME_PREFIX" "$i")
    FILEPATH="$FULL_PATH_TO_FOLDER/$FILENAME"

    # Создадим файлы с заданным Размером в Мегобайтах
    dd if=/dev/zero of="$FILEPATH" bs=1M count="$FILES_SIZE_MB" status=none
    echo "Создан файл: $FILENAME"
done
echo "Все файлы созданы."
```

**Скрипт Bash #2.**

**Назначение:** Для `удаления` файлов.

**Имя скрипта:** `smart_cleanup_files.sh`

**Описание скрипта:** Он будет удалять файлы с префиксом `dummy_file_` по 10 штук за одну итерацию, пока процент свободного места не достигнет `95%` или выше.

**Код скрипта:**

```Bash
#!/bin/bash

# --- SETTINGS ---

FLASH_DRIVE_MOUNT_POINT="/media/dima/SMARTBUY"
FILES_FOLDER="test_files_generation"
FULL_PATH_TO_FOLDER="$FLASH_DRIVE_MOUNT_POINT/$FILES_FOLDER"

# Процент свободного места, при достижении которого скрипт Прекратит удаление (то есть если места меньше 20%, удаляем, пока не станет >=20%).
# Однако, в рамках Теста, процент выставим не 20, а 95.
MIN_FREE_SPACE_PERCENT=95

# Шаблон имени файлов для удаления (должен соответствовать файлам, созданным generation_files.sh).
FILENAME_PATTERN="dummy_file_*.bin"

# Колическтво файлов, удалемых за одну итерацию (для поэтапного удаления)
FILES_TO_DELETE_PER_ITERATION=10

# Путь для лог-файла. Будем использовать в Функции логирования.
LOG_FILE="/tmp/log/smart_cleanup.log"

# --- /SETTINGS ---

# --- LOGGING FUNCTION ---
log_message () {
    echo "$(date '+%Y-%m-%d %H-%M-%S') - $1" | tee -a "$LOG_FILE"
}

# --- MAIN SCRIPT LOGIC ---
log_message "INFO: Script started. Checking flash drive: $FLASH_DRIVE_MOUNT_POINT"

# 1. Проверка, что флэшка Примонтирована  и Доступна.
if ! mountpoint -q "$FLASH_DRIVE_MOUNT_POINT"; then
    log_message "ERROR: Flash drive '$FLASH_DRIVE_MOUNT_POINT' is not mounted or not accessible. Exiting."
    exit 1
fi

# 2. Проверка, Существует ли целевая папка
if [ ! -d "$FULL_PATH_TO_FOLDER" ]; then
    log_message "WARNING: Target folder '$FULL_PATH_TO_FOLDER' not found. Nothing to cleanup. Exiting."
    exit 0
fi

while true; do
    
    # 3. Получение информации о Свободом месте
    DF_OUTPUT=$(df -P "$FLASH_DRIVE_MOUNT_POINT" 2>/dev/null | awk 'NR==2 {print $2, $4}') 
    if [ -z "$FLASH_DRIVE_MOUNT_POINT" ]; then
        log_message "ERROR: Could not get disk space info for '$FLASH_DRIVE_MOUNT_POINT'. Check path or permissions. Exiting."
        exit 1
    fi
    
    TOTAL_BLOCKS=$(echo "$DF_OUTPUT" | awk '{print $1}')
    AVAILABLE_BLOCKS=$(echo "$DF_OUTPUT" | awk '{print $2}')

    # Прежде чем приступать к вычислению свободного места на флэшке - процент свободного места, - Сделаем проверку Отсутствия деления на ноль.
    if [ "$TOTAL_BLOCKS" -eq 0 ]; then
        log_message "ERROR: Total blocks on '$FLASH_DRIVE_MOUNT_POINT' is 0. Cannot calculate free space. Exiting."
        exit 1
    fi

    FREE_SPACE_PERCENT=$(( AVAILABLE_BLOCKS * 100 / TOTAL_BLOCKS ))
    log_message "INFO: Current free space on '$FLASH_DRIVE_MOUNT_POINT': $FREE_SPACE_PERCENT% (Target: >= $MIN_FREE_SPACE_PERCENT%.)"

    # 4. Проверка Условия на Свободное место
    if [ "$FREE_SPACE_PERCENT" -ge "$MIN_FREE_SPACE_PERCENT" ]; then
        log_message "INFO: Free space ($FREE_SPACE_PERCENT%) is sufficient. No cleanup needed. Exiting."
        break
    fi

    # 5. Если места не хватает, то Ищем и Удаляем файлы
    log_message "WARNING: Free space is low ($FREE_SPACE_PERCENT%). Starting cleanup."

    # Найдём Самые Старые файлы с заданным шаблоном
    FILES_TO_REMOVE=$(find "$FULL_PATH_TO_FOLDER" -maxdepth 1 -type f -name "$FILENAME_PATTERN" -printf '%T@ %p\n' 2>/dev/null | sort -n | head>
    if [ -z "$FILES_TO_REMOVE" ]; then
        log_message "INFO: No more files matching '$FILENAME_PATTERN' found in '$FULL_PATH_TO_FOLDER' to delete. Free space is still $FREE_SPAC>
        exit 0
    fi

    log_message "INFO: Deleting the following files:"
    echo "$FILES_TO_REMOVE" | while IFS= read -r file; do
        log_message " - $file"
        rm -f "$file" || log_message "ERROR: Failed to delete $file."
    done

    # Дадим системе немного времени обновить статистику диска.
    sleep 2

done

log_message "INFO: Script finished."
```


#### Шаг 2
**Назначим права обоим скриптам.**

```Bash
chmod +x `generation_files.sh`
```

```Bash
chmod +x `smart_cleanup_files.sh`
```


#### Шаг 3
**Тестирование.**

**1)** **Запускаем скрипт генерации файлов:** `generation_files.sh`

```Bash
./generation_files.sh
```

Он создаёт `200` файлов, которые займут 2ГБ на флэшке.

**2)** **Убедимся, что свободное место `ниже` `95%`.**
// В домашнем задании была задана величина в 20%. `Онако`, для упрощения процесса, я `увеличил` `до 95%`.

Запускаем команду:

```Bash
df -h /media/user/SMARTBUY
```

Так же, `для сравнения`, запускал эту же команду в другом формате в 1К-блоках.

```Bash
df -P /media/user/SMARTBUY
```

Убедимся, что процент занятого места `больше` `5%`.

**3)** **Запускаем скрипт очистки:** `smart_cleanup_files.sh`

```Bash
./smart_cleanup_files.sh
```

Этот **скрипт должен определить**, что **места недостаточно, и начать** удалять файлы по 10 штук за одну итерацию, пока свободное место не достигнет 95% или выше.

**4)** **Проверим содержимое лог-фйла**

```Bash
cat /tmp/log/smart_cleanup.log
```


#### Шаг 4
**Добавим задания через `crontab`, по загрузке этих скриптов по-очереди.**

Откроем редактор заданий `cron`.

```Bash
crontab -e
```

Добавим задачи в `crontab`:
`generation_files.sh`: должен запускаться каждые 10 минут каждого часа.
`smart_cleanup_files.sh`: должен запускаться каждые 18 минут каждого часа.

```Bash
10 * * * * /home/user/..../`generation_files.sh` >> /tmp/log/cron_smart_cleanup.log 2>&1
18 * * * * /home/user/..../`smart_cleanup_files.sh` >> /tmp/log/cron_smart_cleanup.log 2>&1
```

### Результат выполнения.
`Изображения` результатов выполнения скриптов `прилагаю`.


---
## RU

### Описание
**Автоматическая очистка дискового пространства**
Версия 2 Bash-скрипта. Улучшенная логика и добавлены новые функции.
Скрипт, проверяет процент свободного места и начинаетудалять файлы, если он опускается ниже заданного порога.
Удаление производится по шаблону имени, начиная с самых старых файлов, по 10 штук за раз. Оба скрипта предназначены для запуска по расписанию через `cron`.

[**Ссылка на первую версию Bash-скрипта**](https://github.com/DShJr/BashTimedFlashOps)

**В репозитории представлены два основных скрипта:**

* Один скрипт: `generation_files.sh` - **создаёт** Bash-файлы на флешке в заданное время с помощью планировщика `cron`.
* Другой скрипт: `smart_cleanup_files.sh` - **удаляет** эти файлы, также по расписанию `cron`, что избавляет от необходимости ручной очистки.

### Процесс выполнение задания

**Будем использовать:**
  
1. **Путь к флэшке:**
   `/media/user/SMARTBUY`
   
   `SMARTBUY` - имя моей флэшки.
   
2. **Папка для файлов:**
  `test_files_generation`

   Эта папка будет создана на этой флэшке.
   
   В ней скрипт будет создавать файлы для удаления. А другой скрипт будет их удалять. И всё по расписанию в `cron`.

3. **Префикс имени файла:**
   
   `dummy_file_`    
   Шаблон названия имён файлов.


#### Шаг 1

**Скрипт Bash #1.**

**Назначение:** Только `создание` файлов.

**Имя скрипта:** `generation_files.sh`

**Описание скрипта:** Он будет просто создавать определённое количество файлов (в нашем случае, 200), каждый размером в 10МБ.

**Код скрипта:**

```Bash
#!/bin/bash

# --- SETTINGS ---

FLASH_DRIVE_MOUNT_POINT="/media/dima/SMARTBUY"
FILES_FOLDER="test_files_generation"
FULL_PATH_TO_FOLDER="$FLASH_DRIVE_MOUNT_POINT/$FILES_FOLDER"

FILES_SIZE_MB=10
NUMBER_OF_FILES=200
FILENAME_PREFIX="dummy_file_"

# ---/SETTINGS ---

# Проверка, на монтирование флэшки
if ! mountpoint -q "$FLASH_DRIVE_MOUNT_POINT"; then
    echo "ERROR: Флэшка '$FLASH_DRIVE_MOUNT_POINT' не примантирована или не доступна. Проверь, подключена ли флэшка."
    exit 1
fi

# Проверим, Существует ли папка, и создадим, если её нет
if [ ! -d "$FULL_PATH_TO_FOLDER" ]; then
    mkdir -p "$FULL_PATH_TO_FOLDER"
    echo "Папка '$FULL_PATH_TO_FOLDER' создана."
fi

# Генерация файлов
for ((i=1; i<="$NUMBER_OF_FILES"; i++)); do
    # Форматирование имени файла с нулями
    FILENAME=$(printf "%s%03d.bin" "$FILENAME_PREFIX" "$i")
    FILEPATH="$FULL_PATH_TO_FOLDER/$FILENAME"

    # Создадим файлы с заданным Размером в Мегобайтах
    dd if=/dev/zero of="$FILEPATH" bs=1M count="$FILES_SIZE_MB" status=none
    echo "Создан файл: $FILENAME"
done
echo "Все файлы созданы."
```

**Скрипт Bash #2.**

**Назначение:** Для `удаления` файлов.

**Имя скрипта:** `smart_cleanup_files.sh`

**Описание скрипта:** Он будет удалять файлы с префиксом `dummy_file_` по 10 штук за одну итерацию, пока процент свободного места не достигнет `95%` или выше.

**Код скрипта:**

```Bash
#!/bin/bash

# --- SETTINGS ---

FLASH_DRIVE_MOUNT_POINT="/media/dima/SMARTBUY"
FILES_FOLDER="test_files_generation"
FULL_PATH_TO_FOLDER="$FLASH_DRIVE_MOUNT_POINT/$FILES_FOLDER"

# Процент свободного места, при достижении которого скрипт Прекратит удаление (то есть если места меньше 20%, удаляем, пока не станет >=20%).
# Однако, в рамках Теста, процент выставим не 20, а 95.
MIN_FREE_SPACE_PERCENT=95

# Шаблон имени файлов для удаления (должен соответствовать файлам, созданным generation_files.sh).
FILENAME_PATTERN="dummy_file_*.bin"

# Колическтво файлов, удалемых за одну итерацию (для поэтапного удаления)
FILES_TO_DELETE_PER_ITERATION=10

# Путь для лог-файла. Будем использовать в Функции логирования.
LOG_FILE="/tmp/log/smart_cleanup.log"

# --- /SETTINGS ---

# --- LOGGING FUNCTION ---
log_message () {
    echo "$(date '+%Y-%m-%d %H-%M-%S') - $1" | tee -a "$LOG_FILE"
}

# --- MAIN SCRIPT LOGIC ---
log_message "INFO: Script started. Checking flash drive: $FLASH_DRIVE_MOUNT_POINT"

# 1. Проверка, что флэшка Примонтирована  и Доступна.
if ! mountpoint -q "$FLASH_DRIVE_MOUNT_POINT"; then
    log_message "ERROR: Flash drive '$FLASH_DRIVE_MOUNT_POINT' is not mounted or not accessible. Exiting."
    exit 1
fi

# 2. Проверка, Существует ли целевая папка
if [ ! -d "$FULL_PATH_TO_FOLDER" ]; then
    log_message "WARNING: Target folder '$FULL_PATH_TO_FOLDER' not found. Nothing to cleanup. Exiting."
    exit 0
fi

while true; do
    
    # 3. Получение информации о Свободом месте
    DF_OUTPUT=$(df -P "$FLASH_DRIVE_MOUNT_POINT" 2>/dev/null | awk 'NR==2 {print $2, $4}') 
    if [ -z "$FLASH_DRIVE_MOUNT_POINT" ]; then
        log_message "ERROR: Could not get disk space info for '$FLASH_DRIVE_MOUNT_POINT'. Check path or permissions. Exiting."
        exit 1
    fi
    
    TOTAL_BLOCKS=$(echo "$DF_OUTPUT" | awk '{print $1}')
    AVAILABLE_BLOCKS=$(echo "$DF_OUTPUT" | awk '{print $2}')

    # Прежде чем приступать к вычислению свободного места на флэшке - процент свободного места, - Сделаем проверку Отсутствия деления на ноль.
    if [ "$TOTAL_BLOCKS" -eq 0 ]; then
        log_message "ERROR: Total blocks on '$FLASH_DRIVE_MOUNT_POINT' is 0. Cannot calculate free space. Exiting."
        exit 1
    fi

    FREE_SPACE_PERCENT=$(( AVAILABLE_BLOCKS * 100 / TOTAL_BLOCKS ))
    log_message "INFO: Current free space on '$FLASH_DRIVE_MOUNT_POINT': $FREE_SPACE_PERCENT% (Target: >= $MIN_FREE_SPACE_PERCENT%.)"

    # 4. Проверка Условия на Свободное место
    if [ "$FREE_SPACE_PERCENT" -ge "$MIN_FREE_SPACE_PERCENT" ]; then
        log_message "INFO: Free space ($FREE_SPACE_PERCENT%) is sufficient. No cleanup needed. Exiting."
        break
    fi

    # 5. Если места не хватает, то Ищем и Удаляем файлы
    log_message "WARNING: Free space is low ($FREE_SPACE_PERCENT%). Starting cleanup."

    # Найдём Самые Старые файлы с заданным шаблоном
    FILES_TO_REMOVE=$(find "$FULL_PATH_TO_FOLDER" -maxdepth 1 -type f -name "$FILENAME_PATTERN" -printf '%T@ %p\n' 2>/dev/null | sort -n | head>
    if [ -z "$FILES_TO_REMOVE" ]; then
        log_message "INFO: No more files matching '$FILENAME_PATTERN' found in '$FULL_PATH_TO_FOLDER' to delete. Free space is still $FREE_SPAC>
        exit 0
    fi

    log_message "INFO: Deleting the following files:"
    echo "$FILES_TO_REMOVE" | while IFS= read -r file; do
        log_message " - $file"
        rm -f "$file" || log_message "ERROR: Failed to delete $file."
    done

    # Дадим системе немного времени обновить статистику диска.
    sleep 2

done

log_message "INFO: Script finished."
```


#### Шаг 2
**Назначим права обоим скриптам.**

```Bash
chmod +x `generation_files.sh`
```

```Bash
chmod +x `smart_cleanup_files.sh`
```


#### Шаг 3
**Тестирование.**

**1)** **Запускаем скрипт генерации файлов:** `generation_files.sh`

```Bash
./generation_files.sh
```

Он создаёт `200` файлов, которые займут 2ГБ на флэшке.

**2)** **Убедимся, что свободное место `ниже` `95%`.**
// В домашнем задании была задана величина в 20%. `Онако`, для упрощения процесса, я `увеличил` `до 95%`.

Запускаем команду:

```Bash
df -h /media/user/SMARTBUY
```

Так же, `для сравнения`, запускал эту же команду в другом формате в 1К-блоках.

```Bash
df -P /media/user/SMARTBUY
```

Убедимся, что процент занятого места `больше` `5%`.

**3)** **Запускаем скрипт очистки:** `smart_cleanup_files.sh`

```Bash
./smart_cleanup_files.sh
```

Этот **скрипт должен определить**, что **места недостаточно, и начать** удалять файлы по 10 штук за одну итерацию, пока свободное место не достигнет 95% или выше.

**4)** **Проверим содержимое лог-фйла**

```Bash
cat /tmp/log/smart_cleanup.log
```


#### Шаг 4
**Добавим задания через `crontab`, по загрузке этих скриптов по-очереди.**

Откроем редактор заданий `cron`.

```Bash
crontab -e
```

Добавим задачи в `crontab`:
`generation_files.sh`: должен запускаться каждые 10 минут каждого часа.
`smart_cleanup_files.sh`: должен запускаться каждые 18 минут каждого часа.

```Bash
10 * * * * /home/user/..../`generation_files.sh` >> /tmp/log/cron_smart_cleanup.log 2>&1
18 * * * * /home/user/..../`smart_cleanup_files.sh` >> /tmp/log/cron_smart_cleanup.log 2>&1
```

### Результат выполнения.
`Изображения` результатов выполнения скриптов `прилагаю`.
