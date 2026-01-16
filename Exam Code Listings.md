# Раздел 1. Линковка и библиотеки. Понятие сисколлов
## 1. Стадии сборки. Как увидеть, из каких этапов состоит сборка программы через g++? Что получается в результате каждого из этапов? Как выполнить каждый из этапов по отдельности? Чем объектный файл отличается от исполняемого? Как дизассемблировать исполняемый файл?

```cpp
#include <iostream>

int main() {
	int x;
	
	std::cin >> x;
	std::cout << x + 5;
}
```

```cpp
%:include <iostream>

int main() <%
	int arr<:5:> = <%1, 2, 3, 4, 5%>;
	std::cout << "Element: " << arr<:2:> << std::endl;
	return 0;
%>
```

```bash
g++ -E -v hello.cpp > hello.e
g++ -S hello.cpp > hello.s
g++ -c hello.s
```

!! Рассказать про инструменты для анализа!
```bash
hexdump -C hello.o
objdump -D hello.o

objdump -t hello.o
readelf -S hello.o
```

## 2. Что такое линковка? Что такое библиотеки (либы)? Покажите на примере простейшей программы, как вручную с помощью ld слинковать объектный файл со стандартной либой C++. Как добиться, чтобы полученный исполняемый файл удалось запустить?

```cpp
#include <iostream>

int main() {
	int x;
	
	std::cin >> x;
	std::cout << x + 5;
}
```

```bash
ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o main.out /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /lib/gcc/x86_64-linux-gnu/14/crtbeginS.o main.o /lib/gcc/x86_64-linux-gnu/14/crtendS.o /usr/lib/x86_64-linux-gnu/crtn.o /lib/x86_64-linux-gnu/libstdc++.so.6 /lib/x86_64-linux-gnu/libc.so.6
```

Извините, контента здесь мало.

## 3. Что такое статическая и динамическая линковка, статические и динамические либы? Как принудительно сделать статическую линковку вместо динамической и зачем это бывает нужно? Что делает команда ldd и как она работает? Как собрать программу с либой C++, расположенной по нестандартному адресу? Как создать свою динамическую либу и собрать программу с ее использованием? Для чего нужны переменные LD_PRELOAD, LD_LIBRARY_PATH? Что такое rpath и как его использовать? Как посмотреть, какие вызовы библиотечных функций делает данная программа в ходе выполнения?

Определения. Почему в скомпиленном файле не будет cout:
```cpp
#include <iostream>

int main() {
	int x;
	
	std::cin >> x;
	std::cout << x + 5;
}
```

Определение дин.либы (shared libs). Сказать, что есть общий термин - DSO.

Статическая сборка
```bash
g++ -static hello.cpp
```

```bash
ldd a.out
```
Как работает?

### Создаем либу
Рассмотрим код

**`mylib.cpp`:**
```cpp
int multiply(int a, int b) {
    return a * b;
}
```

**`hello.cpp`:**
```cpp
int multiply(int a, int b);

int main() {
    multiply(3, 4);
}
```

```bash
g++ -shared -fPIC mylib.cpp -o libmylib.so
readelf --dyn-syms libmylib.so
```

```bash
g++ main.cpp -L. -lmylib -o main.out
```

```bash
LD_LIBRARY_PATH=. ./main.out
g++ -Wl,-rpath=. main.cpp -L. -lmylib -o main.out
sudo cp libmylib.so /usr/lib/
LD_PRELOAD=./libmylib.so ./main.out
```

Вызовы библиотечных функций
```bash
ltrace -C ./main.out 2>output.txt
```

## 4. Формат ELF. Какие типы ELF-файлов существуют? Покажите по одному примеру каждого типа. Из каких основных секций состоят ELF-файлы? Что хранится в секциях strtab, shstrtab, interp, dynamic? Какие есть утилиты для чтения содержимого ELF-файлов? Для чего нужна утилита objcopy? Приведите пример использования.
Рассказать про определение, 2 представления: для линковщика и загрузчика

![[elf_layout_3_colored.jpg|500]]

Каждый тип. Примеры. Основные секции (+data/rodata/bss).
```cpp
int a;
int b = 0;
int c = 42;
const char* s = "hello";
```

```bash
cd /var/lib/apport/coredump
```

```bash
readelf -h a.out      # 
readelf -l a.out      # 
readelf -S a.out      # 
readelf -s a.out      # 
readelf -r a.out      # 
readelf -d a.out      # 

objdump -D hello.o         # 
objdump -t hello.o         # 
```

**objcopy:**
```bash
objcopy -O binary -j .text a.out code.bin

objcopy --add-section .mydata=data.bin a.out

objcopy --redefine-sym old=new a.out

objcopy --strip-debug main.out
```


## 5. Что такое символы в терминах линковщика? Что такое манглирование и как (в общих чертах) оно работает? Как по манглированному имени восстановить исходное? Как посмотреть список символов в данном ELF-файле? Что делает команда strip? Какое бывает связывание у символов (global, local и weak)? Какая бывает видимость у символов (default, hidden, protected)?
**Символ (с точки зрения линковщика)** - это? Про symtab и dynsym.

```bash
c++filt _Z4funcv
nm -C a.out
```

Привязка (Binding)

```cpp
// mylib.cpp:
int __attribute__((weak)) a = 3;

// main.cpp:
int a = 4;
```

Видимость (Visibility). Про DSO. 
```cpp
void default_func() {}

__attribute__((visibility("hidden"))) void hidden_func() {}  

__attribute__((visibility("protected"))) void protected_func() {}  
```

```cpp
// lib.cpp
#define EXPORT __attribute__((visibility("default")))

EXPORT void api() {}
void internal() {}

g++ -fPIC -shared -fvisibility=hidden lib.cpp -o libfoo.so
```

## 6. Запуск программы. Что происходит при запуске бинаря до начала функции main, а также после ее окончания (вопрос с открытым ответом, чем подробнее, тем лучше)? Кто и когда вызывает конструкторы и деструкторы глобальных объектов? Как попросить g++ поставить точку старта программы на конкретную функцию? Почему происходит segfault, если точка старта программы выбрана неудачно? Как сделать, чтобы segfault в таком случае не было?

Процесс запуска программы.

```cpp
#include <iostream>

int main() {
	int x;
	
	std::cin >> x;
	std::cout << x + 5;
}
```

```bash
ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o main.out /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /lib/gcc/x86_64-linux-gnu/14/crtbeginS.o main.o /lib/gcc/x86_64-linux-gnu/14/crtendS.o /usr/lib/x86_64-linux-gnu/crtn.o /lib/x86_64-linux-gnu/libstdc++.so.6 /lib/x86_64-linux-gnu/libc.so.6
```

```bash
g++ -e main hello.cpp
```

```cpp
// custom.cpp
#include <cstdlib>
#include <iostream>

extern "C" void my_start() {
  std::cout << "Hello from my_start!" << std::endl;
  exit(0);
}
```

```bash
g++ -nostartfiles custom.cpp -o custom -Wl,-e,my_start

# g++ -nostartfiles custom.cpp -o custom -Wl,-e,my_start -lstdc++
```

Почему SegFault?

```cpp
#include <iostream>
#include <cstdlib>

extern "C" {
    void __libc_init_first(int argc, char** argv, char** envp);
    int __libc_start_main(
        int (*main)(int, char**, char**),
        int argc, char** argv,
        void (*init)(void),
        void (*fini)(void),
        void (*rtld_fini)(void),
        void (*stack_end)
    );

    void _init(void);
    void _fini(void);
}

int my_main(int argc, char** argv, char** envp) {
    std::cout << "Hello from my_main!\n";
    std::cout << "argc = " << argc << std::endl;
    for (int i = 0; i < argc; i++) {
        std::cout << "argv[" << i << "] = " << argv[i] << std::endl;
    }
    return 0;
}

void _init(void) {
}

void _fini(void) {
}

extern "C" void _start() {
    long argc;
    char** argv;
    
    __asm__ volatile (
        "mov (%%rbp), %0\n"
        "lea 8(%%rbp), %1\n"
        : "=r"(argc), "=r"(argv)
        :
        : "memory"
    );
    
    char** envp = argv + argc + 1;
    
    __libc_init_first(argc, argv, envp);
    
    exit(__libc_start_main(
        my_main,
        argc, argv,
        _init,     // 
        _fini,     // 
        0,         // 
        0          // 
    ));
    
    __builtin_unreachable();
}
```

```bash
g++ -nostartfiles main.cpp -o main.out
```

## 7. Дебаг. Что такое сборка в дебаг-режиме и в чем ее отличие от обычной? Как с помощью gdb отлаживать программу: поставить breakpoint на строчку кода или на функцию, делать шаги по строчкам (с заходом в функции и без него), выводить текущие значения переменных? Как посмотреть backtrace от текущего места исполнения? Что означает фраза “core dumped” и как с помощью gdb посмотреть содержимое coredump-файла после того, как программа упала?

```cpp
#include <iostream>

int main() {
	std::cout << "Good" << std::endl;
	int *ptr = nullptr;
	*ptr = 4;
	std::cout << "Bad" << std::endl;
	
	return 0;
}
```

```bash
cd /var/lib/apport/coredump
```

Рассказать про базу: b (`b file:line`), next, step, continue, info locals, info registers, bt, frame 0, q

### 8. Что такое системный вызов (сисколл)? Как посмотреть в режиме реального времени, какие сисколлы происходит во время работы какой-нибудь программы? Как посмотреть мануал для данного сисколла? Покажите использование сисколлов на примере read и write. Как использовать возвращаемые значения этих сисколлов? Как проверить, успешно ли был выполнен сисколл, а в случае неудачи узнать, какая ошибка произошла?

```bash
strace -p 1234 -e write,read
```

```cpp
#include <unistd.h>

int main() {
    const char* str = "Hello world!";
    int len = 12;
    int count = len;
    while (count > 0) {
        int res = write(1, str + len - count, count);
        if (res < 0) {
            return 1;
        }
        count -= res;
    }
}
```

```cpp
#include <errno.h>
#include <stdio.h>
#include <string.h>

#include <unistd.h>

int main() {
    write(3, "Hello world!", 10);
    printf("Error code: %d\n", errno);
    printf("Error message: %s\n", strerror(errno));
}
```


# Раздел 2. Файлы и файловые системы
## 9. Что такое файловый дескриптор? Расскажите про сисколлы open, close и lseek для работы с файлами. Реализуйте программу cp с помощью данных сисколлов, а также read и write. Что произойдет, если сделать lseek на позицию больше чем размер файла и записать туда что-либо?

Что такое: Открытое файловое описание, файловый дейскриптор.

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

А если открыть 2 раза?

Что такое race condition.
```cpp
int fd = open("./etc/config", O_RDONLY);
```
Что такое `openat`.

`lseek`.

```cpp
#include <fcntl.h>
#include <unistd.h>

int main() {
	int fd = open("output.txt", O_WRONLY | O_CREAT, S_IRUSR | S_IWUSR);
	
	const char* str = "Hello world!";
    write(fd, str, 10);
	
	int off = lseek(fd, 5, SEEK_SET);
	write(fd, str, 10);

	close(fd);
	return 0;
}
```

Про sparse files.
```bash
dd if=/dev/zero of=sparse bs=1 count=0 seek=1G
ls -lh sparse
du -h sparse
```

`copy`
```cpp
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

#define BUFFER_SIZE 4096

int main(int argc, char *argv[]) {
    if (argc != 3) {
        const char *error_msg = "Usage: cp source_file destination_file\n";
        write(2, error_msg, 36);
        return 1;
    }

    int src_fd = open(argv[1], O_RDONLY);
    if (src_fd < 0) {
        const char *error_msg = "Error opening source file\n";
        write(2, error_msg, 27);
        return 1;
    }

    int dst_fd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (dst_fd < 0) {
        const char *error_msg = "Error creating destination file\n";
        write(2, error_msg, 33);
        close(src_fd);
        return 1;
    }

    char buffer[BUFFER_SIZE];
    ssize_t bytes_read, bytes_written;

    while ((bytes_read = read(src_fd, buffer, BUFFER_SIZE)) > 0) {
        bytes_written = write(dst_fd, buffer, bytes_read);
        if (bytes_written != bytes_read) {
            const char *error_msg = "Write error\n";
            write(2, error_msg, 12);
            close(src_fd);
            close(dst_fd);
            return 1;
        }
    }

    if (bytes_read < 0) {
        const char *error_msg = "Read error\n";
        write(2, error_msg, 11);
    }

    close(src_fd);
    close(dst_fd);
    return 0;
}
```

## 10. Перенаправление ввода-вывода. Как в терминале направить вывод команды в файл с перезаписью файла? А без перезаписи, в режиме добавления в файл? Как перенаправить вывод одного из потоков (cout или cerr) в файл или в другой поток? Как подавить вывод какого-то из потоков? Что делает команда tee? Что делают сисколлы dup и dup2? Реализуйте программу tee.

Все примеры.

```bash
ls -l | tee log.txt
```

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open");
        return 1;
    }
	
    char buffer[4096];
    ssize_t bytes;
    while ((bytes = read(0, buffer, sizeof(buffer))) > 0) {
        if (write(1, buffer, bytes) != bytes) {
            perror("write to stdout");
            return 1;
        }
        if (write(fd, buffer, bytes) != bytes) {
            perror("write to file");
            return 1;
        }
    }
	
    if (bytes == -1) {
        perror("read");
        return 1;
    }
	
    close(fd);
    return 0;
}
```

Дублирование ФД. `dup`
Рассказать про открытое файловое описание.

```cpp
#include <unistd.h>
int dup(int oldfd);
int dup2(int oldfd, int newfd);
int dup3(int oldfd, int newfd, int flags);
```

```cpp
int fd1 = open("file.txt", O_RDONLY);
int fd2 = dup(fd1);
read(fd1, buf, 10);
```

## 11. Файловые системы. Что такое файловая система? Какие виды файлов существуют в Linux (обычные файлы, директории, …)? Покажите примеры каждого вида файлов. Что из себя представляют директории с точки зрения файловой системы? Что такое inode и как узнать inode для файлов в данной директории? Что такое виртуальная файловая система? Покажите примеры файлов, которым не соответствует никакое дисковое пространство. Что такое swapfile?

Определение ФС. Все виды файлов + примеры.
Inode.
swapfile. Определение
```bash
dd if=/dev/zero of=/swapfile bs=1M count=1024
mkswap /swapfile
swapon /swapfile
```

## 12. Как пользоваться функциями opendir, readdir? Как пользоваться сисколлами stat, fstat, lstat, fstatat? Как на Си реализовать программу ls с помощью всего этого?

`opendir`, `readdir`, `closedir`:
```cpp
#include <dirent.h>
#include <stdio.h>

int main() {
    DIR* dir = opendir(".");
    if (!dir) {
        perror("opendir");
        return 1;
    }
    
    struct dirent* entry;
    while ((entry = readdir(dir)) != NULL) {
        printf("%s\n", entry->d_name);
    }
    
    closedir(dir);
    return 0;
}
```
Сделать замечание про `struct dirent`.

`stat`
```bash
strace ls -l
```

```cpp
#include <sys/stat.h>
#include <dirent.h>
#include <stdio.h>

int main() {
    DIR* dir = opendir(".");
    struct dirent* entry;
    struct stat statbuf;
    
    while ((entry = readdir(dir)) != NULL) {
        const char* name = entry->d_name;
        int res = stat(name, &statbuf);
        if (res == 0) {
            printf("%s %ld bytes\n", name, statbuf.st_size);
        }
    }
    
    closedir(dir);
    return 0;
}
```

Современная альтернатива: `statx`
```cpp
#define _GNU_SOURCE
#include <sys/stat.h>
#include <fcntl.h> // for AT_FDCWD
#include <stdio.h>  

int main() {
    struct statx stx;
    if (statx(AT_FDCWD, "file.txt", AT_SYMLINK_NOFOLLOW, STATX_ALL, &stx) == 0) {
        printf("Size: %lu\n", stx.stx_size);
        printf("Birth time: %lld.%u\n", stx.stx_btime.tv_sec, stx.stx_btime.tv_nsec);
    }
    return 0;
}
```

## 13. Как работает и какие сисколлы использует программа mv? Как работает и какие сисколлы использует программа rm? Напишите упрощенную реализацию и того, и другого.

```bash
strace rm test.txt
strace mv old new
```

Что делают `unlink`, `rename`. Слабое описание. `mv` медлу ФС.

`mv`:
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/stat.h>

#define BUFFER_SIZE 4096

void copy_file(const char *src, const char *dst) {
    FILE *source, *dest;
    char buffer[BUFFER_SIZE];
    size_t bytes;

    source = fopen(src, "rb");
    if (!source) {
        perror("fopen source");
        exit(EXIT_FAILURE);
    }

    dest = fopen(dst, "wb");
    if (!dest) {
        perror("fopen dest");
        fclose(source);
        exit(EXIT_FAILURE);
    }

    while ((bytes = fread(buffer, 1, BUFFER_SIZE, source)) > 0) {
        fwrite(buffer, 1, bytes, dest);
    }

    fclose(source);
    fclose(dest);
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <source> <destination>\n", argv[0]);
        return 1;
    }

    if (rename(argv[1], argv[2]) == 0) {
        return 0;
    }

    if (errno == EXDEV) {
        copy_file(argv[1], argv[2]);
        if (unlink(argv[1]) != 0) {
            perror("unlink");
            return 1;
        }
        return 0;
    }

    perror("rename");
    return 1;
}
```

`rm`:
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/stat.h>
#include <dirent.h>

void remove_recursive(const char *path) {
    struct stat statbuf;
    if (lstat(path, &statbuf) != 0) {
        perror("lstat");
        return;
    }

    if (S_ISDIR(statbuf.st_mode)) {
        DIR *dir = opendir(path);
        if (!dir) {
            perror("opendir");
            return;
        }

        struct dirent *entry;
        while ((entry = readdir(dir)) != NULL) {
            if (strcmp(entry->d_name, ".") == 0 || 
                strcmp(entry->d_name, "..") == 0) {
                continue;
            }

            char subpath[1024];
            snprintf(subpath, sizeof(subpath), "%s/%s", path, entry->d_name);
            remove_recursive(subpath);
        }
        closedir(dir);

        if (rmdir(path) != 0) {
            perror("rmdir");
        }
    } else {
        if (unlink(path) != 0) {
            perror("unlink");
        }
    }
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s [-r] <file1> [file2 ...]\n", argv[0]);
        return 1;
    }

    int recursive = 0;
    int start_idx = 1;

    if (strcmp(argv[1], "-r") == 0) {
        recursive = 1;
        start_idx = 2;
        if (argc < 3) {
            fprintf(stderr, "Usage: %s [-r] <file1> [file2 ...]\n", argv[0]);
            return 1;
        }
    }

    for (int i = start_idx; i < argc; i++) {
        struct stat statbuf;
        
        if (lstat(argv[i], &statbuf) != 0) {
            perror(argv[i]);
            continue;
        }

        if (S_ISDIR(statbuf.st_mode) && recursive) {
            remove_recursive(argv[i]);
        } else if (S_ISDIR(statbuf.st_mode)) {
            fprintf(stderr, "%s: is a directory\n", argv[i]);
        } else {
            if (unlink(argv[i]) != 0) {
                perror(argv[i]);
            }
        }
    }

    return 0;
}
```

## 14. Жесткие и символические ссылки. В чем разница? Как создать жесткую ссылку, символическую ссылку, что они представляют из себя с точки зрения файловой системы? Как работает и какие сисколлы использует программа ln в случае создания жестких ссылок и символических ссылок? Напишите упрощенную реализацию.

Жесткие ссылки
```bash
echo "NiHao!" > file1.txt
ln file1.txt file2.txt
ls -l
```
Что делает? Ограничение.

Символические ссылки
```bash
ln -s file1.txt file1_s.txt
ls -l
```
Что делает? Преймущество по сравнению с жесткими. Размер.

Syscalls.

Реализация.
```cpp
// ln source link (без -s)
link(argv[1], argv[2]);
// ln -s source link
symlink(argv[1], argv[2]);
```

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

void print_help(const char *prog_name) {
    printf("Использование: %s [-s] ЦЕЛЬ ИМЯ_ССЫЛКИ\n", prog_name);
    printf("  -s    создать символическую ссылку (по умолчанию - жесткую)\n");
}

int main(int argc, char *argv[]) {
    int symbolic = 0;
    int opt;

    while ((opt = getopt(argc, argv, "s")) != -1) {
        switch (opt) {
            case 's':
                symbolic = 1;
                break;
            default:
                print_help(argv[0]);
                return 1;
        }
    }
    
    if (optind + 2 != argc) {
        print_help(argv[0]);
        return 1;
    }
    
    const char *target = argv[optind];
    const char *link_name = argv[optind + 1];
    
    if (symbolic) {
        if (symlink(target, link_name) == -1) {
            perror("Ошибка создания символической ссылки");
            return 1;
        }
        printf("Создана символическая ссылка '%s' -> '%s'\n", link_name, target);
    } else {
        if (link(target, link_name) == -1) {
            perror("Ошибка создания жесткой ссылки");
            return 1;
        }
        printf("Создана жесткая ссылка '%s' для файла '%s'\n", link_name, target);
    }
    
    return 0;
}
```

## 15. Права доступа к файлам. Как посмотреть, как изменить права доступа? Почему r и x - это разные права? Как понимать эти права доступа для директорий? Как поменять владельца файла, как поменять группу владельца? Какие сисколлы используются для всего вышеперечисленного? Что такое sticky bit? Что такое suid-бит, что означают права доступа s и S у файла? Покажите примеры файлов, которые ими обладают. Что такое атрибуты файлов, какие они бывают, как посмотреть и как поменять атрибуты файлов?

Посмотреть. Изменить
```bash
echo "Nihao" > output.txt
chown user:group output.txt
chgrp group output.txt
```

Syscall'ы.
```cpp
#include <sys/stat.h>
int chmod(const char *pathname, mode_t mode);

#include <unistd.h>
int chown(const char *pathname, uid_t owner, gid_t group);
```

Про биты.
```bash
mkdir test
chmod +t test
ls -l
```

SUID-бит. Куда ставится?
```bash
chmod u+s /usr/bin/passwd
ls -l /usr/bin/passwd
```
`S`.

`s` для группы. Вторая особенность
```bash
chown :developers /project
chmod g+s /project
ls -ld /project
```

Про расширенные атрибуты файлов.
```bash
sudo chattr +i output.txt  # 
ls -l                      # 
lsattr output.txt          # 
```

```cpp
#include <sys/ioctl.h>
// OR #include <linux/fs.h>

int flags = FS_IMMUTABLE_FL;
ioctl(fd, FS_IOC_SETFLAGS, &flags);
```

## 16. Как из терминала посмотреть, какие файловые дескрипторы сейчас открыты у данного процесса и какие файлы им соответствуют? Как из терминала посмотреть, какие процессы сейчас держат открытым данный файл? Какие команды есть в терминале для этого и как эти команды реализовать на языке Си?

Список открытых процессом файлов.

Упрощенная реализация `lsof`:
```cpp
#include <dirent.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main() {
    DIR *dir = opendir("/proc/self/fd");
    struct dirent *entry;
    
    while ((entry = readdir(dir)) != NULL) {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
            continue;
        
        char path[256];
        snprintf(path, sizeof(path), "/proc/self/fd/%s", entry->d_name);
        
        char target[256];
        int len = readlink(path, target, sizeof(target)-1);
        target[len] = '\0';
        
        printf("%s -> %s\n", entry->d_name, target);
    }
    
    closedir(dir);
    return 0;
}
```

## 17. Блочные и символьные устройства. В чем разница? Приведите примеры того и другого. Как прочитать данные с какого-нибудь символьного устройства, а также отправить данные на устройство? Что такое виртуальные устройства, что такое /dev/null, /dev/random, /dev/zero? Как сделать, чтобы приложение направило свой stdout / stderr на определенный терминал? Где в файловой системе хранится информация о CPU, об оперативной памяти, о жестких дисках / SSD?

```bash
cd /dev
ls -l
```

Про утилиту dd.
```bash
dd if=/dev/zerо оf=/dev/nvme0n1р1
dd if=/dev/random of=random_data.bin bs=1 count=16  # 
```

```cpp
int fd = open("/dev/tty", O_WRONLY);
write(fd, "Hello\n", 6);
```

`/proc`— информация о процессах и ядре в реальном времени.
Что это такое.
Про вторую псевдоФС.
```bash
ls -l /sys/dev/
ls -l /sys/kernel
```

# Раздел 3. Память

## 18. Виртуальная память. Что это такое и зачем нужно? Что такое страничная организация памяти, таблицы страниц, как они устроены и где хранятся? Что такое page fault, в чем отличие minor от major page fault? Что такое TLB cache? Как происходит обращение процессора по адресу к памяти с учетом всего вышеназванного (вопрос с открытым ответом)? В какой ситуации возникает ошибка Segmentation fault и что в этой ситуации происходит на уровне ОС и процессора?

Виртуальная память.
Про MMU. Page Table'ы.
Про maps.

Комментарий, что сейчас используется XXX-уровневая таблица страниц.

TLB. Pipeline обращения к памяти (обязательно упомянуть page fault!).

```bash
sudo perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses -p $(pgrep a.out) sleep 2
```
Рассказать про iTLB, dTLB. Зачем такое разделение?
Пространственная локальность на примере вектора и списка.

Что такое PageFault.
```bash
sudo perf stat -e page-faults -p $(pgrep a.out) sleep 2
```
minor и major.

SegFault'ы. Сказать, что память просто могла быть зарегистрирована, но не выделенна.

## 19. На какие секции делится адресное пространство процесса? В чем разница между секциями .data, .rodata и .bss? Зачем нужны сисколлы brk и sbrk? Покажите использование сисколлов mmap и munmap базовом сценарии. Как посмотреть, как выглядит в данный момент адресное пространство процесса?

Рассказать про 6 секций. data, rodata, bss.
```cpp
int a;
int b = 0;
int c = 42;
const char* s = "hello";
```

```cpp
#include <unistd.h>

int brk(void *addr);
void *sbrk(intptr_t increment);
```

```cpp
std::vector<int> a;
for (int i = 0; i < 100'000'000; ++i) {
	a.push_back(1);
	if ((i & (i - 1)) == 0) {
		std::cout << i << ' ' << a.data() << '\n';
	}
}
```

```cpp
#include <iostream>
#include <sys/mman.h>

int main() {
	getchar();

	char* ptr = (char *)mmap(NULL, 1000, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
	
	std::cout << (void*)ptr << '\n';
	getchar();
	
	for (int i = 0; i < 1000; ++i) {
		*(ptr + i) = 'a';
	}
	
	munmap(ptr, 1000);
	return 0;
}
```

## 20. Можно ли запросить конкретный виртуальный адрес для выделения памяти? Почему при обращении за границу массива segfault происходит не всегда? Покажите пример использования mremap. Как с помощью mmap загрузить файл в оперативную память? Можно ли таким образом поменять файл? В чем разница между MAP_SHARED и MAP_PRIVATE? Зачем нужен сисколл msync?

```cpp
#include <sys/mman.h>

void *mmap(void addr[.length], size_t length, int prot, int flags, int fd, off_t offset);

int munmap(void addr[.length], size_t length);
```

```cpp
#include <iostream>
#include <sys/mman.h>

int main() {
	getchar();

	char* ptr = (char *)mmap(NULL, 1000, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
	
	std::cout << (void*)ptr << '\n';
	getchar();
	
	for (int i = 0; i < 1000; ++i) {
		*(ptr + i) = 'a';
	}
	
	munmap(ptr, 1000);
	return 0;
}
```

```cpp
for (int i = 0; i < 5000; ++i) {
	std::cout << i << std::endl;
	*(ptr + i) = 'a';
}
```

Упомянуть, что все выше - это UB.

```cpp
char* ptr = (char *)mmap((void *)0x60000000, 1000, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
```

`mremap`
```cpp
void* ptr = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
ptr = mremap(ptr, 4096, 8192, MREMAP_MAYMOVE);
```

Про SHARED, PRIVATE, msync.
```cpp
int fd = open("example.txt", O_RDWR);

struct stat sb;
fstat(fd, &sb);

// MAP_SHARED. With MAP_PRIVATE there is no change in file
char* data = (char*)mmap(NULL, sb.st_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

std::cout << (void*)data << '\n';
getchar();

data[0] = 'X';

// msync() flushes changes made to the in-core copy of a file that was mapped into memory using mmap(2) back to the filesystem.
// With‐out use of this call, there is no guarantee that changes are written back before munmap(2) is called.
// See: man 2 msync
msync(data, sb.st_size, MS_SYNC);

munmap(data, sb.st_size);
```

## 21. Какие бывают права доступа к памяти? Зачем нужен сисколл mprotect и как им пользоваться? Покажите, как с помощью mmap и mprotect загрузить код из библиотеки в память на выполнение. Что означает ошибка Illegal intstruction? Покажите пример программы на Си, которая приводит к этой ошибке (естественным путем, т.е. не генерируя эту ошибку из кода напрямую).

```cpp
#include <sys/mman.h>

int mprotect(void addr[.len], size_t len, int prot);
```

```cpp
mprotect(ptr, size, PROT_READ);
```

Смертельный номер:

**`func.c`:**
```c
double sqr(double x) {
	return x * x;
}
```

```bash
g++ -c func.c
readelf -a func.o
objdump -d func.o
```

**`mmap_dynlib.c`:**
```cpp
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    const char *file_name = argv[1];
    double argument = strtod(argv[2], NULL);

    int fd = open(file_name, O_RDONLY);
    struct stat st = {};
    fstat(fd, &st);

    void *addr =
        mmap(NULL, st.st_size, PROT_READ | PROT_EXEC, MAP_PRIVATE, fd, 0);
    double (*func)(double) = (double (*)(double))((char *)addr + 0x40);

    close(fd);
    double result = func(argument);
    printf("func(%f) = %f\n", argument, result);
    munmap(addr, st.st_size);
}
```

```bash
./a.out func.o 54  # func(54.000000) = 2916.000000
```

Рассказать про DEP (Data Execution Prevention).

Про Illegal Instruction.
```cpp
#include <vector>
#include <list>
#include <iostream>
#include <sys/mman.h>
#include <errno.h>

int main() {
	std::vector<char> v;
	for (int i = 0; i < 1000000; ++i) {
		v.push_back(i);
	}
	
	std::cout << "vector starts at " << (int*)&v[0] << std::endl;
	std::cout << mprotect(&v[0] - 16, 10000, PROT_READ|PROT_WRITE |PROT_EXEC) << std::endl;
	getchar();
	
	v[0] = 0x33;
	void (*f)() = (void(*)()) &v[0];
	f();
}
```

Практический способ
```cpp
#include <dlfcn.h>
#include <iostream>

int main() {
    void* lib = dlopen("./libfunc.so", RTLD_LAZY);
    if (!lib) {
        std::cerr << dlerror() << std::endl;
        return 1;
    }
    
    auto sqr = (double(*)(double))dlsym(lib, "sqr");
    if (!sqr) {
        std::cerr << dlerror() << std::endl;
        return 1;
    }
    
    std::cout << sqr(2.0) << std::endl;
    dlclose(lib);
    return 0;
}
```


## 22. Как реализованы функции malloc и free в стандартной библиотеке Си? Расскажите про механизм бакетов, малые и большие бакеты. Как malloc выбирает, какой бакет использовать? Как происходит освобождение бакетов и слияние соседних свободных бакетов? Что такое fastbins? Какие сисколлы использует malloc для работы с памятью?
Не повезло.
Бакеты, структура чанка (glibc), fast bins, слияние.

# Раздел 4. Процессы и треды
## 23. Что такое процесс? Как посмотреть все процессы в системе? Что такое pid, ppid? Как посмотреть дерево процессов? Как посмотреть потребление памяти, потребление CPU каждым из процессов? Что такое uid, euid и cwd данного процесса, как их узнать и как поменять?

Процесс. Что включает в себя.
```bash
ps -eo pid,ppid,comm | head -10
ps -eo pid,comm,%cpu,%mem,rss,vsz --sort=-%cpu
ps -p $$ -o pid,uid,euid,gid,egid,comm
pstree
```

UID, EUID, CWD.
```cpp
#include <unistd.h>
#include <iostream>

int main() {
	std::cout << getuid() << ' ' << geteuid() << '\n';

	while (true) {}
}
```

## 24. Что такое приоритет процесса, какой он бывает, как его узнать и как поменять? Что такое CPU affinity данного процесса, как его узнать и как поменять? Что такое process capabilities в Linux, какие они бывают? Как выдать процессу определенные capabilities, как посмотреть имеющиеся?

htop, PR, NI (nice value).
```cpp
int main() {
	while (true) {}
}
```
```bash
nice -n 15 ./a.out
```

CPU Affinity. 
```bash
man 7 sched
```

```cpp
#include <sched.h>
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(0, &set);
sched_setaffinity(0, sizeof(set), &set);
```

Process Capabilities.
```bash
man 7 capability
sudo setcap cap_net_raw+ep /path/to/program
cat /proc/<PID>/status | grep Cap
```

## 25. Расскажите про сисколлы fork и exec. Какие версии сисколла exec существуют и в чем разница между ними? В чем необычность функций fork и exec, что происходит при их вызове? Покажите пример вызова из программы другой программы, используя fork+exec. Что такое fork-бомба?
```cpp
#include <iostream>
#include <unistd.h>

int main() {
	std::cout << "Hello!" << std::endl;
	
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}

	return 0;
}
```

`getpid` - что это? Особенность.

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int fd = open("example.txt", O_RDWR | O_CREAT);
	
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
		write(fd, "CHILD!", 7);
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
		write(fd, "PARENT!", 7);
	}
	
	close(fd);

	return 0;
}
```

Семейство `exec`. Что обнуляется, а что нет?

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
		
		char* argv[] = {"/usr/bin/ls", ".", "-l", NULL};
		char* envp[] = {NULL};

		execve("/usr/bin/ls", argv, NULL);
		perror("execve failed");
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}
	
	return 0;
}
```

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int fd = open("example.txt", O_RDWR | O_CREAT);

	int pid = fork();
	
	if (pid == 0) {
		dup2(fd, 1);

		char* argv[] = {"/usr/bin/ls", ".", "-l", NULL};
		char* envp[] = {NULL};

		execve("/usr/bin/ls", argv, NULL);
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}

	close(fd);
	return 0;
}
```

Fork-бомба
```bash
:(){ :|:& };:
```

## 26. Какие бывают состояния у процессов? Как в терминале приостановить процесс, как возобновить приостановленный процесс? Как пользоваться командами fg и bg? Что такое процессы-зомби, как они возникают? Как посмотреть, в каком состоянии находится сейчас какой-либо процесс? Что делает сисколл wait, как им пользоваться? Что делает функция sleep в Си?

Про 5 состояний. 

```bash
ps -eo pid,ppid,state,cmd
jobs
fg ./main.out
./a.out &
```

Зомби
```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
		getchar();
	}

	return 0;
}
```

А если родитель умирает раньше ребенка? Рассказать, насколько systemd классный демон, что "собирает урожай".
```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
		sleep(1);
		std::cout << getppid() << std::endl;
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}

	return 0;
}
```


Sleep, Stop
```cpp
#include <iostream>

int main() {
	std::cout << "Hello\n";
	getchar();
	std::cout << "Hello again\n";
	
	return 0;
}
```

Wait.
```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

// Include
#include <sys/wait.h>

int main() {
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';

		char* argv[] = {"/usr/bin/ls", ".", "CRINGE_LALALA", NULL};
		char* envp[] = {NULL};

		execve("/usr/bin/ls", argv, NULL);
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
		
		int res;
		int pid = wait(&res);
		std::cout << "Child exited with status " << res << '\n';
	}

	return 0;
}
```

Sleep. Что делает? Интересный пример.
```cpp
#include <signal.h>
#include <sys/wait.h>
#include <stdio.h>

void child_handler(int sig) {
    int status;
    while (waitpid(-1, &status, WNOHANG) > 0) {
        printf("Child exited\n");
    }
}

int main() {
    signal(SIGCHLD, child_handler);
    if (fork() == 0) {
        // child
        return 0;
    }
    sleep(1);
    return 0;
}
```

## 27. Что такое rlimit для процесса? Как пользоваться функциями getrlimit и setrlimit? Покажите, как из кода программы запросить себе больший размер стека, чем дан изначально. Как установить процессу ограничение на использование памяти и/или процессорного времени? Что произойдет, если эти ограничения будут превышены?

Рассказать про hard limit и soft limit.

```bash
ulimit -a

ulimit -S -t 10
ulimit -H -t 20
```

```cpp
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, const struct rlimit *rlim);

struct rlimit {
    rlim_t rlim_cur;  /* Soft limit */
    rlim_t rlim_max;  /* Hard limit */
};
```

```cpp
#include <sys/resource.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    printf("CPU limit exceeded!\n");
    _exit(1);
}

int main() {
    struct rlimit limit;
    limit.rlim_cur = 1;
    limit.rlim_max = 5;
    setrlimit(RLIMIT_CPU, &limit);
    
    signal(SIGXCPU, handler);
    
    while(1);
    return 0;
}
```

```cpp
#include <sys/resource.h>

struct rlimit rlim;
getrlimit(RLIMIT_NOFILE, &rlim);
rlim.rlim_cur = 1024;
setrlimit(RLIMIT_NOFILE, &rlim);
```

```cpp
#include <stdio.h>
#include <sys/resource.h>
#include <stdlib.h>

int increase_stack_size() {
    struct rlimit rlim;
    
    if (getrlimit(RLIMIT_STACK, &rlim) != 0) {
        perror("getrlimit failed");
        return -1;
    }
    
    rlim.rlim_cur = rlim.rlim_max;
    
    if (setrlimit(RLIMIT_STACK, &rlim) != 0) {
        perror("setrlimit failed");
        return -1;
    }
    
    printf("Новый размер стека: %ld MB\n", 
           (long)rlim.rlim_cur / (1024 * 1024));
    return 0;
}
```

```cpp
if (setrlimit(RLIMIT_AS, &rlim) != 0) {
    perror("Ошибка установки ограничения памяти");
}
if (setrlimit(RLIMIT_CPU, &rlim) != 0) {
    perror("Ошибка установки ограничения процессора");
}
```


## 28. Расскажите про библиотеку seccomp. Покажите на примере, как запретить программе вызывать определенные сисколлы. Как получить ошибку Bad system call (core dumped)?

```bash
sudo apt install libseccomp-dev
man 2 seccomp
```

```cpp
#include <cerrno>
#include <cstdlib>
#include <iostream>
#include <seccomp.h>
#include <sys/prctl.h>
#include <unistd.h>

void setup_seccomp() {
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW); 
	if (ctx == nullptr) {
		perror("seccomp_init");
		exit(1);
	}
	
	seccomp_rule_add(ctx, SCMP_ACT_KILL_PROCESS, SCMP_SYS (execve), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS (fork), 0);
	
	if (seccomp_load(ctx) < 0) {
		perror("seccomp_load");
		exit(1);
	}
	
	seccomp_release(ctx);
}

int main() {
	setup_seccomp();
	
	std::cout << "Process is running with restricted syscalls." << std::endl;
	
	char *args[] = {"/bin/ls", NULL};
	execve(args[0], args, NULL);
	
	while (true) {}	
	return 0;
}
```

```bash
g++ main.cpp -lseccomp -o main.out
```

## 29. Что такое сигналы? Как послать сигнал процессу из терминала, а также из кода программы? Перечислите известные вам стандартные сигналы с объяснением, для чего они применяются. Какова стандартная реакция процессов на каждый из сигналов? Как вручную из терминала вызвать у стороннего процесса segfault? Как из кода послать сигнал самому себе? Как в коде программы заснуть до прихода сигнала?

```cpp
int main() {
	while(true) {}
	
	return 0;	
}
```

```bash
man 7 signal
man 2 kill

kill $(pgrep main.out)
kill -9 $(pgrep main.out)
kill -11 $(pgrep main.out)
kill -2 $(pgrep main.out)
```

Стандартные сигналы. Упомянуть про 2 стопа.
Как послать самому себе?

*Можно рассказать про таблицу прерываний, синхронные и асинхронные сигналы.*

Как заснуть?

```cpp
#include <stdio.h>
#include <unistd.h>
#include <signal.h>

void signal_handler(int sig) {
    printf("Получен сигнал %d! Пробуждаемся...\n", sig);
}

int main() {
    signal(SIGUSR1, signal_handler);

    printf("Процесс PID=%d приостановлен. Отправьте ему 'kill -SIGUSR1 %d'\n", getpid(), getpid());
    pause();

    printf("Программа продолжила работу после pause().\n");
    return 0;
}
```

## 30. Как сделать кастомный обработчик сигналов? Покажите на примере, как из кода программы перехватывать segfault и делать что-то нестандартное при его наступлении. Что, если во время обработки сигнала приходит другой сигнал? Как можно заблокировать получение других сигналов во время обработки сигнала? Что, если сигнал приходит во время выполнения сисколла? Что такое signal-safety и что такое реентрабельная функция?

А какие можно навесить свой обработчик, а на какие нельзя?

```bash
man 2 signal
```

```cpp
#include <cstdio>
#include <signal.h>

void handler(int num) {
	printf("I won't die\n");
}

int main() {
	signal(SIGINT, &handler);

	while(true) {}

	return 0;
}
```

```cpp
#include <cstdio>
#include <signal.h>

void handler(int num) {
	printf("I won't die\n");
	// Потом добавить: *p = 1;
}

int* p = nullptr;

int main() {
	signal(SIGSEGV, &handler);

	*p = 1;

	return 0;
}
```

```cpp
#include <cstdio>
#include <signal.h>

int x = 0;

void handler_fpe(int num) {
	printf("I won't die FPE\n");
}

void handler(int num) {
	printf("I won't die SEGV\n");
	num / x;
}

int* p = nullptr;

int main() {
	signal(SIGFPE, &handler_fpe);
	signal(SIGSEGV, &handler);

	*p = 1;

	return 0;
}
```

Можно рассказать про синхронные и асинхронные сигналы, маску.

```cpp
#include <cstdio>
#include <signal.h>
#include <sys/mman.h>

int* ptr = nullptr;

void handler(int num) {
	printf("I won't die\n");
	mprotect(ptr, 1000, PROT_READ | PROT_WRITE);
}

int main() {
	signal(SIGSEGV, &handler);
	
	ptr = (int*)mmap(0, 1000, PROT_READ, MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
	*ptr = 5;

	return 0;
}
```

```cpp
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

void signal_handler(int signum) {
  printf("На 5 секунд блокирую другие сигналы...\n");
  for (int i = 5; i > 0; i--) {
    printf("%d...\n", i);
    sleep(1);
  }

  printf("Обработка завершена\n");
}

int main() {
  struct sigaction sa;

  sigemptyset(&sa.sa_mask);

  sigaddset(&sa.sa_mask, SIGTERM);
  sigaddset(&sa.sa_mask, SIGUSR1);

  sa.sa_handler = signal_handler;
  sa.sa_flags = 0;

  if (sigaction(SIGINT, &sa, NULL) == -1) {
    perror("sigaction");
    return 1;
  }

  printf("PID: %d\n", getpid());
  printf("  kill -TERM %d\n", getpid());

  while (1) {
    pause();
  }

  return 0;
}
```

Рассказать, что с syscall'ами.

```cpp
sa.sa_flags = SA_RESTART;
sigaction(SIGUSR1, &sa, NULL);
```

Сказать, что все было UB.
```bash
man 7 signal-safety
```

## 31. Что такое pipes? Покажите в коде пример создания pipe и общения между двумя процессами с помощью pipe. В какой ситуации возникает ошибка Broken pipe? Как реализовать аналог оператора | в bash на Си?

Что такое Pipe. 2 типа pipe'ов.

```cpp
#include <unistd.h>
int pipe(int pipefd[2]);
```

```cpp
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>

int main() {
    int fd[2];
    pipe(fd); // fd[0] - чтение, fd[1] - запись
    
    if (fork() == 0) {
        // child: пишет в pipe
        close(fd[0]);
        write(fd[1], "Hello", 6);
        close(fd[1]);
        return 0;
    } else {
        // parent: читает из pipe
        close(fd[1]);
        char buf[100];
        read(fd[0], buf, sizeof(buf));
        printf("Parent got: %s\n", buf);
        close(fd[0]);
        wait(NULL);
    }
    return 0;
}
```

Рассказать еще про Broken Pipe.
```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <signal.h>
#include <errno.h>
#include <string.h>

void sigpipe_handler(int sig) {
    printf("Caught SIGPIPE signal! Broken pipe detected.\n");
}

int main() {
    int pipefd[2];
    char buffer[10];
    
    signal(SIGPIPE, sigpipe_handler);
    
    pipe(pipefd);
    
    if (fork() == 0) {
        close(pipefd[1]);
        sleep(1);
        close(pipefd[0]);
        exit(0);
    }
    
    close(pipefd[0]);
    
    sleep(2);
    
    ssize_t bytes = write(pipefd[1], "hello", 5);
    
    if (bytes == -1) {
        printf("write failed: %s (errno=%d)\n", strerror(errno), errno);
        if (errno == EPIPE) {
            printf("EPIPE error - broken pipe!\n");
        }
    }
    
    close(pipefd[1]);
    wait(NULL);
    return 0;
}
```

Реализация pipeline
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <string.h>

void simple_pipeline(const char *cmd1[], const char *cmd2[]) {
    int pipefd[2];
    pid_t pid1, pid2;
    
    if (pipe(pipefd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }
    
    pid1 = fork();
    if (pid1 == 0) {
        dup2(pipefd[1], STDOUT_FILENO)
        close(pipefd[0]);концы
        close(pipefd[1]);
        
        execvp(cmd1[0], (char *const *)cmd1);
        perror("execvp failed for cmd1");
        exit(EXIT_FAILURE);
    }
    
    pid2 = fork();
    if (pid2 == 0) {
        dup2(pipefd[0], STDIN_FILENO);
        close(pipefd[0]);
        close(pipefd[1]);
        
        execvp(cmd2[0], (char *const *)cmd2);
        perror("execvp failed for cmd2");
        exit(EXIT_FAILURE);
    }
    
    close(pipefd[0]);
    close(pipefd[1]);
    
    waitpid(pid1, NULL, 0);
    waitpid(pid2, NULL, 0);
}

int main() {
    const char *cmd1[] = {"ls", "-l", NULL};
    const char *cmd2[] = {"wc", "-l", NULL};
    simple_pipeline(cmd1, cmd2);
    return 0;
}
```


## 32. Что такое fifo-файлы? Как создать такой файл из терминала, а также программно? Покажите пример общения между двумя процессами с помощью fifo-файла. Что, если несколько процессов пишут в один и тот же fifo? Что, если несколько процессов читают один и тот же fifo?

Что такое Pipe. 2 типа pipe'ов.

```bash
man mknod
mknod myfifo p
```

```cpp
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
```

Пример.
```cpp
// write_to_fifo.cpp

#include <sys/types.h>
#include <sys/stat.h>

#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main() {
	mkfifo("./test_fifo", 0666);
	int fd = open("./test_fifo", O_WRONLY);
	
	char* str = "Hello!";
	write(fd, str, strlen(str));
	
	close(fd);
	return 0;
}
```

```cpp
// read_from_fifo.cpp

#include <sys/types.h>
#include <sys/stat.h>

#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
	int fd = open("./test_fifo", O_RDONLY);
	
	char str[10];
	read(fd, str, 5);
	
	printf("%s", str);
	
	close(fd);
	return 0;
}
```

```bash
echo "Hello" > test_fifo
```

Рассказать про `O_NONBLOCK` - неблокирующий режим. Где хранятся?
Рассказать про существование PIPE_BUF.

Когда несколько пишут?
Когда несколько читают?
Еще есть хаос.

## 33. Что такое разделяемая память? Какие сисколлы существуют для создания и управления разделяемой памятью? Покажите на примере, как устроить общение через разделяемую память между двумя процессами. Как посмотреть, какие участки разделяемой памяти существуют в ОС и кто их создал? Как посмотреть, какие страницы разделяемой памяти сейчас использует данный процесс?

System V Shared Memory. Где хранится технически?

```bash
man 2 shmop
```

```cpp
// shm_read.cpp

#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

#define SHM_SIZE 1000000

int main() {
	int shm_id;
	char *shm_ptr;
	
	/*
	key_t key = ftok("shmfile", 65);
	if (key== -1) {
		perror("ftok");
		exit(1);
	}*/
	
	shm_id = shmget(12222, SHM_SIZE, 0666);
	if (shm_id == -1) {
		perror("shmget");
		exit(1);
	}
	
	shm_ptr = (char*) shmat(shm_id, NULL, 0);
	if (shm_ptr == (char *)-1) {
		perror("shmat");
		exit(1);
	}
	
	printf("Reader: Read from shared memory: %s\n", shm_ptr);
	
	shmdt(shm_ptr);
	
	shmctl(shm_id, IPC_RMID, NULL);
	
	return 0;
}
```

```cpp
// shm_write.cpp

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

#define SHM_SIZE 1000000

int main() {
	int shm_id;
	char *shm_ptr;
	
	shm_id = shmget(12222, SHM_SIZE, IPC_CREAT | 0666);
	if (shm_id == -1) {
		perror("shmget");
		exit(1);
	}
	
	shm_ptr = (char*)shmat(shm_id, NULL, 0);
	if (shm_ptr == (char *)-1) {
		perror("shmat");
		exit(1);
	}
	
	const char *message = "Hello from writer!";
	sprintf(shm_ptr, "%s", message);
	printf("Writer: Wrote to shared memory: %s\n", message);
	
	getchar();
	shmdt(shm_ptr);
	return 0;
}
```

## 34. Что такое потоки выполнения (треды, threads, нити)? Покажите пример создания и использования thread на С++. Покажите пример параллельной обработки из двух тредов каких-либо данных. Что делают методы join и detach? Что происходит, если main завершается, но при этом еще не все треды завершили свою работу?

Классический пример. С ним можно почти на все ответить.
```cpp
#include <iostream>
#include <vector>
#include <thread>

std::vector<int> v;
long long arr[8];

void sum(int index, int from, int to) {
	long long result = 0;
	for (int i = from; i < to; ++i) {
		result += v[i];
	}
	arr[index] = result;
	std::cout << index << ' ' << result << std::endl;
}

int main() {
	v.resize(800'000);
	for (auto& x : v) {
		x = rand();
	}
	
	std::vector<std::thread> vt;
	vt.reserve(8);
	
	for (int i = 0; i < 8; ++i) {
		vt.emplace_back(sum, i, i * 100'000, (i + 1) * 100'000);
	}
	
	for (auto& t : vt) {
		t.join();
	}
	
	long long result = 0;
	for (int i = 0; i < 8; ++i) {
		result += arr[i];
	}
	std::cout << result << std::endl;

	return 0;
}
```

Не забыть сказать про `detach`

## 35. Что такое race condition? Приведите пример, когда возникает UB из-за одновременного изменения одних и тех же данных из разных тредов. Что такое мьютекс? Приведите пример решения проблемы race condition с помощью мьютекса. Что такое deadlock и как он может возникнуть?

```cpp
#include <iostream>
#include <vector>
#include <thread>

std::vector<int> v;

void f() {
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		v.push_back(i);
	}
}

int main() {
	std::thread t(f);

	for (int i = 0; i < 1'000'000; ++i) {
		v.push_back(i);
	}  
	
	t.join();
	
	return 0;
}
```

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>

std::vector<int> v;

std::mutex m;

void f() {
	m.lock();
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		v.push_back(i);
	}
	m.unlock();
}

int main() {
	std::thread t(f);

	m.lock();
	for (int i = 0; i < 1'000'000; ++i) {
		v.push_back(i);
	}
	m.unlock();

	t.join();
	std::cout << v.size();
	
	return 0;
}
```

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>

std::vector<int> v;

std::mutex m;

void f() {
	std::lock_guard<std::mutex> lg(m);
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		v.push_back(i);
	}
}

int main() {
	std::thread t(f);
	
	{
		std::lock_guard<std::mutex> lg2(m);
		for (int i = 0; i < 1'000'000; ++i) {
			v.push_back(i);
		}
	}
	
	t.join();
	std::cout << v.size();
	
	return 0;
}
```

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>

std::vector<int> v;

std::mutex m1;
std::mutex m2;

void f() {
	std::unique_lock<std::mutex> lg(m2);
	std::unique_lock<std::mutex> lg2(m1);
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		v.push_back(i);
	}
}

int main() {
	std::thread t(f);

	std::unique_lock<std::mutex> lg(m1);
	std::unique_lock<std::mutex> lg2(m2);
	for (int i = 0; i < 1'000'000; ++i) {
		v.push_back(i);
	}
	lg.unlock();
	lg2.unlock();
	
	t.join();
	std::cout << v.size();
	
	return 0;
}
```

```cpp
std::unique_lock<std::mutex> lock1(m1, std::defer_lock);
std::unique_lock<std::mutex> lock2(m2, std::defer_lock);
std::lock(lock1, lock2);
```

## 36. Как реализовать std::thread, используя сисколлы? Как пользоваться сисколлом clone и какие у него есть параметры? Покажите, как надо вызывать сисколл clone, чтобы создать полноценный std::thread. Что из себя представляют треды с точки зрения ОС? Что такое тред-группа? Что такое tid, tgid, как их узнать, в чем разница с pid? Как послать сигнал отдельному треду?

```cpp
#include <atomic>
#include <csignal>
#include <cstring>
#include <functional>
#include <iostream>
#include <linux/futex.h>
#include <sched.h>
#include <stdexcept>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <unistd.h>

class Thread {
  pid_t tid;
  const size_t stack_size = 2 * 1024 * 1024;
  void *stack;
  std::atomic<int> *state;
  std::function<void()> *func_ptr;

  static int starter(void *arg) {
    Thread *self = static_cast<Thread *>(arg);
    (*self->func_ptr)();

    self->state->store(1, std::memory_order_release);

    syscall(SYS_futex, self->state, FUTEX_WAKE, 1, nullptr, nullptr, 0);

    return 0;
  }

public:
  template <typename Callable> Thread(Callable &&f) {
    func_ptr = new std::function<void()>(std::forward<Callable>(f));
    state = new std::atomic<int>(0);

    stack = mmap(nullptr, stack_size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);

    if (stack == MAP_FAILED) {
      delete func_ptr;
      delete state;
      throw std::runtime_error("mmap failed");
    }

    int flags = CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND |
                CLONE_THREAD | SIGCHLD;

    tid = clone(starter, static_cast<char *>(stack) + stack_size, flags, this);

    if (tid == -1) {
      munmap(stack, stack_size);
      delete func_ptr;
      delete state;
      throw std::runtime_error("clone failed");
    }
  }

  void join() {
    while (state->load(std::memory_order_acquire) == 0) {
      syscall(SYS_futex, state, FUTEX_WAIT, 0, nullptr, nullptr, 0);
    }

    munmap(stack, stack_size);
    delete func_ptr;
    delete state;

    func_ptr = nullptr;
    state = nullptr;
  }

  ~Thread() {
    if (func_ptr != nullptr) {
      join();
    }
  }
};

int main() {
  const long long N = 100'000'000;

  std::cout << "Параллельное сложение (2 потока)..." << std::endl;

  long long sum1 = 0, sum2 = 0;

  Thread t1([&sum1, N]() {
    for (long long i = 1; i <= N / 2; i++) {
      sum1 += i;
    }
  });

  Thread t2([&sum2, N]() {
    for (long long i = N / 2 + 1; i <= N; i++) {
      sum2 += i;
    }
  });

  t1.join();
  t2.join();

  long long sum_par = sum1 + sum2;
  std::cout << "Результат: " << sum_par << std::endl;

  long long sum_seq = 0;
  for (long long i = 1; i <= N; i++) {
    sum_seq += i;
  }

  std::cout << "Последовательный результат: " << sum_seq << std::endl;
  std::cout << "Совпадают? " << (sum_par == sum_seq ? "Да" : "Нет")
            << std::endl;

  return 0;
}
```

```cpp
#define _GNU_SOURCE
#include <sched.h>
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>

int child_func(void *arg) {
    printf("Child thread: PID=%d, TID=%ld\n", getpid(), syscall(SYS_gettid));
    return 0;
}

int main() {
    void *stack = mmap(NULL, 8192, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
    
    clone(child_func, stack + 8192, CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD, NULL);
    
    sleep(1);
    return 0;
}
```

## 37. Что такое семафоры в Linux, для чего они нужны? Что позволяет делать сисколл futex? Покажите набросок реализации std::mutex с использованием сисколла futex. (Можно использовать std::atomic без объяснения.)

```cpp
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <atomic>
#include <climits>

class FutexMutex {
    std::atomic<int> val;
public:
    FutexMutex() : val(0) {}
    
    void lock() {
        int expected = 0;
        while (!val.compare_exchange_weak(expected, 1, std::memory_order_acquire)) {
            syscall(SYS_futex, &val, FUTEX_WAIT, 1, nullptr, nullptr, 0);
            expected = 0;
        }
    }
    
    void unlock() {
        val.store(0, std::memory_order_release);
        syscall(SYS_futex, &val, FUTEX_WAKE, 1, nullptr, nullptr, 0);
    }
};
```

# Раздел 5. Ассемблер и устройство процессора
## 38. Что такое ассемблер? Какие есть разновидности ассемблера? Что такое регистры? Перечислите основные регистры в архитектуре x86 и их предназначение. Расскажите про основные ассемблерные инструкции и их синтаксис: mov, арифметические инструкции, логические инструкции.

```cpp
int sum(int a, int b) {
	int c = a + b;
	return c;
}
```

```asm
sum(int, int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-20], edi
        mov     DWORD PTR [rbp-24], esi
        mov     edx, DWORD PTR [rbp-20]
        mov     eax, DWORD PTR [rbp-24]
        add     eax, edx
        mov     DWORD PTR [rbp-4], eax
        mov     eax, DWORD PTR [rbp-4]
        pop     rbp
        ret
```

```asm
mov     eax, edi
mov     edx, 4908534051
imul    rax, rdx
shr     rax, 35
```

Регистры и их историческая перспектива.

## 39. Инструкции безусловного и условного перехода в ассемблере. Регистр флагов. Какие есть условные переходы и как (идейно) они работают? Как написать аналоги if, while и for на ассемблере?

```cpp
int sum(int a) {
	if (a > 25) {
		e++;
	}
	return a;
}
```

```asm
	cmp DWORD PTR [rbp-24], 25
	jle .L2
	add QWORD PTR [rbp-24], 1
.L2:
	; ...
	ret
```

```cpp
for (int i = 0; i < a; ++i) {
	d += b;
}
```

```asm
	mov DWORD PTR [rbp-12], 0 ; счётчик i
	jmp .L_check
.L_body:
	; тело цикла
	add DWORD PTR [rbp-8], edx ; d += b
	add DWORD PTR [rbp-12], 1  ; ++i
.L_check:
	mov eax, DWORD PTR [rbp-12]
	cmp eax, DWORD PTR [rbp-24] ; i < a
	jl .L_body
```

```cpp
int k = 0;
while (k == 0) {
	k = 1;
}
```

```asm
	mov DWORD PTR [rbp-4], 0
	jmp .L2
.L3:
	mov DWORD PTR [rbp-4], 1
.L2:
	cmp DWORD PTR [rbp-4], 0
	je .L3
```

```cpp
int foo(int a) {
	if (a > 1) a = 2;
	return a;
}
```

```asm
mov eax, 2
cmp edi, eax
cmovle eax, edi   ; если edi <= 2, то eax = edi
ret
```

## 40. Как использовать в программе на Си ассемблерную функцию из другого файла? Покажите на примере функции проверки числа на простоту. Как с помощью gdb делать отладку ассемблерного кода? Как просматривать текущие значения регистров, как делать пошаговое исполнение ассемблерных инструкций?

```asm
section .text
global is_prime

is_prime:
    cmp     edi, 2
    jl      .not_prime
    je      .prime
    
    mov     ecx, 2
    
.loop:
    cmp     ecx, edi
    jge     .prime
    
    mov     eax, edi
    xor     edx, edx
    div     ecx
    test    edx, edx
    jz      .not_prime
    
    inc     ecx
    jmp     .loop

.prime:
    mov     eax, 1
    ret

.not_prime:
    mov     eax, 0
    ret
```

```cpp
#include <iostream>

extern "C" int is_prime(int n);

int main() {
	std::cout << is_prime(1) << is_prime(2) << is_prime(10) << is_prime(47);
}
```

```bash
nasm -f elf64 -o is_prime.o is_prime.asm
readelf -a is_prime.o
g++ -g main.cpp is_prime.o -o main.out
```

Желательно успеть показать: дизассемблировать is_prime, выполнить одну инструкцию.

## 41. Инструкции call и ret. Что такое стековый фрейм? Что такое stack pointer и base pointer, регистры rbp и rsp? Что происходит на уровне ассемблера при вызове функций и при возврате из них? Где хранятся аргументы функций при вызове? Где хранится результат функции сразу после вызова? Что делает флаг компиляции -fno-omit-frame-pointer, зачем он нужен?

```cpp
int sum(int a, int b) {
	int res = a + b;
	return res;
}

int main() {
	int a = 5;
	int b = 38;
	int d = sum(a, b);
	
	return 0;
}
```

```asm
sum(int, int):
	push rbp
	mov rbp, rsp
	
	; ...
	pop rbp
	ret

main:
	; ...
	mov edi, edx
	mov esi, eax
	call sum(int, int)
	; ...
```


Можно попробовать рассказать про `call <адрес>`, `push <операнд>`, `pop <операнд>`, `leave`, `ret`

```asm
func:
    push rbp
    mov rbp, rsp
    sub rsp, 4000
    ; ... работа с массивом
    leave
    ret
```
Упомянуть про RedZone

```cpp
int foo(int a, int b, int c, int d, int e, int f, int g, int h, int i, int j, int k, int l, int m, int n) {

}

int main() {
	int a = 0;
	int b = 10;
	int c = 20;
	int d = 40;
	int e = 50;
	int f = 60;
	int g = 100;
	int h = 100;
	int i = 200;
	int j = 300;
	int k = 500;
	int l = 4000;
	int m = 5032;
	int n = 20;
	foo(a, b, c, d, e, f, g, h, i, j, k, l, m, n);
	
	return 0;
}
```

```cpp
struct Small { int a, b; };  // Возврат в rax:rdx
struct Large { int a[10]; }; // Возврат через указатель
```

## 42. Что такое атака переполнения буфера и какие средства защиты от нее существуют? Что такое stack protector, для чего он используется, что делает флаг компиляции -fno-stack-protector? Как получить ошибку “Stack smashing detected”? Что такое ASLR и как его отключить?

```cpp
#include <cstring>

void vulnerable() {
	char buf[16];
	strcpy(buf, "very long string that overflows buffer");
}

int main() { vulnerable(); }
```

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

## 43. Кэши процессора. Сколько уровней кэша есть в процессоре, зачем они нужны? Как узнать размеры кэшей своего процессора? Что такое кэш-линия? Почему делать обход матрицы по строкам эффективнее, чем по столбцам? Покажите эксперимент, доказывающий, что кэш-линии существуют. Покажите эксперимент, доказывающий, что существуют кэши разных уровней.

Уровни кешей. Почему L3 общий? Почему есть разделение кеша L1 на инструкционный (L1i) и кеш данных (L1d) (Гарвардская модель vs модель Фон Неймана)?

```cpp
#define SIZE (64 * 1024 * 1024)

int main() {
	int *arr = new int[SIZE];
	
	// Loop 1
	for (int i = 0; i < SIZE; i++) arr[i] *= 3;
	
	// Loop 2
	for (int i = 0; i < SIZE; i += 16) arr[i] *= 3;
	
	return 0;
}
```

```cpp
#include <iostream>
#include <ctime>

using namespace std;

const int d = 10000;
int** A = new int*[d];

int main(int argc, const char * argv[]) {
	for (int i = 0; i < d; ++i)
		A[i] = new int[d];
	
	clock_t RowMajor = clock();
	for (int a = 0; a < d; ++a)
		for (int b = 0; b < d; ++b)
			A[a][b]++;
	
	double row = static_cast<double>(clock() - RowMajor) / CLOCKS_PER_SEC;
	
	clock_t ColMajor = clock();
	
	for (int b = 0; b < d; ++b)
		for (int a = 0; a < d; ++a)
			A[a][b]++;
	
	double col = static_cast<double>(clock() - ColMajor) / CLOCKS_PER_SEC;
	
	cout << "Row Major:" << row << std::endl;
	cout << "Col Major:" << col << std::endl;
	
	return 0;
}
```

```cpp
int steps = 64 * 1024 * 1024;
int lengthMod = arr.Length - 1;
for (int i = 0; i < steps; i++)
{
    arr[(i * 16) & lengthMod]++;
}
```

```cpp
#include <algorithm>
#include <chrono>
#include <cmath>
#include <iomanip>
#include <iostream>
#include <vector>

using namespace std;
using namespace std::chrono;

double run_test(size_t size_bytes) {
  size_t num_elements = size_bytes / sizeof(int);
  int *arr = new int[num_elements]();
  int steps = 64 * 1024 * 1024;
  int lengthMod = num_elements - 1;

  for (int i = 0; i < min(steps, 100000); i++) {
    arr[(i * 16) & lengthMod] += 1;
  }

  auto start = high_resolution_clock::now();

  for (int i = 0; i < steps; i++) {
    arr[(i * 16) & lengthMod] += 1;
  }

  auto end = high_resolution_clock::now();

  volatile int dummy = arr[rand() % num_elements];
  (void)dummy;

  delete[] arr;

  auto duration = duration_cast<milliseconds>(end - start);
  return duration.count() / 1000.0;
}

void detailed_cache_test() {
  cout << "Размер теста\tВремя (сек)\tСкорость (МБ/с)\tПримечание" << endl;
  cout << "----------------------------------------------------------------"
       << endl;

  vector<pair<size_t, string>> test_points = {
      {1 * 1024, "1 KB"},
      {4 * 1024, "4 KB"},
      {16 * 1024, "16 KB"},
      {32 * 1024, "32 KB"},
      {64 * 1024, "64 KB"},
      {128 * 1024, "128 KB"},
      {256 * 1024, "256 KB"},
      {512 * 1024, "512 KB"},
      {1024 * 1024, "1 MB"},
      {2 * 1024 * 1024, "2 MB"},
      {4 * 1024 * 1024, "4 MB"},
      {8 * 1024 * 1024, "8 MB"},
      {16 * 1024 * 1024, "16 MB"},
      {32 * 1024 * 1024, "32 MB"},
      {64 * 1024 * 1024, "64 MB"},
      {128 * 1024 * 1024, "128 MB"},
      {256 * 1024 * 1024, "256 MB"},
      {512 * 1024 * 1024, "512 MB"},
      {1024 * 1024 * 1024, "1 GB"}
  };

  for (const auto &[size_bytes, label] : test_points) {
    int steps;
    if (size_bytes < 8 * 1024 * 1024) {
      steps = 64 * 1024 * 1024;
    } else if (size_bytes < 128 * 1024 * 1024) {
      steps = 32 * 1024 * 1024;
    } else {
      steps = 16 * 1024 * 1024;
    }

    size_t num_elements = size_bytes / sizeof(int);
    int *arr = new int[num_elements]();
    int lengthMod = num_elements - 1;

    for (int i = 0; i < min(steps, 100000); i++) {
      arr[(i * 16) & lengthMod] += 1;
    }

    auto start = high_resolution_clock::now();

    for (int i = 0; i < steps; i++) {
      arr[(i * 16) & lengthMod] += 1;
    }

    auto end = high_resolution_clock::now();

    volatile int dummy = arr[rand() % num_elements];
    (void)dummy;

    auto duration = duration_cast<milliseconds>(end - start);
    double time_sec = duration.count() / 1000.0;

    double data_processed = steps * 8.0 / (1024 * 1024);
    double throughput = data_processed / time_sec;

    cout << fixed << setprecision(2);
    cout << setw(10) << label << "\t" << setw(8) << time_sec << " сек\t"
         << setw(10) << throughput << " МБ/с\t";

    if (size_bytes <= 32 * 1024) {
      cout << "L1 кэш";
    } else if (size_bytes <= 256 * 1024) {
      cout << "L2 кэш";
    } else if (size_bytes <= 8 * 1024 * 1024) {
      cout << "L3 кэш";
    } else {
      cout << "RAM";
    }

    cout << endl;

    delete[] arr;
  }
}

void simple_cache_test() {
  cout << "Размер\t\tВремя (сек)" << endl;
  cout << "----------------------" << endl;

  vector<size_t> sizes = {
      16 * 1024,
      32 * 1024,
      64 * 1024,
      128 * 1024,
      256 * 1024,
      512 * 1024,
      1024 * 1024,
      2 * 1024 * 1024,
      4 * 1024 * 1024,
      8 * 1024 * 1024,
      16 * 1024 * 1024,
      32 * 1024 * 1024,
      64 * 1024 * 1024,
      128 * 1024 * 1024,
      256 * 1024 * 1024,
      512 * 1024 * 1024,
      1024 * 1024 * 1024
  };

  for (size_t size_bytes : sizes) {
    double time = run_test(size_bytes);
    cout << size_bytes / 1024 << " KB\t\t" << fixed << setprecision(3) << time
         << endl;
  }
}

int main() {
  cout << "Тест иерархии памяти на основе паттерна доступа" << endl;
  cout << "================================================" << endl << endl;

  cout << "1. Простой тест (быстрый)" << endl;
  cout << "2. Детальный тест с анализом" << endl;
  cout << "Выберите вариант (1 или 2): ";

  int choice;
  cin >> choice;

  if (choice == 1) {
    simple_cache_test();
  } else {
    detailed_cache_test();
  }

  cout << endl << "Интерпретация результатов:" << endl;
  cout << "---------------------------" << endl;
  cout << "1. Когда данные помещаются в L1 (<32KB): максимальная скорость"
       << endl;
  cout << "2. При переходе за L1 (>32KB): первый спад скорости" << endl;
  cout << "3. При переходе за L2 (>256KB): второй спад скорости" << endl;
  cout << "4. При переходе за L3 (>8MB): скорость падает до уровня RAM" << endl;
  cout << "5. Самые большие размеры (>100MB): скорость стабилизируется на "
          "уровне RAM"
       << endl;

  return 0;
}
```

## 44. Что такое branch prediction, в чем его идея? Покажите эксперимент, доказывающий существование данного явления. Как подсказать компилятору (средствами C++20, а также без него), какая из веток if более вероятна? Как это отразится на ассемблерном коде?

```cpp
#include <algorithm>
#include <ctime>
#include <iostream>

int main() {
	// Generate data
	const unsigned arraySize = 32768;
	int data[arraySize];
	
	for (unsigned c = 0; c < arraySize; ++c)
		data[c] = std::rand() % 256;
	
	// !!! With this, the next loop runs faster.
	// std::sort(data, data + arraySize);
	
	// Test
	clock_t start = clock();
	long long sum = 0;
	for (unsigned i = 0; i < 100000; ++i) {
		for (unsigned c = 0; c < arraySize; ++c) {
			// Primary loop.
			if (data[c] >= 128)
				sum += data[c];
		}
	}
	double elapsedTime = static_cast<double>(clock()-start) / CLOCKS_PER_SEC;
	
	std::cout << elapsedTime << ' ' << sum << std::endl;
	return 0;
}
```

```cpp
if (condition) [[likely]] {
	// ...
}
if (condition) [[unlikely]] {
    // ...
}

// GCC/Clang
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)
```

## 45. Что такое instruction-level parallelism в процессоре? Покажите эксперимент, доказывающий существование этого явления. Что такое out-of-order execution в процессоре, к каким неожиданным эффектам это может приводить? Что такое спекулятивное исполнение кода в процессоре? Расскажите в общих чертах, к каким уязвимостям оно может приводить.

```cpp
int main() {
	int steps = 256 * 1024 * 1024;
	int a[2] = {0, 0};
	
	// Loop 1
	for (int i=0; i<steps; i++) { a[0]++; a[0]++; }
	
	// Loop 2
	for (int i=0; i<steps; i++) { a[0]++; a[1]++; }
	
	return 0;
}
```

Сказать, что для наблюдателя все хорошо.

```cpp
int a = 0, b = 0;
void thread1() { a = 10; b = 42; }
void thread2() { while(b != 42); std::cout << a; }
```

```cpp
if (num != 0 && 10 / num == 3) {
	// ...
}
```

## 46. Что такое векторные инструкции? Что такое Intel intrinsics, какие они бывают? Что такое SIMD, AVX? Какие специальные регистры процессора используются для векторных инструкций? Покажите на примерах, как применить векторизацию для ускорения какого-либо кода.
```cpp
int foo(int a, int b) {
	float f = 3.14;
	f += 1;
}

int main() {
	foo(1, 2);
}
```

```asm
foo(int, int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-20], edi
        mov     DWORD PTR [rbp-24], esi
        movss   xmm0, DWORD PTR .LC0[rip]
        movss   DWORD PTR [rbp-4], xmm0
        movss   xmm1, DWORD PTR [rbp-4]
        movss   xmm0, DWORD PTR .LC1[rip]
        addss   xmm0, xmm1
        movss   DWORD PTR [rbp-4], xmm0
        mov eax, 1
		pop rbp
		ret
main:
        push    rbp
        mov     rbp, rsp
        mov     esi, 2
        mov     edi, 1
        call    foo(int, int)
        mov     eax, 0
        pop     rbp
        ret
.LC0:
        .long   1078523331
.LC1:
        .long   1065353216
```

```cpp
#include <stdio.h>
#include <emmintrin.h>

#define ARRAY_SIZE 8

int main() {
	alignas(16) float a[ARRAY_SIZE] = {1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0}; 
	alignas(16) float b[ARRAY_SIZE] = {8.0, 7.0, 6.0, 5.0, 4.0, 3.0, 2.0, 1.0};
	
	__m128 sum = _mm_setzero_ps();
	for (int i = 0; i < ARRAY_SIZE; i += 4) {
		__m128 a_vec = _mm_loadu_ps(&a[i]);
		__m128 b_vec = _mm_loadu_ps(&b[i]);
		__m128 prod = _mm_mul_ps(a_vec, b_vec);
		sum = _mm_add_ps(sum, prod);
	}
	
	float result[4];
	_mm_storeu_ps(result, sum);
	
	float scalar_product = result[0] + result[1] + result[2] + result[3];
	printf("The scalar product is %f\n", scalar_product);
	
	return 0;
}
```

Рассказать про семейства SIMD-инструкций.

```cpp
#include <immintrin.h>

void vector_add_avx2(float* a, float* b, float* result, size_t n) {
	size_t i;
	for (i = 0; i + 8 <= n; i += 8) {
		__m256 va = _mm256_loadu_ps(&a[i]);
		
		__m256 vb = _mm256_loadu_ps(&b[i]);
		
		__m256 vsum = _mm256_add_ps(va, vb);
		
		_mm256_storeu_ps(&result[i], vsum);
	}
	
	for (; i < n; ++i) {
		result[i] += a[i] + b[i];
	}
}
```

## 47. Как делать ассемблерные вставки в коде на Си? Расскажите о синтаксисе в общих чертах. Зачем нужно слово volatile? Приведите простейший пример, сделайте обмен значений двух переменных через ассемблерную вставку. Как с помощью ассемблерной вставки узнать количество тактов процессора, прошедшее между двумя данными строчками кода? Как применяются ассемблерные вставки в криптографии для защиты от timing attacks?

```cpp
asm [volatile] ( AssemblerTemplate 
                 : OutputOperands 
                 [ : InputOperands
                 [ : Clobbers ] ])
```

```cpp
int src = 1;
int dst;   

asm ("mov %1, %0\n\t"
    "add $1, %0"
    : "=r" (dst) 
    : "r" (src));

printf("%d\n", dst);
```

```cpp
#include <stdio.h>

int main() {
  int a = 10, b = 20;

  printf("a = %d, b = %d\n", a, b);

  __asm__ volatile("xor %0, %1\n\t"
                   "xor %1, %0\n\t"
                   "xor %0, %1"
                   : "+r"(a), "+r"(b)
                   :
                   :);

  printf("a = %d, b = %d\n", a, b);
  return 0;
}
```

```cpp
#include <stdio.h>

int main() {
  int a = 10, b = 20;

  printf("a = %d, b = %d\n", a, b);

  __asm__ volatile("xor %[x], %[y]\n\t"
                   "xor %[y], %[x]\n\t"
                   "xor %[x], %[y]"
                   : [x] "+r"(a), [y] "+r"(b)
                   :
                   :);

  printf("a = %d, b = %d\n", a, b);
  return 0;
}
```

Не забыть упомянуть `volatile`
```cpp
int x;
x = 4;
x = 5;

volatile int y;
y = 4;
y = 5;
```

```cpp
#include <stdio.h>

unsigned long long rdtsc(void) {
	unsigned long long tsc;
	asm volatile ("rdtsc": "=A" (tsc));
	return tsc;
}

int main(void) {
	unsigned long long start, end, ticks;
	
	start = rdtsc();
	int x = 0;
	for (int i = 0; i < 100; ++i) {
		x *= 5;
	}
	end = rdtsc();
	
	ticks = end - start;
	printf("Ticks: %llu\n", ticks);
	
	return 0;
}
```

```cpp
#include <cstdint>

uint32_t unsafe_add_mod(uint32_t a, uint32_t b, uint32_t mod) {
	uint32_t sum = a + b;
	if (sum >= mod) {
		return sum - mod;
	}
	return sum;
}
```

```cpp
#include <inttypes.h>

uint32_t constant_time_add_mod(uint32_t a, uint32_t b, uint32_t mod) {
	uint32_t result, temp;
	
asm volatile (
    "movl %2, %0\n\t"
    "addl %3, %0\n\t"
    "movl %0, %1\n\t"
    "subl %4, %1\n\t"
    "cmovae %1, %0\n\t"
    : "=&r" (result), "=&r" (temp)
    : "r" (a), "r" (b), "r" (mod)
    : "cc"
);
	
	return result;
}
```


## 48. Какие есть режимы работы у процессора и чем они отличаются? Чем принципиально отличается вызов сисколла от вызова обычной функции? Что делает ассемблерная инструкция syscall? Покажите пример кода, где она встречается. Покажите, как сделать сисколл execve напрямую через ассемблерную вставку. Что такое привилегированные инструкции процессора и что к ним относится?

```asm
mov $0x14,%r9d
; ...
syscall
; ...
```

```cpp
#include <unistd.h>

int main() {
	const char* path = "/bin/ls";
	const char* argv[] = { "ls", "-l", NULL };
	const char* envp[] = { NULL };
	
	asm volatile (
		"mov $59, %%rax\n\t"
		"mov %0, %%rdi\n\t"       // 
		"mov %1, %%rsi\n\t"       // 
		"mov %2, %%rdx\n\t"       // 
		"syscall"
		:
		: "r"(path), "r"(argv), "r"(envp)
		: "rax", "rdi", "rsi", "rdx", "rcx", "r11"
	);
	
	return 0;
}
```

```cpp
#include <stdio.h>

int main() {
	asm volatile ("int $0x0\n");  // Div by 0
	
	return 0;
}
```

```cpp
#include <stdint.h>

struct idt_record
{
    uint16_t  limit;      /*  - 1 */
    uintptr_t base;       /*   */
} __attribute__((packed));

void load_idt (struct idt_record *idt_r)
{
    __asm__ ("lidt %0" :: "m"(*idt_r));
}
```

## 49. Что такое прерывание, какие они бывают? Что такое interrupt descriptor table? Чем похожи и чем отличаются прерывание и вызов сисколла? Как сгенерировать прерывание с помощью ассемблерной вставки напрямую? Как ОС с помощью механизма прерываний держит под контролем все процессы?

Хорошо бы рассказать про **три основные группы** событий, требующие обработки через IDT.

```cpp
#include <stdio.h>

int main() {
	asm volatile ("int $0x3\n");
	
	return 0;
}
```

Переключение контекста.

