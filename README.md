# Конфигурация и Руководство по установке Fluentd (fluent-package)

Этот репозиторий содержит базовую конфигурацию Fluentd (теперь известного как `fluent-package`), предназначенную для сбора логов через HTTP и записи их в файл, а также включает пример фильтрации для отбрасывания определенных сообщений.

---

## 1. Установка Fluentd (fluent-package) на Ubuntu 24.04 (Noble Numbat)

Обратите внимание, что `td-agent` был переименован в `fluent-package`. Следуйте этим шагам для установки последней LTS-версии `fluent-package` на вашу систему Ubuntu 24.04.

1.  **Удаление старых файлов GPG ключей и списка репозиториев (если они были созданы):**
    ```bash
    sudo rm -f /usr/share/keyrings/td-agent-archive-keyring.gpg
    sudo rm -f /etc/apt/sources.list.d/td-agent.list
    ```

2.  **Установка необходимых пакетов для добавления репозитория:**
    ```bash
    sudo apt update && sudo apt install -y lsb-release gnupg curl
    ```

3.  **Добавление GPG ключа Fluent Package:**
    ```bash
    curl -fsSL https://packages.fluent.io/fluent.gpg | gpg --dearmor | sudo tee /usr/share/keyrings/fluent-archive-keyring.gpg > /dev/null
    ```

4.  **Добавление репозитория Fluent Package для Ubuntu Noble (24.04):**
    ```bash
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/fluent-archive-keyring.gpg] https://packages.fluent.io/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/fluent.list
    ```

5.  **Обновление списка пакетов:**
    ```bash
    sudo apt update
    ```

6.  **Установка fluent-package:**
    ```bash
    sudo apt install -y fluent-package
    ```

---

## 2. Описание Конфигурационного Файла (`fluentd.conf`)

Конфигурационный файл Fluentd находится по пути `/etc/fluent/fluentd.conf`.
Пример конфигурации, представленный в этом репозитории:

```xml
<source>
  @type http
  port 8888
</source>

<filter test1>
  @type grep
  <exclude>
    key source
    pattern localhost
  </exclude>
</filter>

<match test1>
  @type file
  path /var/log/fluent/access
</match>
```

**Объяснение секций:**

*   **`<source> @type http`**:
    *   Тип: `source` (источник данных).
    *   Плагин: `http`.
    *   Назначение: Fluentd будет прослушивать входящие HTTP POST запросы на порту `8888`. Данные из этих запросов будут обрабатываться как логи.

*   **`<filter test1>`**:
    *   Тип: `filter` (фильтр данных).
    *   Плагин: `grep`.
    *   Назначение: Этот фильтр применяется ко всем событиям, помеченным тегом `test1` (который автоматически присваивается входящим HTTP-сообщениям в данном случае).
    *   **`<exclude>`**: Указывает, что события, соответствующие указанному правилу, должны быть **отброшены**.
    *   `key source`: Фильтр будет проверять поле с именем `source` в структуре JSON-сообщения.
    *   `pattern localhost`: Если значение поля `source` содержит подстроку "localhost", это сообщение будет отброшено и не попадет в следующие секции `match`.

*   **`<match test1>`**:
    *   Тип: `match` (вывод данных).
    *   Плагин: `file`.
    *   Назначение: Эта секция обрабатывает все события, помеченные тегом `test1`, которые **не были отброшены** предыдущим фильтром.
    *   `path /var/log/fluent/access`: Fluentd будет записывать полученные логи в файлы внутри директории `/var/log/fluent/access/`. Fluentd автоматически создает файлы с временными метками в этой директории (например, `access.20250616.log`).

---

## 3. Использование Конфигурации

После установки `fluent-package`, выполните следующие шаги для применения и тестирования конфигурации:

1.  **Переименование стандартного конфиг-файла (если он существует):**
    ```bash
    sudo mv /etc/fluent/fluentd.conf /etc/fluent/fluentd.conf.bak
    ```

2.  **Создание директории `/etc/fluent` (если она еще не существует):**
    ```bash
    sudo mkdir -p /etc/fluent
    ```

3.  **Создание нового конфиг-файла `/etc/fluent/fluentd.conf`:**
    Скопируйте содержимое, приведенное выше в разделе "2. Описание Конфигурационного Файла", и вставьте его в файл:
    ```bash
    echo '<source>
  @type http
  port 8888
</source>

<filter test1>
  @type grep
  <exclude>
    key source
    pattern localhost
  </exclude>
</filter>

<match test1>
  @type file
  path /var/log/fluent/access
</match>' | sudo tee /etc/fluent/fluentd.conf
    ```

4.  **Перезагрузка конфигурации Fluentd:**
    ```bash
    sudo systemctl reload fluentd.service
    ```

5.  **Тестирование фильтра:**

    *   **Отправка сообщения, которое ДОЛЖНО быть отброшено (содержит "localhost"):**
        ```bash
        curl -X POST -d 'json={"source":"localhost", "message":"test"}' http://localhost:8888/test1
        ```

    *   **Отправка сообщения, которое ДОЛЖНО быть записано (не содержит "localhost"):**
        ```bash
        curl -X POST -d 'json={"source":"remotehost", "message":"another test"}' http://localhost:8888/test1
        ```

6.  **Проверка логов:**
    Проверьте содержимое директории `/var/log/fluent/access/`. Вы должны увидеть только сообщения, которые не содержали "localhost". Fluentd буферизирует логи, поэтому может потребоваться несколько секунд, чтобы они появились в файле.

    ```bash
    sudo ls -l /var/log/fluent/access/
    sudo cat /var/log/fluent/access/*
    ```

    Вы должны увидеть только сообщение, подобное этому (дата и время могут отличаться):
    `2025-06-16TXX:XX:XX+00:00 test1 {"source":"remotehost","message":"another test"}`
    Сообщение с "localhost" не должно быть в файле.

---
