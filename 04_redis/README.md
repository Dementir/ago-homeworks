# Домашнее задание к занятию «2.1. Кэширование данных - Redis»

Все задачи этого занятия нужно делать в **одном репозитории**.

В качестве результата пришлите ссылки на ваши GitHub-проекты через личный кабинет студента на сайте [netology.ru](https://netology.ru).

**Важно**: если у вас что-то не получилось, то оформляйте Issue [по установленным правилам](../report-requirements.md).

**ВАЖНО**: НИ В КОЕМ СЛУЧАЕ НЕ ПОДСТАВЛЯЙТЕ ДАННЫЕ СВОИХ РЕАЛЬНЫХ КАРТ В КОД! Это очень частая "оплошность": разработчики случайно коммитят и заливают на GitHub "чувствительные" (sensitive) данные (ключи, логины, пароли, адреса и т.д.). Используйте генераторы вроде: https://www.freeformatter.com/credit-card-number-generator-validator.html

Если вы всё же "случайно" залили чувствительные данные на GitHub, то используйте [инструкцию по удалению данных](https://help.github.com/en/github/authenticating-to-github/removing-sensitive-data-from-a-repository). Кроме того, как бы это печально ни было, рекомендуем вам заблокировать карту и заказать в банке новую.

# Как сдавать задачи

1. Создайте на вашем компьютере Go-модуль (см. доп.видео к первой лекции).
1. Добавьте в него в качестве зависимостей pgx v4.
1. Инициализируйте в нём пустой Git-репозиторий.
1. Добавьте в него готовый файл [.gitignore](../.gitignore).
1. Добавьте в этот же каталог остальные необходимые файлы (убедитесь, что они аккуратно разложены по пакетам).
1. Сделайте необходимые коммиты.
1. Создайте публичный репозиторий на GitHub и свяжите свой локальный репозиторий с удалённым.
1. Сделайте пуш (удостоверьтесь, что ваш код появился на GitHub).
1. Ссылку на ваш проект отправьте в личном кабинете на сайте [netology.ru](https://netology.ru).
1. Задачи, отмеченные как необязательные, можно не сдавать, это не повлияет на получение зачета (в этом ДЗ все задачи являются обязательными).

## LRU vs LFU

### Легенда

Достаточно часто, сталкиваясь с новой системой, вам придётся проводить предварительное изучение особенностей различных режимов её работы, чтобы лучше её понять (а не слепо доверять документации). Этим мы и займёмся в этой задаче. 

### Предисловие

Наша задача - разобраться в том, как работает LRU и LFU, которые мы упоминали на лекции. Для этого мы можем настроить Redis, передав ему конфигурационный файл.

<details>
<summary>Справка</summary>

Redis позволяет выполнять скрипты на языке Lua. Нас особо сам язык не интересует, интересует лишь возможность использовать его для быстрого создания большого количества записей (эти скрипты не нужно использовать в production, они нужны только в качестве вспомогательного инструмента для лабораторной работы).

Скрипты выглядят следующим образом:

1. `eval 'for i=1, 1000 do redis.call("SET", KEYS[1] .. i, i) end' 1 A` - создаёт тысячу ключей A1-A1000 со значениями 1-1000
1. `eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 A` - выполняет GET для ключей A1-A1000

Дополнительные команды, которые нам понадобятся:
1. `FLUSHALL` - удаляет все записи
1. `KEYS *` - показывает все ключи (используется только для отладки)
1. `KEYS A*` - показывает все ключи, начинающиеся с префикса A

</details>

### Общая часть

1\. Для выполнения задания используйте следующий `docker-compose.yml`:

```yaml
version: '3.7'
services:
  bankcache:
    image: redis:6.0-alpine
    ports:
      - 6379:6379
    volumes:
      - ./conf/redis.conf:/usr/local/etc/redis/redis.conf
    command: [ "redis-server", "/usr/local/etc/redis/redis.conf" ]
```

2\. Возьмите [redis.conf](assets/redis.conf) и положите в каталог `conf` вашего проекта.

#### LRU

3\. В файле redis.conf раскомментируйте указанные строки и установите следующие значения:

`maxmemory 1mb`

`maxmemory-policy allkeys-lru`

`maxmemory-samples 10`

4\. Запустите контейнер и подключитесь к redis-cli.

5\. Убедитесь, что настройки выставлены правильно с помощью команды `info memory`:

```
maxmemory_human:1.00M
maxmemory_policy:allkeys-lru
```

6\. Выполните следующие команды:

```
eval 'for i=1, 1000 do redis.call("SET", KEYS[1] .. i, i) end' 1 A

eval 'for i=1, 1000 do redis.call("SET", KEYS[1] .. i, i) end' 1 B

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 A

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 A

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 A

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 B

eval 'for i=1, 2000 do redis.call("SET", KEYS[1] .. i, i) end' 1 C

keys A*

keys B*

keys C*
``` 

7\. Подсчитайте количество оставшихся ключей A, B и C.

8\. Выполните команду `flushall`, после чего повторите п.5-п.6, чтобы получить средний результат.

9\. Удалите контейнер с помощью команды `docker-compose rm`.

#### LFU

3\. В файле redis.conf раскомментируйте указанные строки и установите следующие значения:

`maxmemory 1mb`

`maxmemory-policy allkeys-lfu`

`maxmemory-samples 10`

4\. Запустите контейнер (убедитесь, что вы создали новый контейнер, а не запустили его с предыдущими настройками) и подключитесь к redis-cli.

5\. Убедитесь, что настройки выставлены правильно с помощью команды `info memory`:

```
maxmemory_human:1.00M
maxmemory_policy:allkeys-lfu
```

6\. Выполните следующие команды:

```
eval 'for i=1, 1000 do redis.call("SET", KEYS[1] .. i, i) end' 1 A

eval 'for i=1, 1000 do redis.call("SET", KEYS[1] .. i, i) end' 1 B

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 A

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 A

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 A

eval 'for i=1, 1000 do redis.call("GET", KEYS[1] .. i) end' 1 B

eval 'for i=1, 2000 do redis.call("SET", KEYS[1] .. i, i) end' 1 C

keys A*

keys B*

keys C*
``` 

7\. Подсчитайте количество оставшихся ключей A, B и C.

8\. Выполните команду `flushall`, после чего повторите п.5-п.6, чтобы получить средний результат.

### Результаты

В качестве результата пришлите ссылку на ваш репозиторий, а также в тексте ответа укажите усреднённые значения количества оставшихся ключей A, B и C и ваше объяснение результата (почему так произошло).
