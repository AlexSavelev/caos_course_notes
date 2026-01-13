# 1. Этапы компиляции

Стадии компиляции кода:
1. Препроцессинг
2. Транслирование (Компиляция)
3. Ассемблирование
4. Линковка

Рассмотрим каждый из этапов компиляции.
В рабочей директории создадим файл `hello.cpp` со следующим содержимым:

```cpp
#include <iostream>

int main() {
	int x;
	
	std::cin >> x;
	std::cout << x + 5;
}
```

## 1.1 Препроцессинг
- Этакая подготовка C++ кода к "настоящей" компиляции
- cpp -> cpp

**Препроцессором выполняются следующие действия:**
- Замена соответствующих диграфов и триграфов на эквивалентные символы «`#`» и «`\`»
- Удаление экранированных символов перевода строки
- Замена строчных и блочных комментариев пустыми строками (с удалением окружающих пробелов и символов табуляции)
- Вставка (включение) содержимого произвольного файла (`#include`)
- Макроподстановки (`#define`)
- Условная компиляция (`#if`, `#ifdef`, `#elif`, `#else`, `#endif`)
- Вывод сообщений (`#warning`, `#error`)
- Обработка `#pragma` директив

**Примеры:**
- Макросы с переменным числом аргументов:
  ```cpp
  #define LOG(...) printf(__VA_ARGS__)
  ```
- `#pragma once` — защита от множественного включения

**Выполнение только стадии препроцессора:**
```bash
g++ -E -v hello.cpp > hello.e
```

Узнаем количество строк в файле (будет >30'000 строк):
```bash
wc -l hello.e
```

Заметим, что даже после подстановки всех header'ов все упоминания `std::cin` и `std::cout` — это лишь объявления через `extern`. Сами они нигде пока не определены.

## 1.2 Трансляция (компиляция)
- Перевод кода в ассемблер - язык процессорных инструкций
	- cpp -> asm
- Осуществляется манглирование имен, инстанцирование шаблонов, рассахаривание ООП и многое другое

```bash
g++ -S hello.cpp > hello.s
```
- Здесь всего 18 строк

**Оптимизации на этапе трансляции:**
```bash
g++ -S -O2 hello.cpp -o hello_opt.s  # Развертывание циклов, инлайнинг
```

## 1.3 Ассемблирование
- Перевод ассемблера в бинарный формат
- Самая легкая операция - просто перевод уже-и-так-не-человеко-языка в бинарный формат

```bash
g++ -c hello.s
```

Out: `hello.o` (object file). Прочитать уже невозможно

**Содержимое объектного файла:**
- Машинный код (секция `.text`)
- Данные (`.data`, `.rodata`, `.bss`)
- Таблица символов (`.symtab`)
- Релокационные записи (`.rela.text`, `.rela.data`)
- Информация для отладки (если `-g`)

**Инструменты для анализа:**
```bash
hexdump -C hello.o         # HEX-дамп
objdump -D hello.o         # Дизассемблирование

objdump -t hello.o         # Таблица символов
readelf -S hello.o         # Секции ELF
```
- HEX-дамп значит расшифровать "бинарь" в последовательность 16-тиричных символов и, по возможности, экранизированные названия сущностей

## 1.4 Линковка
- Необходима, даже если всего 1 cpp'шник.
- Присоединяет к написанному коду стандарные сущности (`std::cin`, `std::cout`, `malloc`, `new`, ...)
- Разрешает символы — ищет определения для объявлений

**Механизм линковки:**
1. Собирает все `.o` файлы
2. Строит общую таблицу символов
3. Разрешает символы (undefined symbols)
4. Исправляет адреса (релокация)
5. Создает секции выходного файла

В коде `hello.o` (если посмотреть, скажем, через `hexdump`) лежат латинские названия функций, которые необходимо найти и присоединить
- Для точного распознавания и исключения коллизии как раз и необходимо манглирование имен
- Если посмотреть на ассемблер `hello.o`, то адреса функций, которые вызываются в этом файле, но здесь не определены, будут пустыми (zero)

После линковки содержимое файла значительно возрастает, а адреса вызовов функций заменяются на нормальные.

## 1.5 Комментарии
`.o` файл — промежуточный объектный файл, содержащий машинный код из одного исходного файла.
`.out` файл — финальный исполняемый файл, созданный линковщиком из одного или нескольких `.o` файлов.

> A `.o` file is an intermediate object file, containing machine code from a single source file, while a `.out` file is the final executable file, produced by a linker that combines one or more `.o` files.

# 2. ELF (Executable and Linkable Format)

Рассматривая различные исполняемые или "полуисполняемые" файлы (например, `libstdc++.so` или `hello.o`), мы можем увидеть что их структура схожа — одного и того же формата.

Этот формат называется ELF (Executable and Linkable Format):
- Это линуксовский формат
- В самом начале каждого такого файла есть шапка с явным указанием (ELF)

ELF — стандартный формат файлов для исполняемых файлов, объектного кода, разделяемых библиотек и core-дампов в Linux и других Unix-подобных ОС. Он предоставляет структуру для корректной интерпретации и выполнения машинного кода, поддерживая динамическую линковку и позиционно-независимый код. ELF-файл содержит заголовки, сегменты и секции, описывающие код и данные.

>ELF (Executable and Linkable Format) is the standard file format for executables, object code, shared libraries, and core dumps in Linux and other Unix-like operating systems. It provides the structure for the operating system to correctly interpret and execute machine code, supporting features like dynamic linking and position-independent code. An ELF file contains headers, segments, and sections that describe the code and data, allowing the system to load and execute the program.

- Есть программа, которая рассказывает о ELF-файле
```bash
readelf -a a.out
```
- [[Programs#readelf]]

## 2.1 Структура ELF

Каждый ELF-файл имеет строгую структуру. ELF имеет одинаковый макет для всех архитектур, однако порядок байт (endianness) и размер слова могут различаться; типы релокации, типы символов могут иметь платформенно-специфичные значения, и, конечно, содержащийся код зависит от архитектуры.

>ELF has the same layout for all architectures, however endianness and word size can differ; relocation types, symbol types and the like may have platform-specific values, and of course the contained code is architecture specific.

![[elf_layout_2.png|500]]

## 2.2 Две точки зрения на ELF: для линковщика и загрузчика
ELF-файлы могут быть разделены на сегменты (segments) и секции (sections). Сегменты и секции могут перекрываться (несколько секций могут составлять один сегмент). Это потому, что ELF предлагает два представления исполняемых файлов:

- **Представление для линковщика (секционное):** При связывании объектных файлов линковщик должен объединить несколько экземпляров данной секции из разных объектных файлов в одну секцию в исполняемом файле.
- **Представление для загрузчика (сегментное):** Сегменты классифицируют различные регионы исполняемого файла или разделяемой библиотеки для целей загрузки в память для выполнения.
    - Не все секции должны быть загружены в память для выполнения (например, отладочная информация)
    - Сегмент содержит одну или более секций. На диаграмме выше показаны два загружаемых сегмента: один содержит данные только для чтения, другой — данные для чтения/записи.

> ELF files can be broken down into segments and sections. Segments and sections may overlap (i.e., several sections may compose a single segment). This is because ELF offers two views of the executable files:
> 
> - Linker/Section view: As object files are linked together, the linker will have to merge multiple instances of a given section from different object files into one section in the executable.
> - Loader/Segment view: Segments classify different regions of the executable or shared library for the purpose of loading into memory for execution.
>     - Not all sections are to be loaded into memory for execution (e.g, debugging information)
>     - A segment contains one more sections. The diagram above shows two loadable segments, one containing read-only data and one containing read/write data.

**Более расширенное объяснение:**

ELF-файлы используются двумя инструментами: **линковщиком** и **загрузчиком**. Линковщик комбинирует несколько ELF-файлов в исполняемый файл или библиотеку, а загрузчик загружает исполняемый ELF-файл в память процесса. В реальных ОС загрузка может требовать релокации (например, если файл динамически связан, он должен быть снова связан со всеми разделяемыми библиотеками, от которых зависит).

Линковщику и загрузчику нужны два разных представления ELF-файла, то есть **они обращаются к нему по-разному**.

С одной стороны, линковщику нужно знать, где находятся секции DATA, TEXT, BSS и другие, чтобы объединить их с секциями из других библиотек. Если требуется релокация, линковщику нужно знать, где находятся таблицы символов и информация о релокации.

С другой стороны, загрузчику не нужны эти детали. Ему просто нужно знать, какие части ELF-файла являются кодом (исполняемыми), какие — данными и данными только для чтения, и где разместить BSS в памяти процесса.

Таким образом, ELF-файл предоставляет два отдельных представления данных внутри: **представление для линковщика** с множеством деталей и **представление для загрузчика**, более высокоуровневое с меньшим количеством деталей.

Чтобы предоставить эти представления, каждый ELF-файл содержит две таблицы (или массива):
- **Таблица заголовков секций (Section Header Table)** с указателями на **секции** для использования **линковщиком**.
- **Таблица заголовков программ (Program Header Table)** с указателями на **сегменты** для использования **загрузчиком**.

Обе таблицы — это просто массивы записей, содержащих информацию о каждой части ELF-файла (например, где находятся секции/сегменты, используемые линковщиком/загрузчиком, внутри ELF-файла).

**Важно:** Части ELF-файла, используемые линковщиком, называются "секциями", а части, используемые загрузчиком, называются "сегментами". Секции и сегменты перекрываются. Обычно несколько секций (как видит линковщик, например `.text`, `.init`) содержатся в одном исполняемом сегменте (что видит загрузчик).

> ELF files are used by two tools: the **linker** and the **loader**. A linker combines multiple ELF files into an executable or a library and a loader loads the executable ELF file in the memory of the process. On real operating systems, loading may require relocation (e.g., if the file is dynamically linked it has to be linked again with all the shared libraries it depends on) - but it's really hard operation.
> 
> Linker and loader need two different views of the ELF file, i.e., **they access it differently**.
> 
> On the one hand, the linker needs to know where the DATA, TEXT, BSS, and other sections are to merge them with sections from other libraries. If relocation is required the linker needs to know where the symbol tables and relocation information is.
> 
> On the other hand, the loader does not need any of these details. It simply needs to know which parts of the ELF file are code (executable), which are data and read-only data, and where to put the BSS in the memory of a process.
> 
> Hence, the ELF file provides two separate views on the data inside the ELF file: **a linker view** with several details, and a **loader view**, a higher level view with less details.
> 
> To provide these views each ELF file contains two tables (or arrays):
> - **Section Header Table** With pointers to **sections** to be used by the **linker**.
> - **Program Header Table** With pointers to **segments** to be used by the **loader**.
> 
> Both tables are simply arrays of entries that contain information about each part of the ELF file (e.g., where the sections/segments used by the linker/loader are inside the ELF file).

Here is a simple figure of a typical ELF file that starts with the ELF header. The header contains pointers to the locations of Section Header Table and Program Header Table within the ELF file. Then each tables have entries that point to the starting locations of individual sections and segments.

![[elf_layout_3_colored.jpg]]

## 3.3 ELF header
Все ELF-файлы начинаются с заголовка, содержащего метаданные для ELF-файла:
- Четыре "магических" байта в начале файла: `0x7f`, `'E'`, `'L'`, `'F'`
- Информация о системе, для которой предназначен ELF-бинар
- Смещения в файле для таблицы заголовков программ (Phdr) и таблицы заголовков секций (Shdr). Первая перечисляет сегменты, вторая — секции в ELF-файле.

> All ELF files begin with a header that contains metadata for the ELF file, including:
> - Four “magic” bytes in the beginning of the file: `0x7f`, `'E'`, `'L'`, `'F'`
> - Information about the system on which the ELF binary is meant to run
> - Offsets into the file for the Program header (Phdr) table and the Section header (Shdr) table. The former lists the segments and the latter lists the sections in the ELF file.

**Просмотр заголовка:**
```bash
readelf -h a.out
```

## 3.4 Types of ELF files
Executable and Linkable Format (ELF) определяет четыре основных типа файлов:

- **Исполняемый файл (ET_EXEC):** Полная, готовая к запуску программа. Содержит всю необходимую информацию для загрузки в память и начала выполнения.
- **Релоцируемый файл (ET_REL):** Также известен как объектный файл (например, `.o` файлы). Содержит код и данные, которые еще не были связаны в финальный исполняемый файл или разделяемый объект. Предназначен для объединения с другими релоцируемыми файлами и библиотеками линковщиком для формирования исполняемого файла или разделяемого объекта.
- **Разделяемый объектный файл (ET_DYN):** Динамически связываемые объектные файлы, обычно называемые разделяемыми библиотеками (например, `.so` файлы в Linux). Могут быть загружены и связаны с образом процесса программы во время выполнения, позволяя нескольким программам разделять один и тот же библиотечный код в памяти.
- **Core-дамп файл (ET_CORE):** Генерируется ядром при падении программы (например, из-за segmentation fault). Core-дамп — это снимок памяти программы и состояния регистров на момент падения. Основное предназначение — отладка post-mortem и анализ для определения причины падения.

> The Executable and Linkable Format (ELF) defines four primary types of files:
> - **Executable File (ET_EXEC):** This is a complete, ready-to-run program. It contains all the necessary information for the operating system to load it into memory and begin execution.
> - **Relocatable File (ET_REL) (Position Independent Code):** Also known as an object file (e.g., `.o` files), this type contains code and data that has not yet been linked into a final executable or shared object. It is designed to be combined with other relocatable files and libraries by a linker to form an executable or shared object.
> - **Shared Object File (ET_DYN):** These are dynamically linkable object files, commonly referred to as shared libraries (e.g., `.so` files on Linux). They can be loaded and linked into a program's process image at runtime, allowing multiple programs to share the same library code in memory.
> - **Core Dump File (ET_CORE):** Generated by the kernel when a program crashes (e.g., due to a segmentation fault), a core dump file is a snapshot of the program's memory and register state at the time of the crash. Its primary purpose is for post-mortem debugging and analysis to determine the cause of the crash.

**Position-Independent Executable (PIE):** Исполняемые файлы, которые могут быть загружены по любому адресу в памяти (адреса функций кодируются относительными). Создаются с флагом `-pie`. Иначе `-no-pie` или `-fno-pie` создает `EXEC (Position-Dependent)`.

## 2.5 Symbol table
Таблица символов ELF (хранится в секции `.symtab`) — это список всех символов, на которые ссылается исполняемый файл, а именно функций и статических переменных. Также содержит символы, на которые есть ссылки в программе, но которые фактически не определены в ней, такие как функции стандартной библиотеки C (`printf()`).

Таблица символов — это массив структур `Elf64_Sym`, определенных следующим образом:

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

**Важные поля:**
- `st_name` — индекс в таблице строк символов, хранящейся в секции `.strtab`.
- `st_info` — упакованный байт, содержащий тип и привязку символа.
- `st_value` — адрес символа.
- `st_size` — размер символа. Для функций `st_size` — общий размер в байтах всех инструкций в функции.

**Просмотр таблицы символов:**
```bash
readelf -s a.out
nm a.out
```

> The ELF symbol table (stored at the `.symtab` section) is a list of all symbols referred to by the executable, namely functions and static variables. It also contains symbols that are referred to in the program by not actually defined by it, such as C library functions like `printf()`.
> 
> The symbol table is an array of `Elf64_Sym` structs, defined as follows:
> 
> A few notes about some fields of interest:
> 
> - `st_name` is the index into the symbol table’s string table, stored in the `.strtab` section.
> - `st_info` is a packed byte containing the symbol’s type and binding.
> - `st_value` is the address of the symbol.
> - `st_size` is the size of the symbol. For functions, `st_size` is the total byte size of all of the instructions in the function.

## 2.6 Динамическое связывание (Dynamic Linking)
Динамические библиотеки (`*.so`) загружаются в память один раз и могут использоваться несколькими процессами.

**Механизм:**
- Вместо кода библиотеки в исполняемый файл вставляются **заглушки (PLT — Procedure Linkage Table)**.
- При первом вызове функции происходит **ленивое связывание (lazy binding)**:
  1. Динамический линковщик (`ld.so`) ищет функцию в библиотеках
  2. Записывает её адрес в **GOT (Global Offset Table)**
  3. Перенаправляет вызов на реальную функцию
- Последующие вызовы идут напрямую через GOT.

**Просмотр динамических символов:**
```bash
readelf --dyn-syms a.out
nm -D a.out
```

## 2.7 Утилиты для работы с ELF
**objcopy — манипуляция объектными файлами:**
```bash
# Извлечь только секцию .text
objcopy -O binary -j .text a.out code.bin

# Добавить секцию
objcopy --add-section .mydata=data.bin a.out

# Изменить символ
objcopy --redefine-sym old=new a.out

# Удалить отладочную информацию
objcopy --strip-debug a.out
```

**readelf — детальный анализ ELF:**
```bash
readelf -h a.out      # заголовок
readelf -l a.out      # программные заголовки (сегменты)
readelf -S a.out      # секционные заголовки
readelf -s a.out      # таблица символов
readelf -r a.out      # релокации
readelf -d a.out      # динамическая секция
```

## 2.8 Libelf
- Библиотека для работы с ELF-файлами
- Прилагается [книжка](https://atakua.org/old-wp/wp-content/uploads/2015/03/libelf-by-example-20100112.pdf)

![[libelf-by-example-20100112.pdf]]

## Extended information
- [Article 1](https://cs4157.github.io/www/2024-1/lect/15-elf-intro.html)
- [Article 2](https://ics.uci.edu/~aburtsev/238P/hw/hw3-elf/hw3-elf.html)
- [Article 3](https://gist.github.com/x0nu11byt3/bcb35c3de461e5fb66173071a2379779)

![[elf_layout_1.png|700]]

# 3. Линковка (Linking)
- Вообще, есть статическая и динамическая линковки. 
## 3.1 Статическая и динамическая линковка
- **Статическая линковка** происходит на этапе линковки (static -> до запуска). Все сущности статической библиотеки включаются в исполняемый файл.
- **Динамическая линковка** происходит во время выполнения. В исполняемый файл записываются лишь адреса и указания к подключению библиотеки.

Поэтому определения функций, ответственных за ввод/вывод в консоль, в `a.out` по-прежнему отсутствуют: программа обращается к стандартной библиотеке прямо во время выполнения (Runtime), которая используется одновременно несколькими программами.
- То есть `a.out`, как и большинство программ на C++, не самодостаточны

Благодаря динамической линковке используется меньше ресурсов оперативной памяти.

## 3.2 Разделяемые библиотеки (Shared Objects)
- Динамическая библиотека (`.so`)
	- В Windows - DLL (Dynamic Link Library)

**Просмотр зависимостей разделяемых библиотек:**
- Print shared object dependencies
- Очень полезная тулза
```bash
ldd a.out
```

## 3.3 Статическая сборка
Статическая сборка создает самодостаточный файл, не зависящий от системных библиотек.

**Компиляция статически:**
```bash
g++ -static hello.cpp
```

Теперь программа не зависит от внешних библиотек (чек через `ldd a.out`), но размер значительно увеличивается (около 16Кб). Динамический линковщик не требуется, так как весь код включен в исполняемый файл. ==TODO== правда ли не требуется динамический линковщик?

**Проблема:** при запуске скомпилированного на одной системе статического бинарника на другой системе могут возникнуть проблемы из-за различий в версиях библиотек, интерфейсах системных вызовов и т.д. Статический бинарник будет работать только там, где совпадают архитектура и интерфейс системных вызовов.

## 3.4 Создаем библиотеку
Рассмотрим код

**mylib.cpp:**
```cpp
int multiply(int a, int b) {
    return a * b;
}
```

**hello.cpp:**
```cpp
// `multiply` как бы есть, но он не зарезолвен
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

**Создание разделяемой библиотеки:**
```bash
g++ -shared -fPIC mylib.cpp -o mylib.so
```
- Это либа, по `ldd` видно, что там есть раздел `.dynsym`, где перечислены функции, которые публично видны (но из индексируемого там только `multiply`)
- ==TODO== обязательно ли -fPIC?

**Компиляция с библиотекой:**
- Теперь скомпилируем с библиотекой

```bash
g++ -lmylib hello.cpp
```

- Вообще все библиотеки должны начинаться с `lib` (EX: `libc.so`, `libmylib.so`)
- Поэтому нужно переименовать на `libmylib.so` и перенести в `/usr/lib` (локально он её не найдет) (или дописать `-L.`, типа libraries in `.` (there!))
```bash
g++ hello.cpp -L. -lmylib -o hello
```

**Проблемы поиска библиотеки:**
Скомпилируем с вариантом, где либа находится в той же папке, посмотрим в `ldd`.
- Там `libmylib.so => not found`
Мы также не сможем запустить. Проблема ясна: после компиляции программа может не найти библиотеку при запуске.
- Фактически, дело обстоит так: при компиляции мы попросили компилятор скомпилировать прогу вместе с `libmylib.so`. Однако компилятор не вписал в программу путь, где эту либу при запуске искать

Решения:
1. **Переменная окружения `LD_LIBRARY_PATH`:**
   ```bash
   LD_LIBRARY_PATH=. ./a.out
   ```

2. **Изменение rpath в исполняемом файле:**
	- Самый адекватный вариант: явно указать программе, где искать её зависимость
	- При линковке дописать `rpath=dir`. У `g++` флага `-rpath` нет, `-Wl` говорит g++, что флаг надо передать линковщику. Он запишет адрес прямо в ELF-файл программы
   ```bash
   g++ -Wl,-rpath=. hello.cpp -L. -lmylib -o a.out
   ```

4. **Копирование библиотеки в стандартный каталог:**
   ```bash
   sudo cp mylib.so /usr/lib/
   ```

5. **Использование `LD_PRELOAD`:**
	- Это переменная окружения
   ```bash
   LD_PRELOAD=./mylib.so ./hello
   ```

## 3.5 Динамический линковщик (LD)
- Стандартный линуксовский линковщик

**Попробуем слинковать зависимости вручную:**
```bash
ld hello.o /lib/x86_64-linux-gnu/libstdc++.so.6 /lib/x86_64-linux-gnu/libc.so.6
```

`ld` выведет множество warning'ов о невозможности найти символы для `_start` и т.д. Нужны дополнительные файлы инициализации.

**Необходимые компоненты для ручной линковки:**
- `/usr/lib/gcc/x86_64-linux-gnu/12/crtbeginS.o` и `crtendS.o` (C RunTime)
- `crt1.o`, `crti.o`, `crtn.o`
- Динамический линковщик (`ld.so`)

**Почему `crtbeginS.o` / `crtendS.o` имеют суффикс `S`?**
`S` означает **Shared** (для позиционно-независимого кода, PIC). Эти файлы содержат:
- Код инициализации/финализации глобальных объектов C++ (конструкторы/деструкторы)
- Обрамление для `.init_array` и `.fini_array`

Для статической линковки используются `crtbeginT.o` и `crtendT.o` (`T` — static).

## 3.6 Динамический линковщик (`ld.so`) vs статический (`ld`)
- **`ld`** — статический линковщик, вызывается на этапе сборки.
- **`ld.so`** — динамический линковщик (интерпретатор), загружается при запуске программы. Он создает нам окружение: все библиотеки, стек и т.п.
	- Когда мы запускаем `./a.out`, сначала запускается динамический линкировщик, который затем отдает управление `a.out`

**Процесс запуска программы:**
1. Ядро загружает программу и интерпретатор (из `PT_INTERP`)
2. Управление передаётся `ld.so`
3. `ld.so` загружает зависимости (рекурсивно)
4. Выполняет релокации
5. Передаёт управление в `_start` (из `crt1.o`)

## 3.7 Сравнение через `ldd`
Сравним зависимости нормально скомпилированного файла и нашего:
- Нормальный: есть `/lib64/ld-linux-x86_64.so`
- Наш: есть `/lib/ld64.so`

## 3.8 Проблема с `_start` и `main`

Теперь слинкуем программу еще и с `-I /lib64/ld-linux-x86_64.so`. Будет также жаловаться на `_start`, зато программа заработает: но в самом конце будет Segmentation Fault.
- Здесь будет вызываться не `_start`, а сразу `main()`
- `return 0;` не знает, куда идти: в стеке адрес возврата отсутствует
- Хотя может это потому, что `std::cin` - глобальная переменная, которую надо сынициализировать до `main`'а

**Порядок запуска программы:**
```
_start (из crt1.o)
  → __libc_start_main (инициализация libc)
    → конструкторы глобальных объектов C++ (через .init_array)
      → main()
    → деструкторы (через .fini_array)
  → exit()
```

# 4. Символы и секции

- `.text` - текст программы (испольняемый код)
## 4.1 Секции `.data`, `.bss`, `.rodata`
- **`.rodata`** — Read Only Data: константы (например, строковые литералы `const char*`)
- **`.data`** — Инициализированные статические данные (данные для инициализации которых заготовлены и лежат в коде программы)
- **`.bss`** — Неинициализированные статические данные (заполняются нулями при загрузке)

**Пример:**
```cpp
int a;          // .bss
int b = 0;      // .bss (оптимизация)
int c = 42;     // .data
const char* s = "hello";  // указатель в .data, строка в .rodata
```

## 4.2 Таблицы символов
- **`.symtab`** — таблица символов (для статической линковки)
- **`.dynsym`** — таблица динамических символов (для динамической линковки)

**Символ (с точки зрения линковщика)** — сущность, имеющая адрес (функция, переменная). Линковщик работает с символами и ТОЛЬКО с ними.

**Привязка (Binding):**
- **LOCAL** — виден в пределах единицы трансляции (например, `static int x = 5;` в Global scope)
- **GLOBAL** — виден всем ("торчат" из файла, те видны всем)
- **WEAK** — может быть переопределен. Если линковщик видит несколько определений одной сущности, он выберет сильный символ, если есть, иначе — слабый. Так создается возможность переопределить те же `new`, `delete`, ...

**Пример weak-символа:**
```cpp
// mylib.cpp:
int __attribute__((weak)) a = 3;  // аттрибут для линковщика

// main.cpp:
int a = 4;
```

**Видимость (Visibility):**
- **DEFAULT** — виден везде
- **HIDDEN** — виден только внутри DSO
- **PROTECTED** — виден везде, но переопределяем только внутри DSO

**Примеры:**
```cpp
// Примеры visibility:
void default_func() {}      // Виден везде (exported)
__attribute__((visibility("hidden"))) void hidden_func() {}  
// Виден только внутри DSO (динамически разделяемого объекта)
__attribute__((visibility("protected"))) void protected_func() {}  
// Виден везде, но переопределяем только внутри DSO

// Практическое применение:
// hidden ускоряет загрузку (меньше символов для разрешения)
// уменьшает вероятность конфликтов символов
```

```cpp
// lib.cpp
#define EXPORT __attribute__((visibility("default")))

EXPORT void api() {}      // виден снаружи
void internal() {}        // скрыт (с -fvisibility=hidden)

// Компиляция:
g++ -fPIC -shared -fvisibility=hidden lib.cpp -o libfoo.so
```

==TODO== more info

## 4.3 Манглирование имен (Name Mangling)
C++ кодирует имена функций с учетом типов параметров, области видимости и т.д.

**Деманглирование:**
```bash
c++filt _Z4funcv  # выведет "func()"
nm -C a.out       # деманглированные имена
```

**Удаление символов:**
```bash
strip a.out  # удаляет символы и отладочную информацию
```

# 5. Отладка и Core Dump
- Четверный вид ELF-файла
## 5.1 Сборка с отладочной информацией
```bash
g++ -g hello.cpp -o hello_debug
```
Размер значительно возрастет: добавятся debug-символы.

## 5.2 Использование GDB
```bash
gdb a.out

(gdb) break main          # точка останова
(gdb) run                 # запуск
(gdb) next                # следующая строка
(gdb) step                # с заходом в функцию
(gdb) print variable      # значение переменной
(gdb) backtrace           # стек вызовов
(gdb) watch variable      # отслеживание изменения переменной
(gdb) continue            # продолжение выполнения
(gdb) quit                # выход
```

## 5.3 Core Dump
При падении программы с сообщением "Segmentation fault (Core dumped)" создается файл core dump — снимок памяти процесса на момент падения.
- `/var/lib/apport/coredump`

==TODO== научиться смотреть Core Dump файлы. Будет на экзе!

**Настройка создания core-файлов:**
```bash
ulimit -c unlimited  # разрешить core-файлы неограниченного размера
echo '/tmp/core-%e-%p-%t' | sudo tee /proc/sys/kernel/core_pattern
```

**Анализ core dump:**
- Core dump-файл можно читать с помощью `gdb`
```bash
gdb -c core_dump        # выведет без символов

# а можно указать программу
gdb -c core_dump a.out
# В gdb:
(gdb) bt full           # стек вызовов с переменными
(gdb) info registers    # регистры
(gdb) x/20i $pc-20      # дизассемблирование кода вокруг падения
```

## 6. Продвинутые темы

### 6.1 LTO (Link Time Optimization)
Оптимизации на этапе линковки:
```bash
g++ -flto -O2 file1.cpp file2.cpp
```
Компилятор сохраняет промежуточное представление (GIMPLE) в объектные файлы, линковщик оптимизирует всю программу целиком.

### 6.2 Sanitizers
Инструменты для обнаружения ошибок:
```bash
# AddressSanitizer (ASan)
g++ -fsanitize=address -g program.cpp

# UndefinedBehaviorSanitizer (UBSan)
g++ -fsanitize=undefined -g program.cpp

# ThreadSanitizer (TSan)
g++ -fsanitize=thread -g program.cpp
```

### 6.3 Скрипты линковщика (Linker Scripts)
Контролируют расположение секций в памяти:
```
SECTIONS {
  . = 0x10000;
  .text : { *(.text) }
  . = 0x8000000;
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```
Использование: `ld -T script.ld file.o -o output`

### 6.4 Позиционно-независимый код (PIC/PIE)
```bash
# Компиляция shared library
g++ -fPIC -shared lib.cpp -o lib.so

# PIE (Position Independent Executable) — для ASLR
g++ -pie -fPIE main.cpp -o main_pie
```

### 6.5 ltrace — отслеживание библиотечных вызовов
Есть полезная тулза `ltace`, которая регулирует все библиотечные вызовы
- Пишет в `stderr`, что логично (вообще все логи выводятся в `stderr`, отдельно от основного вывода)

- С флагом `-C` выведет неманглированные имена
```bash
ltrace -C ./a.out 2>2.txt  # неманглированные имена
```
- По факту, подменяет стандартный динамический линковщик на тот, что логирует дополнительно вызовы
