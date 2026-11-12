# Instruction Memory Skid Buffer

Для оптимизации интерфейса выборки инструкций был добавлен однозаписный skid-buffer, управляемый макросом `SCR1_IMEM_SKID_BUF`.

Буфер расположен не внутри IFU, а на верхнем уровне между интерфейсом instruction memory ядра и внешней memory subsystem.

Было:
```text
IFU → IMEM interface → cache / AXI / TCM
```

Стало:
```text
IFU → IMEM interface → SKID BUFFER → cache / AXI / TCM
```

Основная цель изменения — разорвать чувствительный handshake-путь instruction memory без обязательного добавления полного такта задержки для каждого запроса.

Все изменения данной оптимизации защищены макросом `SCR1_IMEM_SKID_BUF`.

## Проблема прямого handshake

В исходной схеме готовность instruction memory могла непосредственно влиять на сигнал подтверждения запроса ядру.

Упрощённо:
```text
IFU request
    ↓
memory ready
    ↓
IFU request acknowledge
```

Таким образом между внешней memory subsystem и IFU образовывался комбинационный путь.

Простое добавление регистра в сигнал `ack` действительно разрывает этот путь, однако создаёт дополнительную задержку для каждого запроса инструкции.

В экспериментальной конфигурации вариант с полностью зарегистрированным acknowledge показал существенное ухудшение производительности.

Поэтому вместо обычного register stage был использован fall-through skid-buffer.

## Структура skid-buffer

Буфер имеет глубину одну запись.

Основное состояние:
```text
imem_skid_vd_ff
imem_skid_addr_ff
```

При необходимости вместе с адресом сохраняются остальные управляющие сигналы instruction-memory request.

Логически схема выглядит так:
```text
                ┌─────────────┐
IFU request ───►│ skid buffer │────► memory
                └─────────────┘
```

Буфер работает в двух режимах:
```text
EMPTY
FULL
```

В состоянии `EMPTY` новый запрос может быть передан downstream непосредственно.

В состоянии `FULL` сохранённый запрос удерживается до тех пор, пока memory subsystem не сможет его принять.

## Fall-through режим

Главная особенность skid-buffer — отсутствие обязательной регистрации каждого запроса.

Если downstream готов:
```text
IFU request
     ↓
  combinational
     ↓
memory request
```

Запрос проходит напрямую без дополнительного pipeline cycle.

То есть в нормальном случае:
```text
skid empty
+
memory ready
→
request проходит сразу
```

Если же downstream временно не готов:
```text
IFU request
     ↓
memory not ready
     ↓
request сохраняется в skid
```

После этого запрос удерживается уже регистрами:
```text
skid register
     ↓
memory
```

Таким образом регистрация происходит только при backpressure.

## Подтверждение запроса ядру

В нашей реализации возможность принять новый запрос определяется занятостью skid-buffer.

Использовалась логика вида:
```systemverilog
core_imem_req_ack = ~imem_skid_vd_ff;
```

Если буфер пуст:
```text
imem_skid_vd_ff = 0
→
IFU может передать новый request
```

Если предыдущий запрос уже сохранён:
```text
imem_skid_vd_ff = 1
→
новый request временно не принимается
```

Это предотвращает потерю запроса при остановке downstream-интерфейса.

## Работа без backpressure

Типичный быстрый случай:
```text
Cycle N

IFU request
memory ready
skid empty

        ↓

request сразу передаётся в memory
```

Skid-register при этом не используется.

Поэтому в отличие от обычного зарегистрированного интерфейса каждый instruction fetch не получает дополнительный такт задержки.

## Работа при backpressure

Если memory subsystem не может принять запрос:
```text
Cycle N

IFU request
memory not ready
        ↓
request сохраняется
```

Состояние:
```text
imem_skid_vd_ff = 1
imem_skid_addr_ff = requested address
```

После этого:
```text
Cycle N+1 / N+2 / ...

skid request
      ↓
memory
```

Запрос остаётся стабильным до момента его принятия.

После успешной передачи:
```text
imem_skid_vd_ff ← 0
```

и IFU снова может отправить следующий запрос.

## Почему не использовался обычный register stage

Рассматривался более простой вариант:
```text
IFU
 ↓
register
 ↓
memory
```

Он хорошо разрывает timing path, но превращает каждое обращение к instruction memory в обязательную зарегистрированную транзакцию.

В результате даже при полностью готовой memory subsystem появляется дополнительная latency:
```text
request
 ↓
register
 ↓
next cycle
 ↓
memory
```

В тестах такой вариант оказался заметно медленнее.

Для зарегистрированного `IMEM_ACK` наблюдалось примерно:
```text
6437 cycles
```

вместо порядка:
```text
5432 cycles
```

для рабочей cache-конфигурации.

Поэтому обязательная регистрация acknowledge была отвергнута.

## Почему не использовался FIFO

Также рассматривался полноценный FIFO между IFU и memory subsystem.

Схема:
```text
IFU → FIFO → memory
```

Однако дополнительная буферизация и управляющая логика увеличили latency instruction-fetch path.

Экспериментальный вариант показывал порядка:
```text
6935 cycles
```

что значительно хуже однозаписного fall-through skid-buffer.

Для данного интерфейса большая глубина очереди не давала преимущества, поскольку основной задачей было не накопление большого количества запросов, а кратковременное удержание одного запроса при backpressure.

## Почему skid-buffer оказался лучше

Skid-buffer сочетает два свойства:
```text
при ready:
    минимальная latency

при not-ready:
    зарегистрированное удержание request
```

То есть:
```text
fast path:
IFU ─────────────► memory

slow path:
IFU → skid reg ──► memory
```

В результате большая часть обращений проходит без дополнительного такта, но при необходимости интерфейс всё равно может корректно остановиться.

## Расположение в системе

Skid-buffer был реализован на верхнем уровне AXI/IMEM-обвязки.

Архитектурно:
```text
        SCR1 core
            │
            │ core_imem_*
            ▼
    ┌─────────────────┐
    │   SKID BUFFER   │
    └────────┬────────┘
             │
             ▼
       cache / AXI
             │
             ▼
        external memory
```

Сам `scr1_pipe_ifu.sv` для реализации skid-buffer существенно не изменялся.

Для IFU внешний интерфейс продолжает выглядеть как обычный instruction-memory interface.

## Взаимодействие с I-cache

При использовании instruction cache skid-buffer располагается перед внешней memory subsystem и не изменяет принцип работы самого кэша.

Упрощённо:
```text
IFU
 ↓
I-cache
 ↓
IMEM interface
 ↓
skid-buffer
 ↓
AXI / TCM / external memory
```

При cache hit обращение к внешней памяти вообще не требуется.

При cache miss skid-buffer помогает корректно обрабатывать handshake с более медленным downstream-интерфейсом.

Таким образом cache уменьшает количество внешних обращений, а skid-buffer оптимизирует поведение оставшихся транзакций.

## Влияние на critical path

Основной смысл skid-buffer с точки зрения timing — убрать необходимость напрямую протягивать backpressure внешней memory subsystem до логики ядра.

Без буфера возможен путь:
```text
memory ready
    ↓
IMEM acknowledge logic
    ↓
IFU control
```

При использовании skid-buffer состояние интерфейса локализуется через:
```text
imem_skid_vd_ff
```

То есть внешний stall может быть сохранён локально, а не распространяться непосредственно через весь combinational handshake path.

При этом за счёт fall-through режима не требуется обязательная дополнительная pipeline-stage на нормальном пути.

## Основные изменения файлов

| Файл | Изменение |
|---|---|
| `scr1_arch_description.svh` | Макрос `SCR1_IMEM_SKID_BUF` |
| AXI/top-level файл | Логика однозаписного skid-buffer |
| IMEM handshake logic | Изменение формирования `core_imem_req_ack` |
| `scr1_pipe_ifu.sv` | Архитектурных изменений для skid-buffer не требуется |

Основная логика оптимизации находится именно во внешней обвязке ядра.

## Результат

Для рабочей cache-конфигурации CoreMark результат до добавления skid-buffer составлял примерно:
```text
5432 cycles
```

После включения `SCR1_IMEM_SKID_BUF`:
```text
5423 cycles
```

То есть выигрыш по числу циклов небольшой:
```text
≈ 9 cycles
≈ 0.17 %
```

Однако основная цель данной оптимизации заключалась не в большом уменьшении CPI, а в улучшении структуры timing path без введения обязательной latency instruction fetch.

После добавления skid-buffer post-route критический путь составлял порядка:
```text
29.33 ns
```

При этом альтернативные варианты с обязательной регистрацией интерфейса существенно ухудшали производительность:
```text
cache baseline       ≈ 5432 cycles
skid-buffer          ≈ 5423 cycles
registered IMEM ACK  ≈ 6437 cycles
FIFO                 ≈ 6935 cycles
```

Таким образом однозаписный fall-through skid-buffer оказался наиболее подходящим компромиссом:
```text
минимальная latency при нормальной работе
+
корректное удержание request при backpressure
+
разрыв нежелательных handshake-зависимостей
+
минимальные аппаратные затраты
```

Основная идея оптимизации заключается в том, чтобы регистрировать request не всегда, а только тогда, когда downstream действительно не может принять его сразу.
