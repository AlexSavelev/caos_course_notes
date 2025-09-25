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

