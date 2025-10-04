# ELF
- Рассматривая различные исполняемые или "полуисполняемые" файлы (например, `libstdc++.so` или `hello.o`), мы можем увидеть что их структура схожая, то есть одного и того же формата.
- Этот формат называется ELF (Executable and Linkable Format)
	- Это линуксовский формат
	- В самом начале каждого такого файла есть шапка с явным указанием (ELF)
- ELF (Executable and Linkable Format) is the standard file format for executables, object code, shared libraries, and core dumps in Linux and other Unix-like operating systems. It provides the structure for the operating system to correctly interpret and execute machine code, supporting features like dynamic linking and position-independent code. An ELF file contains headers, segments, and sections that describe the code and data, allowing the system to load and execute the program.

- Есть программа, которая рассказывает о ELF-файле
```bash
readelf -a a.out
```
- [[Programs#readelf]]

- Каждый ELF-файл имеет строгую структуру
- ELF has the same layout for all architectures, however endianness and word size can differ; relocation types, symbol types and the like may have platform-specific values, and of course the contained code is architecture specific.
![[elf_layout_2.png|500]]

## ELF for linker and loader
ELF files can be broken down into segments and sections. Segments and sections may overlap (i.e., several sections may compose a single segment). This is because ELF offers two views of the executable files:

- Linker/Section view: As object files are linked together, the linker will have to merge multiple instances of a given section from different object files into one section in the executable.
- Loader/Segment view: Segments classify different regions of the executable or shared library for the purpose of loading into memory for execution.
    - Not all sections are to be loaded into memory for execution (e.g, debugging information)
    - A segment contains one more sections. The diagram above shows two loadable segments, one containing read-only data and one containing read/write data.

**Более расширенное объяснение:**

ELF files are used by two tools: the **linker** and the **loader**. A linker combines multiple ELF files into an executable or a library and a loader loads the executable ELF file in the memory of the process. On real operating systems, loading may require relocation (e.g., if the file is dynamically linked it has to be linked again with all the shared libraries it depends on) - but it's really hard operation.

Linker and loader need two different views of the ELF file, i.e., **they access it differently**.

On the one hand, the linker needs to know where the DATA, TEXT, BSS, and other sections are to merge them with sections from other libraries. If relocation is required the linker needs to know where the symbol tables and relocation information is.

On the other hand, the loader does not need any of these details. It simply needs to know which parts of the ELF file are code (executable), which are data and read-only data, and where to put the BSS in the memory of a process.

Hence, the ELF file provides two separate views on the data inside the ELF file: **a linker view** with several details, and a **loader view**, a higher level view with less details.

To provide these views each ELF file contains two tables (or arrays):
- **Section Header Table** With pointers to **sections** to be used by the **linker**.
- **Program Header Table** With pointers to **segments** to be used by the **loader**.

Both tables are simply arrays of entries that contain information about each part of the ELF file (e.g., where the sections/segments used by the linker/loader are inside the ELF file).

Here is a simple figure of a typical ELF file that starts with the ELF header. The header contains pointers to the locations of Section Header Table and Program Header Table within the ELF file. Then each tables have entries that point to the starting locations of individual sections and segments.

![[elf_layout_3_colored.jpg]]

**It's a bit annoying but the parts of the ELF file used by the linker are called "Sections", and the parts used by the loader are called "segments"** (my guess is that different CPU segments were configured in the past for each part of the program loaded in memory, hence the name "segments", for example, an executable CPU segment was created for the executable parts of the ELF file (i.e., one segment that contained all executable sections like .text, and .init, etc.).

Also don't get confused: sections and segments do overlap. I.e., typically multiple sections (as the linker sees the ELF file, such as .text, and .init) are all contained in one executable segment (what the loader sees). Check the previous image and see how depending on the perspective, same parts of the ELF file belong to one section **and** one segment. Confusing, huh? It will become clear soon.

## ELF header
All ELF files begin with a header that contains metadata for the ELF file, including:
- Four “magic” bytes in the beginning of the file: `0x7f`, `'E'`, `'L'`, `'F'`
- Information about the system on which the ELF binary is meant to run
- Offsets into the file for the Program header (Phdr) table and the Section header (Shdr) table. The former lists the segments and the latter lists the sections in the ELF file.

### Types of ELF files

The Executable and Linkable Format (ELF) defines four primary types of files:
- **Executable File (ET_EXEC):** This is a complete, ready-to-run program. It contains all the necessary information for the operating system to load it into memory and begin execution.
- **Relocatable File (ET_REL) (Position Independent Code):** Also known as an object file (e.g., `.o` files), this type contains code and data that has not yet been linked into a final executable or shared object. It is designed to be combined with other relocatable files and libraries by a linker to form an executable or shared object.
- **Shared Object File (ET_DYN):** These are dynamically linkable object files, commonly referred to as shared libraries (e.g., `.so` files on Linux). They can be loaded and linked into a program's process image at runtime, allowing multiple programs to share the same library code in memory.
- **Core Dump File (ET_CORE):** Generated by the kernel when a program crashes (e.g., due to a segmentation fault), a core dump file is a snapshot of the program's memory and register state at the time of the crash. Its primary purpose is for post-mortem debugging and analysis to determine the cause of the crash.

- `Position-Independent` - адреса функций кодируются (записываются) относительными
- Иначе `-fno-pie` - `EXEC (Position-Dependent)`

Как описываются типы ELF-файлов в "libelf by example":
As an object format, ELF supports multiple kinds of objects:
- Compilers generate relocatable objects that contain fragments of machine
code along with the “glue” information needed when combining multiple
such objects to form a final executable.
- Executables are programs that are in a form that an operating system can
launch in a process. The process of forming executables from collections
of relocatable objects is called linking.
- Dynamically loadable objects are those that can be loaded by an executable
after it has started executing. Dynamically loadable shared libraries are
examples of such objects.


## Symbol table
The ELF symbol table (stored at the `.symtab` section) is a list of all symbols referred to by the executable, namely functions and static variables. It also contains symbols that are referred to in the program by not actually defined by it, such as C library functions like `printf()`.

The symbol table is an array of `Elf64_Sym` structs, defined as follows:

```c
typedef struct {
    uint32_t      st_name;
    unsigned char st_info;
    unsigned char st_other;
    uint16_t      st_shndx;
    Elf64_Addr    st_value;
    uint64_t      st_size;
} Elf64_Sym;
```

A few notes about some fields of interest:

- `st_name` is the index into the symbol table’s string table, stored in the `.strtab` section.
- `st_info` is a packed byte containing the symbol’s type and binding.
- `st_value` is the address of the symbol.
- `st_size` is the size of the symbol. For functions, `st_size` is the total byte size of all of the instructions in the function.

==TODO== add about ## Dynamic Linking in art.3

## Libelf
- Библиотека для работы с ELF-файлами
- Прилагается [книжка](https://atakua.org/old-wp/wp-content/uploads/2015/03/libelf-by-example-20100112.pdf)

![[libelf-by-example-20100112.pdf]]

## Extended information
- [Article 1](https://cs4157.github.io/www/2024-1/lect/15-elf-intro.html)
- [Article 2](https://ics.uci.edu/~aburtsev/238P/hw/hw3-elf/hw3-elf.html)
- [Article 3](https://gist.github.com/x0nu11byt3/bcb35c3de461e5fb66173071a2379779)

![[elf_layout_1.png|700]]

# About linking
- Вообще, есть статическая и динамическая линковки. Статическая линковка (static -> до запуска) осуществляется на этапе линковки (соответственно, все сущности библиотеки статической линковки появляются в исходном файле). При динамической линковке в исходный файл записываются лишь адреса (и указания к подключению той или иной библиотеке). ==TODO==
- Поэтому, на самом деле, определения функций, ответственных за ввод/вывод в консоль, в `a.out` по-прежднему будут отсутствовать: программа обращается к стандартной библиотеке прямо в Runtime, которая, к слову, используется одновременно несколькими программами.
	- То есть `a.out`, как и большинство программ на C++, не самодостаточны
- И благодаря этому используется куда меньше ресурсов оперативной памяти

# Shared Object
- Динамическая библиотека (`.so`)
	- В Windows - DLL (Dynamic Link Library)

- Print shared object dependencies
	- Очень полезная тулза
```bash
ldd a.out
```

==TODO== LD_PRELOAD

# LD
- Стандартный линуксовский линковщик

- Попробуем слинковать зависимости
```bash
ld hello.o /lib/x86_64-linux-gnu/libstdc++.so.6 /lib/x86_64-linux-gnu/libc.so.6
```
- `ld` выведет множество warning'ов о невозможности найти символы для `_start` и т.д.: то есть нужно еще что-то

- Нужно еще `/usr/lib/gcc/x86_64-linux-gnu/12/crtbeginS.o` и `crtendS.o` (C Run TIme)
==TODO== почему с суффиксом `S`?

- Запускаем и пишем `echo $?` (статус окончания последней программы). Получаем 1 (ошибка). То есть еще не достаточно

- Ему не хватает динамической линковки
	- Не путать с обычным линковщиком: динамический линковщик запускается перед фактическим запуском программы. Он создает нам окружение: все библиотеки, стек и т.п. ==TODO==
	- Когда мы запускаем `./a.out`, сначала запускается динамический линкировщик, который затем отдает управление `a.out`

- Сравним через `ldd` файлы - наш и нормально скомпилированный, через `g++`
- Нормальный: есть `/lib64/ld-linux-x86_64.so`
- Наш: есть `/lib/ld64.so`
	- ==TODO== overview


- Теперь слинкуем программу еще и с `-I /lib64/ld-linux-x86_64.so`. Будет также жаловаться на `_start`, зато программа заработает: но в самом конце будет Segmentation Fault.
	- Здесь будет вызываться не `_start`, а сразу `main()`
	- `return 0;` не знает, куда идти: в стеке адрес возврата отсутствует
		- Хотя может это потому, что `std::cin` - глобальная переменная, которую надо сынициализировать до `main`'а ==TODO== what is right?
	- Что же будет со статическими переменными и глобальными объектами, которые создаются в `_start`'е, до `main`'а

# Статическая сборка

- Это когда делаем свой файл, не зависящий от библиотек
- Скомпилируем `hello.cpp` через g++ и `ldd a.out`
	- Зависит от множества библиотек, весит 16Кб

- Скомпилируем так:
```bash
g++ -static hello.cpp
```
- Теперь не зависит ни от чего (т.е. самодостаточный), но весит очень много
	- Вроде даже не нужен динамический линковщик (мб он уже внутри)

- Но рассмотрим проблему: хочу запустить скомпилированный на убунте файл на астра линкуксе
	- Разные версие `libc`, и т.д.
- Хотим, чтобы можно было запустить везде
	- Где совпадают архитектура, интерфейс системных вызовов...


# Создаем библиотеку

- Рассмотрим код
	- В `hello.cpp` `multiply` как бы есть, но он не зарезолвен
```cpp
// mylib.cpp:

int multiply(int a, int b) {
	return a * b;
}


// hello.cpp:

int multiply(int a, int b);

int main() {
	multiply(3, 4);
}
```

- При компиляции `hello.cpp` будет ошибка
```bash
collect2: error: ld returned 1 exit status
```
- `collect2` - обертка над `ld`

```bash
g++ -shared mylib.cpp -o mylib.so
```
- Это либа, по `ldd` видно, что там есть раздел `.dynsym`, где перечислены функции, которые публично видны (но из индексируемого там только multiply)

- Теперь скомпилируем с библиотекой
	- Вообще все библиотеки должны начинаться с `lib` (EX: `libc.so`, `libmylib.so`)
	- Поэтому нужно переименовать на `libmylib.so` и перенести в `/usr/lib` (локально он её не найдет) (или дописать `-L.`, типа libraries in `.` (there!))
```bash
g++ -lmylib hello.cpp
```

- Скомпилируем с вариантом, где либа находится в той же папке, посмотрим в `ldd`
	- Там `libmylib.so => not found`
- Мы также не сможем запустить

- Фактически, дело обстоит так: при компиляции мы попросили компилятор скомпилировать прогу вместе с `libmylib.so`
- Однако компилятор не вписал в программу путь, где эту либу при запуске искать
- Что надо сделать? Есть 3 варианта;
	1) Через переменную окружения (`LD_PRELOAD`)
	2) Через `LD_LIBRARY_PATH=. ./a.out`
	3) Адекватно: указать программе, где искать её зависимость

- Про последнее:
- Добавить при линковке дописать `-Wl,-rpath=dir`
	- У `g++` флага `-rpath` нет, `-Wl` говорит g++, что флаг надо передать линковщику
	- Он запишет адрес прямо в ELF-файл программы

# ltrace

- Есть полезная тулза `ltace`, которая регулирует все библиотечные вызовы
	- Пишет в `stderr`, что логично (вообще все логи выводятся в `stderr`, отдельно от основного вывода)

- С флагом `-C` выведет неманглированные имена
```bash
ltrace -C ./a.out 2>2.txt
```

- По факту, подменяет стандартный динамический линковщий на тот, что логирует дополнительно

# Больше про `.so`

Помимо `.dynsym`, есть `SECTION HEADER`, где указаны `.data`, `.text`, `.bss`

- `.data`, `.bss`, `.rodata` - статические данные (глобальные переменные и тп)
	- `.rodata` - Read Only Data - константы (там же `const char*`))
		- Зачем разделять? Права доступа к памяти
	- `.data` - Статические данные, данные для инициализации которых заготовлены и лежат в коде программы 
	- `.bss` - То, что нужно проинициализировать при запуске, но по факту не проинициализированные (например, массив из 1'000'000 `int`'ов изначально заполнен нулями) ==TODO== Ctrl+C from cpp_cource_notes
- `.text` - текст программы (испольняемый код)

- `symtab` - таблица символов
	- Символ (с точки зрения линковщика) - это сущности, которые потенциально могут иметь адреса (переменные, фунции...)
		- Линковщик работает с символами и ТОЛЬКО с ними
	- Bind (связывания): LOCAL (видны в пределах единицы трансляции (например, `static int x = 5;` в Global scope)), GLOBAL ("торчат" из файла, те видны всем), WEAK (если линковщик видит несколько определений одной сущности, то он выберет слабое ==TODO== или сильное. Пример; `new`, `delete`, ...)
```cpp
// mylib.cpp:
int __attribute__((weak)) a = 3;  // аттрибут для линковщика

// main.cpp:
int a = 4;
```

-  Vis (visibility, видимость): HIDDEN (), DEFAULT ()
==TODO== check example and research MORE (example is AI generated!)

- `dynsym` - символы, которые подлежат динамической линковке

# Core file

- Четверный вид ELF-файла

- Есть дебаггер (gdb)
```bash
g++ -g main.cpp
```
- Размер значительно возрастет: добавяться debug-символы

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

# Core dumped
- Segmentation fault (Core dumped)

- Core Dumped - память сдамплена и сохранена в файл
`/var/lib/apport/coredump`

==TODO== научиться смотреть Core Dump файлы. Будет на экзе!

- Если посмотреть, то его тип - Core File
	- Это файл, который содержит снимок программы на момент падения
	- Этот файл можно читать с помощью `gdb`

```bash
gdb -c core_dump
```
- Выведет без символов

- А можно указать программу
	- Тогда `gdb` покажет строки, где программа упала
```bash
gdb -c core_dump a.out
```
