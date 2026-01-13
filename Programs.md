# Common tools

## g++
- C++ компилятор
```bash
g++ main.cpp
```
- `-v` - verbose - вывести подробности компиляции (версии, последовательность этапов компиляции)
- `-fno-stack-protector` - скомпилировать без канарейки

## man
- Вывести документацию по программе, syscall'у и пр.
```bash
man syscalls
```

- Посмотреть информацию о главах мануала
```bash
man man
```

- Искать мануал в конкретной главе
	- Пример ниже выведет мануал `write` из 2-й главы (syscall'ы)
```bash
man 2 write
```

## htop
- Рекомендуется к установке
- Как обычный top, но более человеко-ориентированный
```bash
htop
```

## wc
- Word count

- Кол-во строк в файле
```bash
wc -l hello.e
```

## echo
- Вывод в стандартный ввод
```bash
echo "Hello world!"
```

- Вывести статус последней выполненной программы через `$?`
```bash
./program.out
echo $?
```

## tee
- То, что пишется в stdout, параллельно пишется в файл
```bash
ls -l | tee log.txt
```

# C++ tools
## c++filt
- Можно перевести уже сманглированное имя в изначальное:
```bash
c++filt -n _ZNSirsERi
```
- `std::basic_istream< bla-bla > `, короче - `operator>>` для `cin`'а и `int`'а

## nm
- Посмотреть список символов в файле
```bash
nm a.out

# Без манглирования
nm -C a.out
```

- Бывает полезно, решая проблемы вида `Undefined reference to ...`

## gdb
- Дебаггер (gdb)
```bash
g++ -g main.cpp
```
- Размер значительно возрастет: добавяться debug-символы

==TODO==
```bash
gdb a.out

(gdb) b mergesort.cpp:26  # breakpoint
(gdb) bt  # backtrace
(gdb) cont  # continue
(gdb) n
(gdb) s
(gdb) q
(gdb) p
(gdb) watch arr+6
```

- С помощью gdb можно читать Core File'ы
```bash
gdb -c core_dump
```
- Выведет без символов

- А можно указать программу
	- Тогда `gdb` покажет строки, где программа упала
```bash
gdb -c core_dump a.out
```


# Low-level tools
## hexdump
- Расшифровать "бинарь" в последовательность 16-тиричных символов и, по возможности, экранизированные названия сущностей
- `-C` - canonical hex + ASCII display
```bash
hexdump -C hello.o
```

## objdump
- Посмотреть assembler программы
	- Это  из `bin utils` ==TODO==
```bash
objdump -D hello.o
```

## readelf
- Программа рассказывает о ELF-файле
```bash
readelf -a a.out
```

## strip
- Обрезать ненужные куски ELF-файла, которые не сильно нужны (например, `.syntab` - он при исполнении не нужен - нужен только `.dynsym`)

```bash
strip a.out
```

- Бывает полезно, чтобы защитить программу от Reverse Engineering'а

## objcopy
- Сохранить программу посекционно

```bash
objcopy ???
```

## strace
- Какие syscall'ы делает программа
```bash
strace a.out
```

- Можно некоторые позиции понять интуитивно

- Можно мониторить уже запущенные процессы
```bash
strace -p {PID}
```


# Linking & libs

## ldd
- Print shared object dependencies
	- Очень полезная тулза
```bash
ldd a.out
```

## ld
- Стандартный линуксовский линковщик

```bash
ld hello.o /lib/x86_64-linux-gnu/libstdc++.so.6 /lib/x86_64-linux-gnu/libc.so.6
```

## ltrace
- Есть полезная тулза `ltace`, которая регулирует все библиотечные вызовы
	- Пишет в `stderr`, что логично (вообще все логи выводятся в `stderr`, отдельно от основного вывода)

- С флагом `-C` выведет неманглированные имена
```bash
ltrace -C ./a.out 2>2.txt
```

- По факту, подменяет стандартный динамический линковщик на тот, что сверху дополнительно логирует

## lsattr
- У файлов есть дополнительные аттрибуты, если файловая система не суперстарая (появилось при EXT-2, сейчас - EXT-4)
- Поменять аттрибуты через `chattr`

```bash
sudo chattr +i output.txt
ls -l  # Ничего не изменится
lsattr # Появился флаг i
```
- `i` - immutable - нельзя удалять, модифицировать и пр., если ты не суперюзер и без привилегий
- `E` - encrypt
- `F` - сделать все пути нечуствительными к регистру
