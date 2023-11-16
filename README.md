### Часть 1: Основы Tarantool

#### Установка и Инициализация:

 • Установите Tarantool на свой компьютер.

```
sudo apt install tarantool
```

 • Создайте пространство (space) данных с именем call_records для хранения записей о телефонных звонках.

Структура записи должна включать следующие поля: call_id (идентификатор звонка), caller_number (номер звонящего),
callee_number (номер принимающего звонок), duration (длительность звонка в секундах).

-> Активируем tarantool

```
tarantool
```

-> Настраиваем, какой порт будем слушать, создаём новое пространство, задём схему пространства

```
box.cfg{listen = 3301}

s = box.schema.space.create('call_records')

s:format({

    {name = 'call_id', type = 'unsigned'},

    {name = 'caller_number', type = 'string'},

    {name = 'callee_number', type = 'string'},

    {name = 'duration', type = 'integer'},

    })
```

#### Операции с данными:

• Добавьте несколько тестовых записей в пространство данных call_records, представляющих собой различные телефонные звонки.

```
s:insert{2 '89275555555', '92777777883', 300}

s:insert{3, '92777777887', '89275555555', 500}

s:insert{4, '92777788885', '92767643455', 750}

s:insert{5, '89274444445', '92777788885', 300}

s:insert{6, '92744499837', '92754322337', 500}

s:insert{7, '92777788882', '92754322337', 750}
```

---

#### Lua-функции:

  Lua-функции:
 • Напишите Lua-функцию для выборки всех звонков, длительность которых превышает 5 минут.

```
function get_long_calls(minutes)
    local space = box.space.call_records
    local duration_seconds = minutes * 60

    local calls = {}

    for _, call in space:pairs() do 
        if call.duration > duration_seconds then 
            table.insert(calls, call)
        end 
    end 
    return calls
end
```

• Реализуйте Lua-функцию для добавления новой записи о звонке.

```
function add_call_record(call_id, caller_number, callee_number, duration)
    box.space.call_records:insert{call_id, caller_number, callee_number, duration }
end

```

• Реализуйте возможность удаления записей о звонках.

```
function delete_by_caller_name(caller_number)
    local space = box.space.call_records

    for _, call in space:pairs() do 
        if call.caller_number == caller_number then 
            local key = call[1]
            space:delete(key)
        end 
    end 
end

```

---

### Часть 2: Оптимизация запросов и Индексы

  Индексы:
 • Создайте индексы для ускорения запросов:
  индекс для поля call_id и индекс для поля caller_number.

#### Очистим данные:

```
s:truncate{}
```

Зададим новую схему (загодя добавим новое поле timestamp):

```
s:format({
        {name = 'call_id', type = 'unsigned'},
         {name = 'caller_number', type = 'string'},
         {name = 'callee_number', type = 'string'},
         {name = 'duration', type = 'integer'},
         {name = 'timestamp', type = 'unsigned'}
         })
```

  Оптимизация запросов:
 • Создайте индексы для ускорения запросов:  индекс для поля call_id и индекс для поля caller_number.

#### Определим индексируемые колонки:

```
 s:create_index('primary', {
            parts = {'caller_id'},
            type = 'tree',
            unique = False
            })


s:create_index('secondary', {
            parts = {'caller_number'},
            type = 'tree',
            unique = False
            })
```

#### Заполним данными:

```
t = os.time(os.date("!*t"))
s:insert{2, '89275555555', '92777777883', 300, t}
s:insert{3, '92777777887', '89275555555', 500, t}
s:insert{4, '92777788885', '92767643455', 750, t}
s:insert{5, '89274444445', '92777788885', 300, t}
s:insert{6, '92744499837', '92754322337', 500, t}
s:insert{7, '92777788882', '92754322337', 750, t}
```

#### Оптимизация запросов:

 • Напишите Lua-функцию, которая выбирает все звонки с указанным номером caller_number
 за последний час.

```
function select_fresh_calls_by_caller_number(caller_number)

    local current_time = os.time(os.date("!*t"))

    local found_calls = box.space.call_records.index.secondary:select({caller_number})

    local result = {}

    for _, call in ipairs(found_calls) do
        if current_time - call.timestamp <= 3600 then
            table.insert(result, call)
        end
    end

    return result

end
```

---

### Часть 3: HTTP-сервер

HTTP-сервер:
 • Реализуйте простой HTTP-сервер, используя Tarantool HTTP-фреймворк или другой подходящий инструмент.
 • Создайте эндпоинт для получения информации о звонке по его идентификатору (call_id).

```
function call_info(self)
    local call_id = tonumber(self:stash('call_id'))
    local r = box.space.call_records.index['primary']:select{call_id}
    local call = r[1]
    local json = require('json').new()
    if r[0] then
    
        self:response():send({ json = { ['call_id'] = call[1],
                                        ['caller_n'] = call[2],
                                        ['callee_n'] = call[3],
                                        ['duration'] = call[4],
                                        ['timestamp'] = call[5]
                                           } })
    else
        self:response():send({ status = 404 })
    end
end

server = require('http.server').new(nil, 9080)
server:route({ method = 'GET', path = '/call/:call_id' }, call_info)
server:start()
```
