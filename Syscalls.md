# Overview

- Функция, которая является точкой входа с взаимодействием с ОС.
- Есть какой-то интерфейс между библиотеками и операционными системами.

- Предоставляется набор функций

```bash
man syscalls
```

- На самом деле, в большинстве своем работа с syscall'ами осуществляется посредством вызовов функций-оберток (wrappers), чтобы можно было удобно передавать аргумента, не писать явно `вызвать syscall с id=[#] с аргументами [C-style or worth]`

## strace
- Посмотреть, какие syscall'ы делает программа

```bash
strace a.out
```

- Можно мониторить уже запущенные процессы
```bash
strace -p {PID}
```
- [[Programs#strace]]

# IO syscalls

## write
- Получим мануал как syscall'у write (он из главы 2 (см `man man`))
```bash
man 2 write
```
- [[Programs#man]]

```c
#include <unistd.h>

ssize_t write(int fd, const void buf[.count], size_t count);
```
- `fd` - файловый дескриптор - не обязательно файл
	- `stdin`, `stdout`, `stderr` - это файловые дескрипторы с номерами 0, 1, 2 соответственно (поэтому мы направляем так: `1>test.txt`) ==TODO==

```cpp
#inclide <unistd.h>

int main() {
	write(1, "Hello world!", 10);
}
```

- Example 2
```cpp
#inclide <unistd.h>

int main() {
	return write(3, "Hello world!", 10);
}
```

```bash
./a.out
echo $?
# 255 (она же -1)
```

## `errno`

- Чтобы узнать статус ошибки syscall'а
```cpp
#include <errno.h>
#include <iostream>

int main() {
	write(3, "Hello world!", 10);
	std::cout << errno;  // Да, это глобальная переменная
}
```
- Выведет 9 (~Bad file name)

- Можно обрабатывать
```cpp
if (errno == EBADF) {  // global constants
	std::cout << "Bad file descriptor";
	return 1;
}
```

- Возможные ошибки различных syscall'ов описаны в `man`'е

==TODO== man ulimit

- Вообще правильнее писать через цикл: `write` может записать не с первой попытки
```cpp
#include <unistd.h>

int main() {
	const char* str = "Hello world!";
	int len = 10;
	int count = len;
	while (count > 0) {
		int res = write(1, str + len - count, count);
		if (res < 0) {
			return 1;
		}
		count -= res;
	}
	
	return 0;
}
```


## Буфферы

```cpp
#include <iostream>

int main() {
	for (int i = 0; i < 100'000; ++i) {
		std::cout << i << ' ';
	}
}
```

```bash
strace a.out
```

- Увидим, что `write` вызывается для пачки чисел и все строки длиной 1024 (1Кб)
- Все `stream`'ы буфферизируются (`printf` тоже)

```cpp
std::cout.flush();  // Принудительное проталкивание
```

## read
- Читает `count` байт из файлового дескриптора

## open

```bash
man open    # C-шная функция (fopen, freopen)
man 2 open  # Сисколы (open)
```

```c
#include <fcntl.h>

int open(const char *pathname, int flags, ...
         /* mode_t mode */ );

// mode: chmod и прочее - см. manual
```

```cpp
#include <fcntl.h>
#include <unistd.h>

int main() {
	int fd = open("output.txt", O_WRONLY | O_CREAT, S_IRUSR | S_IWUSR);
	
	const char* str = "Hello world!";
	int len = 10;
	int count = len;
	while (count > 0) {
		int res = write(fd, str + len - count, count);
		if (res < 0) {
			return 1;
		}
		count -= res;
	}
	
	close(fd);
	return 0;
}
```
- Важно закрыть файл. Конечно, в конце ОС автоматически все закроет (как и очисит всю динамическую память).

- Если файл уже был создан, то содержимое будет перетираться (полностью не сотреться, как с `std::ifstream`, а будет заменятся по мере продвижения каретки записи)

- Посмотрим через `strace` - увидим, что вернул `fd=3`, и что вызвался syscall `openat`

## openat
- `openat` позволяет указать относительный адрес. То есть вначале передается файловый дескриптор директории, относительно которой считается `pathname`

## lseek
- Сдвинуть каретку
==TODO== manual

```cpp
#include <fcntl.h>
#include <unistd.h>

int main() {
	int fd = open("output.txt", O_WRONLY | O_CREAT, S_IRUSR | S_IWUSR);
	
	const char* str = "Hello world!";
	int len = 10;
	int count = len;
	while (count > 0) {
		int res = write(fd, str + len - count, count);
		if (res < 0) {
			return 1;
		}
		count -= res;
	}
	
	int off = lseek(fd, 5, SEEK_SET);
	write(fd, str, 10);

	close(fd);
	return 0;
}
```
- Выведет `HelloHello world!`

- У него есть интересное свойство
- Если вызвать `lseek` и сдвинуть каретку за пределы дескриптора, то будет "дырка" (hole) из нулей (`\0`). Данные можно писать, но в файле будет про
- Если вызвать `cat`, то "дыры" он просто не выведет, и прочитает файл как будто бы непрерывно, но минуя нули.
- `ls` покажет расстояние от первого до последнего символа - это неправильный размер
- Настоящий размер - размер без "дыров" ==TODO== is this true??

- То есть в линуксе файлом может быть последовательность непрерывных отрезков, разделенные нулями.


## chmod
- Есть команда `chmod`
- Есть syscall `chmod`

## chown
- Сменить владельца
]
## opendir
- Это пока не syscall - это C'шная функция
	- Syscall - `detdents64`, но он ОЧЕНЬ неудобный, так что будем использовать `opendir`
- Возвращает `DIR*`

## readdir
- Приминает `DIR*`, возвращает `dirent` (directory entry)
- Работает итеративно

```cpp
DIR* dir = opendir(path);

struct dirent* entry;

while ((entry = readdir(dir)) != NULL) {
	printf("%s\n", entry->d_name)
}

closedir(dir);
```

## О правах на директорию
- В линуксе все - это файлы
- Даже директории - это файлы
	- В `ls -l` они помечаются сначала флагом `d` (обычные файлы - регулярные - обозначаются прочерком)

- Даже если у пользователя нет прав на чтение файла, он все равно может узнать про этот файл (например, `ls -l` покажет инфу о файле). Но содержимое читать не получится
- Есть также права на исполнение

- У директории также есть права на чтение и выполнение
	- Чтение - посмотреть содержимое
	- Исполнение - открытие директории (`cd`) и первичный доступ к файлам в ней
		- Можно уметь исполнять, но нельзя читать. Как бы информация о файлах в директории не доступна, а сами файлы и ее информация доступна
		- Если исполнять нельзя, то файлы доступны не будут (разве что читать, если дано право на чтение, и то узнать ТОЛЬКО название и тип файла (ни права, ни автора доступно не будет))

```bash
mkdir testdir
cd testdir
touch hahaha
echo "Hahaha!" > hahaha
```

- Уберу у директории `testdir` право на чтение

==TODO== execve

## Просмотр информации о файлах
- Делаем аналог `ls -l`
	- Тип, права, пользователь, когда изменен

```bash
strace ls -l
```

- Видим, что она вызывала `statx` (или `l/fstat`) ==TODO==

- Выведем стату
```cpp
// #include <sys/stat.h>

DIR* dir = opendir(path);

struct dirent* entry;
struct stat* statbuf;

while ((entry = readdir(dir)) != NULL) {
	const char* name = entry->d_name;
	int res = stat(name, statbuf);
	printf("%s %ld\n", name, statbuf->st_size);
}

closedir(dir);
```
- Будет ошибки - надо аллоцировать память под `statbuf`. `stat` сам память не аллоцирует
- `dirent`'ы аллоцируются сами. Разные `dirent`'ы лежат на разных адресах.

## Создание и удаление директории

- `mkdir`
- `rmdir`

- Это тоже syscall'ы

## Что такое файлы

- У файла может иметь несколько равноправных имен
- На низком уровне все работает за счет ID'шников

- Link - привязать имя к файлу
```bash
echo "Hahaha!" > hahaha
ln hahaha pupupu
```
- Это создается hard link

```bash
echo "Pupupu!" >> pupupu
cat hahaha
cat pupupu
```

- Эти файлы неотличимы
	- Оба, если подумать, являются ссылками на один ID'шник

```bash
ls -l
```
- У `hahaha` и `pupupu` будет одно выделяющее свойство: число ссылок = 2.
	- Аналогия с `shared_ptr`

```bash
rm hahaha
```
- Удаляем не файл на диске, а ссылку.
	- Если же у файла ссылок нет, то удаляется он сам
	- Буквально как с `shared_ptr`


- Просто rename файла
```bash
mv pupupu lololo
```

- На самом деле, директория физически - просто список вида: имя + тип + ID'шник

- Как узнать ID файла?
- В структуре `stat` есть `inode`

- Можно так
```bash
ls -i
```

```bash
mkdir testdir
mv ololo testdir
ls -laRi
```

- Появятся скрытые: `.`, `..`. У них по 2 и 3 ссылки соответственно.
	- `.` - ссылка на текущую директорию
	- `..` - ссылка на предыдущую

- Это была hard link
- Сделаем symbolic link
```bash
ln -s data.txt input.txt
```

```bash
ls -l
```
- Это теперь другой файл (с другой inode'ой), другого типа (ни регулярный, ни директория, а link (на Windows'овский манер - ярлык)). Его размер - ровно длина названия файла.

- Лучше всего создавать ссылки с абсолютными путями: если переместить ссылку, она может сослаться на другой файл, если указан относительный путь.

## ln
```bash
strace ln test.txt test2.txt
```
- Syscall `link`

```bash
strace ln -s test2.txt test3.txt
```
- Syscall `symlink`

```bash
rm test.txt
```
- Syscall `unlink`
	- Это приводит к тому, что счетчик уменьшается. И если счетчик = 0, то файл удаляется.
==TODO== прочитать desc мануала unlink


```bash
strace mv
```
- Syscall `rename`

## dup
- Duplicate a file descriptor
- Работает как `2>&1` в bash'е

## lsattr
==TODO== to programs

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

```bash
strace chattr
```
- `ioctl` - мощный инструмент для работы с аттрибутами (и не только) файлов

