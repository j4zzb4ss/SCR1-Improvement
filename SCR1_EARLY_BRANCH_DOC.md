# Документация по добавленному макросу для процессора SCR1

## `SCR1_EARLY_BRANCH`

При включении макроса в файле `scr1_arch_description.svh`
мы меняем место обработки ветвлений с исполнительного блока
EXU на декодирующий блок IDU. Все изменения кода защищены макросом `SCR1_EARLY_BRANCH`.

Засчёт этого мы сокращаем штраф за переход типа branch и при максисимальной конфигурации
процессора в симуляции Verilator получаем выигрыш в скорости на **4.7% в CoreMark**.

## Схема обработки ветвлений

```text
                         ┌───────────────┐
                         │     MPRF      │
                         │               │
                         │ rs1/rs2 data  │
                         └──────┬────────┘
                                │
                                ▼
┌──────────┐  instr + PC  ┌──────────┐  early result  ┌──────────┐
│   IFU    │─────────────►│   IDU    │───────────────►│   EXU    │
│          │◄─────────────│          │                │ verifies │
└──────────┘ taken,target └──────────┘                └────┬─────┘
     ▲                                                     │
     └──────────────── correction redirect ────────────────┘
```

## Ниже приведен полный список изменений для макроса

| название файла                | правки                                                    | зачем нужно                                   |
| ----------------------------- | --------------------------------------------------------- | --------------------------------------------- |
| `scr1_riscv_isa_decoding.svh` | Добавлены поля `early_branch_valid`, `early_branch_taken` | Передача результата раннего вычисления в EXU  |
| `scr1_pipe_mprf.sv`           | Добавлены порты чтения для IDU и bypass                   | Получение `rs1` и `rs2` для вычисления в IDU  |
| `scr1_pipe_idu`               | Добавлено вычисление branch condition и target            | Раннее определение перехода                   |
| `scr1_pipe_ifu.sv`            | Добавлены новые порты и контроль PC                       | Ранний redirect IFU                           |
| `scr1_pipe_exu.sv`            | Добавлена повторная проверка                              | Проверка результата IDU и correction redirect |

---

## 1) `scr1_riscv_isa_decoding.svh`

В структуру `type_scr1_exu_cmd_s` добавлены поля

```systemverilog
`ifdef SCR1_EARLY_BRANCH
logic early_branch_valid; // IDU в действительности вычислил результат ветвления
logic early_branch_taken; // Сам результат вычисления
`endif
```

---

## 2) `scr1_pipe_mprf.sv`

Добавлены новые порты чтения адресов запрашиваемых IDU и новые порты передачи
содержимого регистров для дальнейшего вычисления в IDU

```text
idu2mprf_rs1_addr_i
mprf2idu_rs1_data_o

idu2mprf_rs2_addr_i
mprf2idu_rs2_data_o
```

И организован bypass на случай работы EXU с запрашиваемым IDU регистром.
Мы просто подхватим значение из writeback EXU до обновления регистров,
таким образом получим корректное значение содержимого регистра, а не устаревшее.

> **!ВНИМАНИЕ:** `SCR1_MPRF_RAM = disabled`, пока только на такой режим работы рассчитываем,
> так как в противном случае чтение регистров может быть исключительно синхронным.

---

## 3) `scr1_pipe_idu`

Теперь IDU не исключительно комбинационный, с активным макросом он приобретает дополнительный
функционал.

Получает PC декодируемой инструкции;
Читает rs1 и rs2;
Вычисляет branch condition и target.

Раннее ветвление обрабатывается лишь при выполнении рукопожатия.

```systemverilog
assign idu_accept =
      ifu2idu_vd_i
    & exu2idu_rdy_i;
```

Далее проверяется:

```systemverilog
if (idu_accept
    && ~ifu2idu_imem_err_i
    && ~idu2exu_cmd_o.exc_req
    && idu2exu_cmd_o.branch_req)
```

Это означает:

* IFU передал valid-инструкцию;
* EXU готов её принять;
* нет ошибки чтения инструкции;
* декодирование не создало exception;
* инструкция является условной ветвью.

Вычисляется новый адрес PC при проверке условий ветвлений.

```text
branch_target = branch_pc + sign_extended_immediate
```

Передаются флажки раннего похвата ветвления декодером, EXU видит это.

```systemverilog
idu2exu_cmd_o.early_branch_valid = 1'b1;
idu2exu_cmd_o.early_branch_taken = branch_taken_early;
```

---

## 4) `scr1_pipe_ifu.sv`

IFU получил новые порты

```text
idu2ifu_branch_req_i
idu2ifu_branch_target_i
ifu2idu_pc_o

ifu_idu_pc_ff
ifu_idu_instr_size
ifu_idu_accept
pc_new_req_internal
pc_new_addr_internal
```

Добавил регистр для контроля PC, передаваемого IDU. Чтобы не происходило
случайного взятия инструкции вобход очереди инструкций IFU.

### Такт N:

```text
IFU выдаёт branch и её PC в IDU.
IDU читает rs1/rs2 из MPRF.
IDU вычисляет condition и target.
```

### Такт N, combinational:

```text
branch_taken_early = 1.
idu2ifu_branch_req = 1.
idu2ifu_branch_target = branch_pc + immediate.
```

### Следующий фронт:

```text
IFU очищает старый последовательный поток.
IFU устанавливает fetch PC = branch target.
Branch передаётся в EXU вместе с:
    early_branch_valid = 1
    early_branch_taken = 1
```

---

## 5) `scr1_pipe_exu.sv`

Позже в EXU:

```text
EXU повторно вычисляет condition.
actual taken = early taken.
Mismatch отсутствует.
Повторный redirect в IFU не создаётся.
Внутренний PC EXU обновляется на branch target.
```
