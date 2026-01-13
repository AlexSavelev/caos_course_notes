Процессы и межпроцессные взаимодействия. Сигналы.

Процесс — это экземпляр выполняющейся программы. Он включает в себя:
- Виртуальное адресное пространство (maps)
- Таблицу файловых дескрипторов
- Контекст выполнения (регистры, стек, счетчик команд)
- Учетные данные (UID, GID)
- Состояние (running, sleeping, zombie и т.д.)
- Ресурсы (память, CPU, IO)

Программа — это статичный исполняемый файл (ELF, PE), содержащий код, данные и метаинформацию.

- Одному процессу (в конкретный момент времени) соответствует ровно одна программа
- Одной программе может соответствовать МНОГО процессов (как минимум, можно несколько раз запустить одну программу)

**Посмотреть на текущие процессы:**
```bash
htop
```

# Создание процесса
## `fork`
Будет интересовать создание процессов прямо в программе.
- В Windows все прямолинейно: `CreateProcess` и готово
- В Linux через `fork`
	- В рамках курса это самый сложный syscall (очень не интуитивный)

`fork` клонирует текущий процесс.
- Копирует ВЕСЬ `maps` (всю таблицу страниц (~1Kb)), причем само пространство копируется лениво (COW). Это означает, что физические страницы дублируются только при попытке записи, что экономит память и ускоряет `fork`.

```cpp
#include <iostream>
#include <unistd.h>

int main() {
	std::cout << "Hello!" << std::endl;
	
	int pid = fork();
	
	std::cout << "Goodbye!";

	return 0;
}
```
Запускаем: `Hello!Goodbye!Goodbye!` - как и ожидалось. Без `std::endl` будет дважды `Hello!`, поскольку в таком случае `Hello!` будет висеть в буфере, который тоже скопируется для нового процесса.

Вообще `fork` - самая мемная функция
- Порождения мультивселенных, НМТ - из этой же оперы

**Что возвращает `fork`:**
- Клону вернут `0`
- Оригиналу вернут PID клона
```cpp
if (pid == 0) {
	std::cout << "I am child, my process is " << getpid() << '\n';
} else {
	std::cout << "I am parent of child process " << pid << '\n';
}
```

**Как в целом узнать PID текущего процесса:**
`getpid()` — системный вызов, возвращающий PID текущего процесса. В Linux он кэшируется в пользовательском пространстве для скорости (vsyscall/vdso).

**Полноценный пример с обработкой возращаемого значения:**
```cpp
#include <iostream>
#include <unistd.h>

struct A {
	A() { std::cout << "Created\n"; }
	~A() { std::cout << "Destroyed\n"; }
};

int main() {
	A a;
	
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}
	
	std::cout << "Goodbye!";

	return 0;
}
```
Очевидно, что будет `CreatedDestroyedDestroyed`

Может ли компилятор соптимизировать и создать а после fork?
- Нет, компилятор не может перенести конструктор после `fork`, потому что `fork` — системный вызов с видимыми побочными эффектами (создание процесса). Оптимизации, нарушающие порядок side effects, запрещены.

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int fd = open("example.txt", O_RDWR | O_CREAT);
	
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
		write(fd, "Bobobo!", 7);
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
		write(fd, "Pupupu!", 7);
	}
	
	close(fd);

	return 0;
}
```
Содержимое `example.txt`: `Pupupu!Bobobo!`

## `exec`
`execve` - новый syscall

Процесс становится процессом другой программы
- Обнуляется ПОЧТИ ВСЕ: стек, куча и все data сегменты (инициализированые и неинициализированные)
- PID остается, файловые дескрипторы остаются (если не установлен `O_CLOEXEC`)
Благодаря этому мы можем подготовить для этого процесса окружение: такие-то потоки, такие-то лимиты...
- В этом особенность связки `fork` + `exec`

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
		
		char* argv[] = {"/usr/bin/ls", ".", "-l", NULL};
		char* envp[] = {NULL};  // Набор переменных окружения

		execve("/usr/bin/ls", argv, NULL);  // Можно просто NULL
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}
	
	return 0;
}
```

**Более короотко:**
```c
char *args[] = {"/bin/ls", "-l", NULL};
char *env[] = {"PATH=/usr/bin", NULL};
execve(args[0], args, env);
perror("execve failed"); // выполнится только при ошибке
```

Возможно, вывод `ls` выведется после исполнения программы, потому что родитель завершиться раньше и родительский процесс завершиться

На самом деле, есть целое семейство exec: `execl`, `execv`, `execle`, `execve`, `execlp`, `execvp`. Различия:
- `l` vs `v`: передача аргументов списком или массивом
- `e`: передача переменных окружения
- `p`: поиск программы в PATH

**`execve` сохраняет файловые дескрипторы:**
```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int fd = open("example.txt", O_RDWR | O_CREAT);

	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid();  // Это не выведется, тк буфер очиститься (вернее, его не будет существовать к теоретическому моменту flush)

		dup2(fd, 1);  // stdout - это теперь example.txt

		char* argv[] = {"/usr/bin/ls", ".", "-l", NULL};
		char* envp[] = {NULL};

		execve("/usr/bin/ls", argv, NULL);  // теперь будет писаться в example.txt
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}

	close(fd);
	return 0;
}
```

# `wait`
- Когда мы запустили дочерний процесс, мы как родитель хотим узнать, как он отработал

```bash
man 2 wait
```

- `wait` ждет, когда завершиться какой-нибудь ребенок
- `waitpid` ждет, когда завершиться данный процесс

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
		
		// OR:
		// exit(52) (правда, возможно, res = 52 * 256)
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
		
		int res;
		int pid = wait(&res);
		std::cout << "Child exited with status " << res << '\n';
	}

	return 0;
}
```

Статус завершения кодируется: `WIFEXITED(status)`, `WEXITSTATUS(status)`, `WIFSIGNALED(status)`, `WTERMSIG(status)`. Например, сигнал SIGSEGV (11) будет представлен как `status = 11`.

- Если родитель умирает раньше ребенка?
	- Ребенок переподвязывается к systemd (усыновляется)

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
		sleep(1);
		std::cout << getppid() << std::endl;  // /usr/lib/systemd/...
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
	}

	return 0;
}
```

# Состояния процессов (Process States)
- У каждого процесса есть состояние - State
- R — Running - те, которые сейчас выполняются (не обязательно, что ПРЯМО в данный момент крутятся на ядре. Достаточно просто быть в очереди ядра на исполнение)
- S — (Interruptible) Sleep - висят на IO, wait'е или другом syscall'е
- D — Uninterruptible sleep - процесс завис на операции IO и его нельзя прервать, так как иначе диск (или что-то другое) может испортиться
- Z — Zombie - процессы, которые завершились, но родитель сам продолжает работать и еще не вызвал `wait`
- T — Stopped - процессы, которые остановили
	- ОС замораживает процесс, при этом состояние памяти, инструкция процессора остается как было.

## Zombie

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>

int main() {
	int pid = fork();
	
	if (pid == 0) {
		std::cout << "I am child, my process is " << getpid() << '\n';
		// Ребенок будет зомби
	} else {
		std::cout << "I am parent of child process " << pid << '\n';
		getchar();
	}

	return 0;
}
```

- Причем можем зайти в `/proc/{PID_OF_ZOMBIE_CHILD}`
	- Там многое остается

## Stopped
- Как сделать stop процесса
```cpp
#include <iostream>

int main() {
	std::cout << "Hello\n";
	getchar();
	std::cout << "Hello again\n";
	
	return 0;
}
```
Запустим. Пока будет ждать, он будет в состоянии S.
`Ctrl+Z` - и процесс остановится
- Попробуем вызвать `exit` из терминала - он откажется выходить

**Список всех процессов по порядку:**
```bash
ps -ef
```

**Более подробный вывод:**
```bash
ps aux
```

**Вывод с состояниями:**
```bash
ps -eo pid,ppid,state,cmd
```

**Список работающих процессов:**
```bash
jobs
```

**Продолжать работу:**
- `fg` — foreground — запустить в терминале
- `bg` — background — запустить в фоне
- Мануала под эти программы нет, так как это программы shell'а
```bash
fg
```

- **Можно изначально запустить в background'е, добавив в конец `&`:**
```bash
./a.out &
```

# Взаимодействия процессов. Сигналы
Сигналы — это асинхронные уведомления, отправляемые процессу. Они могут быть отправлены ядром, другим процессом или самим процессом. Сигналы могут быть обработаны (handled), проигнорированы (ignored) или вызвать действие по умолчанию (default action).

Как правило, сигнал — это метод попросить ОС убить какой-нибудь процесс
- Базово — команда `kill` — она вызывает сигналы

**Программа грузит процессор на 100%:**
```cpp
int main() {
	while(true) {}
	
	return 0;	
}
```

**Процесс будет Terminated (`-TERM`):**
```bash
kill 5544
```

Однако некоторые  так "терминировать" нельзя. **Придется полноценно убить:**
```bash
kill -KILL 5544
```
Процесс будет Killed, и если посмотрим на `/proc/`, то там не найдем этот процесс.
- `-KILL` или `-9` (как номер `-KILL`)

**Нельзя убить процесс, запущенный, скажем, root'ом:**
```bash
kill 30
kill -KILL 30
```

Но вообще, сигналы есть не только Kill и Terminate

**Есть сигнал SegFault с номером `11`:**
```bash
kill -11 $(pgrep a.out)
```

**Есть сигнал SigInt (interrupt) с номером `2`:**
```bash
kill -INT
```
- Это тот самый сигнал, который вызывается при нажатии `Ctrl+C`

**Список всех сигналов:**
```bash
man 7 signal
```

Основные сигналы:
- `SIGINT` (2) — прерывание с терминала (Ctrl+C)
- `SIGQUIT` (3) — как `INT`, но с Core Dump'ом (Ctrl+`\`)
- `SIGKILL` (9) — безусловное убийство (нельзя перехватить)
- `SIGSEGV` (11) — нарушение сегментации
- `SIGPIPE` (13) — запись в разорванный pipe
- `SIGCHLD` (17) — ребенок завершился
- `SIGSTOP` (19) — остановка (нельзя перехватить)
- `SIGCONT` (18) — продолжение
- `SIGABRT`

**Рандомный факт:** `Ctrl+D`, кстати, просто печатает `EOF`

`Ctrl+Z` - сигнал остановки (Stopped)
- `SIGSTOP` - стоп
- `SIGTSTP` - тоже стоп,  посылаем мы именно его, когда стопим, его можно перехватить

Нельзя перехватить только `KILL` и `STOP`. При Double free вызывается `SIGABRT`

**По факту, `kill` вызывает syscall `kill`:**
```bash
man 2 kill
```
`kill(pid, sig)` — отправляет сигнал `sig` процессу с PID `pid`. Если `pid = 0`, сигнал отправляется всем процессам текущей группы.

## Как физически происходит, что процессу был послан сигнал
С точки зрения процессора, происходит прерывание.

Есть ряд ячеек процессора, в которые пишется номер ошибки. В самом начале исполнения программы, создается отображения кода ошибки в адрес функции-обработчика, находящейся в ядре ОС.
Обрабочник из ядра ОС, таким образом, перехватывает процесс:
- Если у программы настроен обработчик, то вызывается определенная функция и управление передается обратно процессу
- Обработчик настраивается через специальный syscall
- Если же обработчик не настроен, то ОС процессу судья

**Подробнее:**
Когда ядро решает доставить сигнал процессу, оно:
1. Проверяет маску блокировки сигналов процесса (`sigprocmask`).
2. Если сигнал не заблокирован и есть обработчик, ядро сохраняет контекст процесса и переключает выполнение на обработчик.
3. После возврата из обработчика восстанавливается исходный контекст.

## Как настроить обработчик
Хочу уметь перехватывать сигналы. Можно навесить на все, кроме `KILL`, `STOP`

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
	// SIGINT = 2, но можно макросом
	signal(SIGINT, &handler);

	while(true) {}

	return 0;
}
```
При запуске и попытке нажатия `Ctrl+C`, будет просто выводиться сообщение.

Вообще, когда мы выходим из обработчика, мы возвращаемся на последнюю инструкцию, во время выполнения которой вызвался этот сигнал

**Повесим на SegFault:**
```cpp
#include <cstdio>
#include <signal.h>

void handler(int num) {
	printf("I won't die\n");
}

int* p = nullptr;

int main() {
	// SIGINT = 2, но можно макросом
	signal(SIGSEGV, &handler);

	*p = 1;

	return 0;
}
```
Будет постоянно выводить `I won't die!`. Потому что при выходе из обработчике мы возвращаемся к той же инструкции

**Если будет SegFault из обработчика:**
```cpp
#include <cstdio>
#include <signal.h>

int* p = nullptr;

void handler(int num) {
	printf("I won't die\n");
	*p = 1;
}

int main() {
	signal(SIGSEGV, &handler);

	*p = 1;

	return 0;
}
```
Если будет SegFault из обработчика, то это уже не обрабатываем мы. После первого сообщения будет SegFault Core Dumped.

**Но, пока обрабатываю сигнал, я могу ловить другие:**
```cpp
#include <cstdio>
#include <signal.h>

int x = 0;

void handler_fpe(int num) {
	printf("I won't die\n");
}

void handler(int num) {
	printf("I won't die\n");
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

Как сказано раннее, но на самом деле у ОС есть маска
- Пока не вышли из обработчика конкретного сигнала, значение маски в точке, отвечающей за этот сигнал, 1
- То есть можно обрабатывать только по одному для каждого сигнала
То есть при входе в обработчик сигнала, этот сигнал автоматически блокируется (добавляется в маску блокировки). Это предотвращает рекурсивный вызов обработчика для того же сигнала. Другие сигналы могут приходить и обрабатываться, если они не заблокированы.


Очередь сигналов бинарная. Для `SEGV` и `FFE` же очередь с ожиданием не создается: это сигналы, которые ТОЧНО надо обработать, так как мы не можем обработать инструкцию

**Пример с `sleep`:**
```cpp
#include <cstdio>
#include <unistd.h>
#include <signal.h>

void handler(int num) {
	printf("I won't die\n");
}

int main() {
	signal(SIGINT, &handler);

	sleep(5);

	return 0;
}
```
Во время `sleep` сигналы (`SIGINT`) тоже получаю (в `man sleep` это указано).

Но если процесс находится в состоянии `STOP`, то смогу получать только `CONTinue`, `KILL`

### Как можно отловить SEGV, чтобы от исчез

```cpp
#include <cstdio>
#include <signal.h>

int* p = nullptr;

int x;

void handler(int num) {
	printf("I won't die\n");
	p = &x;
}

int main() {
	signal(SIGSEGV, &handler);

	*p = 1;

	return 0;
}
```
Ничего не получится. Ассемблер уже загрузил адрес в регистр и все пытается его разыменовать

**Еще пример:**
```cpp
#include <cstdio>
#include <signal.h>

int* ptr = nullptr;

void handler(int num) {
	printf("I won't die\n");
	mprotect(ptr, 1000, PROT_READ | PROT_WRITE);
}

int main() {
	signal(SIGSEGV, &handler);
	
	ptr = (int*)mmap(0, 1000, PROT_READ, MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
	*ptr = 5;  // SEGV, права только на чтение

	return 0;
}
```

Обработчик `SIGSEGV` может исправить причину ошибки (например, сделать страницу доступной для записи через `mprotect`), и выполнение продолжится с инструкции, вызвавшей ошибку. Однако это требует аккуратной настройки и понимания архитектуры.

Обработчик сигнала выполняется в том же потоке, что и основная программа, используя тот же стек. Ядро сохраняет контекст (регистры) на стеке перед вызовом обработчика. Если стек переполнен, может произойти двойная ошибка (`SIGSEGV` внутри `SIGSEGV`), что приведет к аварийному завершению.

## `sigaction`
- Почти то же, что и `signal`, но более расширенная

```bash
man 2 sigaction
```

`sigaction` позволяет указать флаги (например, `SA_RESTART` для автоматического перезапуска прерванных системных вызовов) и получить дополнительную информацию о сигнале (`siginfo_t`).

**Пример:**
```cpp
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

void handler(int sig, siginfo_t *info, void *ucontext) {
    printf("Got signal %d from PID %d\n", sig, info->si_pid);
}

int main() {
    struct sigaction sa;
    sa.sa_sigaction = handler;
    sa.sa_flags = SA_SIGINFO;
    sigemptyset(&sa.sa_mask);
    
    sigaction(SIGINT, &sa, NULL);
    
    while(1) pause();
    return 0;
}
```

## `pause`
Syscall паузы.
Засыпаем (переходим в состояние `sleep`) до прихода ЛЮБОГО сигнала.

Как мы помним, сигналы (например, `SIGINT` от Ctrl+C) — это механизм уведомлений ядра процессу. `pause()` — это один из способов **ожидания** такого уведомления.

**Типичный сценарий использования:**
1. Программа устанавливает обработчик для определенного сигнала (например, с помощью `signal()` или `sigaction()`).
2. Затем программа вызывает `pause()`, чтобы приостановиться и дождаться этого сигнала.
3. При получении сигнала:
    - Если для сигнала установлен обработчик, он выполняется.
    - После завершения работы обработчика системный вызов `pause()` завершается (возвращает управление), и программа продолжает работу со следующей после `pause()` инструкции.

```cpp
#include <stdio.h>
#include <unistd.h>
#include <signal.h>

void signal_handler(int sig) {
    printf("Получен сигнал %d! Пробуждаемся...\n", sig);
}

int main() {
    // Устанавливаем обработчик для SIGUSR1
    signal(SIGUSR1, signal_handler);

    printf("Процесс PID=%d приостановлен. Отправьте ему 'kill -SIGUSR1 %d'\n", getpid(), getpid());
    pause(); // Спим здесь до получения сигнала

    printf("Программа продолжила работу после pause().\n");
    return 0;
}
```

## `raise`
`kill`, но для себя

```bash
man 3 raise
```

## `SIGCHLD`
Child stopped or terminated
Дефолтно: игнорировать, но можем задавать самостоятельно

Если не обрабатывать `SIGCHLD`, дочерние процессы становятся зомби.

**Пример обработки:**
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

## `SIGHUP`
Hangup detected on controlling terminal or death of controlling process
- Hangup - бросить трубку
Если я на удаленном сервере по SSH что-то запустил, а потом я отключился, то процессу подается сигнал `SIGHUP`

**Запустить команду, которая игнорирует hangup'ы:**
```bash
man nohup
```

**Продолжает работу даже после закрытия терминала**
- Если же запускать без `nohup`, то терминал при закрытии будет вызывать `SIGHUP` и программа будет останавливаться
```bash
nohup ./a.out &
```

## `SIGXCPU`
Через syscall `setrlimit` (`r` = resource) можно ставить лимиты;
Через syscall `getrlimit` можно эти лимиты получать.

**Можно посмотреть, какими лимитами обладает процесс:**
```bash
ulimit -a
```
По умолчанию cpu time unlimited. Но можно поставить свое и при установки лимита и истечении времени булет вызван сигнал `SIGXCPU`.

Аналогично для `SIGXFSZ` - макс. размер файла

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
    limit.rlim_cur = 1; // 1 second soft limit
    limit.rlim_max = 5; // 5 seconds hard limit
    setrlimit(RLIMIT_CPU, &limit);
    
    signal(SIGXCPU, handler);
    
    while(1); // infinite loop
    return 0;
}
```

```cpp
#include <sys/resource.h>

struct rlimit rlim;
getrlimit(RLIMIT_NOFILE, &rlim); // максимальное число открытых файлов
rlim.rlim_cur = 1024;
setrlimit(RLIMIT_NOFILE, &rlim);
```

### Про hard limit и soft limit
- Soft limit — текущее ограничение, может быть изменено процессом.
- Hard limit — максимальное значение, до которого процесс может поднять soft limit. Изменить hard limit может только суперпользователь.

```bash
ulimit -S -t 10  # soft limit CPU time 10 sec
ulimit -H -t 20  # hard limit CPU time 20 sec
```

## Замечание про обработку

**Вернемся к примеру:**
```cpp
#include <cstdio>
#include <signal.h>

int* p = nullptr;

int x;

void handler(int num) {
	printf("I won't die\n");
	p = &x;
}

int main() {
	signal(SIGSEGV, &handler);

	*p = 1;

	return 0;
}
```
На самом деле, везде UB.
Когда мы находимся в обработчике сигналов и вызываем `printf`, может быть, что какой-нибудь буфер в `printf` неконсистентный (поврежденный), то мы сломаемся - UB.
То есть проблема в следующем: вызываемые при обработке сигналов функции могут быть не в консистентом состоянии.

**Функции, которые можно вызывать при обработке сигналов, называются reentrant-функциями (от сл. re-entered):**
```bash
man 7 signal-safety
```
Тут перечислены все таковые функции, в том числе и syscall'ы. Этот список предоставляется из стандарта POSIX, который описывает, как реализовать ту или иную фукнцию.

Reentrant-функции не используют статические данные, не вызывают не-reentrant функций и безопасны для вызова из обработчиков сигналов. Примеры: `write`, `read`, `sleep`, `_exit`. В свою очередь, `printf`, `malloc` — не reentrant.

# Некоторые параметры процессов
В (h)top'е есть параметры `PR` (priority), `NI` (niceness):
- `PR` - приоритет (насколько много ресурсов хочет есть процесс)
- `NI` - хорошесть - насколько процесс хочет своими (выделенными для себя) делиться ресурсами с другими

**Выполняет процесс с измененным приоритетом:**
```bash
man 1 nice
```

**Скомпилируем код:**
```cpp
int main() {
	while (true) {}
}
```

**Запустим:**
```bash
nice -n 15 a.out
```
Крутиться на ядре. `NI=15`, в `htop` обозначается синим

**Попытаемся запустить процесс с пониженным niceness'ом:**
```bash
nice -n -10 a.out
```
Если мы без прав, будет ошибка.

Niceness: от -20 (самый высокий приоритет) до 19 (самый низкий). По умолчанию 0. Только root может установить отрицательное значение.

## `renice`
Еще есть `renice` - программа, которая ставит приоритет уже запущенной программы

**Документация также на странице:**
```bash
man 2 nice
```

$priority = niceness + 20$, но не всегда
- Но `NI` - первичный, можем менять
- `PR` - вторичный, т.е. определяется ОС на основе `NI` и состояния процессов

На самом деле, в Linux используется динамический приоритет (dynamic priority), который ядро пересчитывает на основе niceness, истории использования CPU и других факторов. `PR` в top показывает именно динамический приоритет.

## `sched_affinity`
CPU affinity позволяет привязать процесс к определенным ядрам CPU. Это полезно для кэш-локальности и предсказуемости производительности.
```bash
man 7 sched
```
- Там же написано, как работает sheduler (планировщик), распределяет приоритеты и тп.

```cpp
#include <sched.h>
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(0, &set); // привязать к CPU 0
sched_setaffinity(0, sizeof(set), &set);
```

## `EUID`
У процесса есть параметр `EUID` (effective user id).
Есть 2 user'а: тот, кто процесс запустил, и от имени которого процесс работает
- Например, когда запускаем процесс от имени другого имени (например, `sudo`), эти пользователи отличаются

Если по правде, то 3:
- Real UID (RUID) — пользователь, запустивший процесс.
- Effective UID (EUID) — пользователь, от имени которого процесс выполняет операции (например, доступ к файлам).
- Saved UID (SUID) — сохраняется при запуске setuid-программ, позволяет временно повысить привилегии.

**Здесь будут равны:**
```cpp
#include <unistd.h>
#include <iostream>

int main() {
	std::cout << getuid() << ' ' << geteuid() << '\n';

	while (true) {}
}
```

**А здесь уже нет:**
```bash
sudo -u other-user ./a.out
```

То есть когда ядро проверяет на привилегии, оно смотрит именно на `euid`.

### `Setuid`-бит
Setuid-бит (`chmod u+s`) заставляет процесс запускаться с EUID владельца файла. Это механизм временного повышения привилегий (например, `passwd`, `sudo`).

```bash
ls -l /usr/bin
```
Там будет `sudo` с битом `s`
- Кто бы ни запускал этот файл, он работает с `euid` как у владельца (т.е. root'а)

## Capability
Мы как процесс по умолчанию многое не можем: создавать новых пользователей, менять порты и тд.
- Если запускать от `root`'а, то все можем
Раньше было простое разделение: процесс привилегированный и не привилегированный. Но было быстро ясно, что это плохая практика.
Поэтому ввели `capabilities`

```bash
man 7 capability
```

Capabilities разбивают привилегии root на отдельные возможности. Например:
- `CAP_NET_RAW` — возможность использовать RAW-сокеты.
- `CAP_SYS_ADMIN` — множество административных операций.

**Установка capabilities:**
```bash
sudo setcap cap_net_raw+ep /path/to/program
```

**Просмотр capabilities процесса:**
```bash
cat /proc/<PID>/status | grep Cap
```

## `seccomp`

Запретить процессу вызывать определенные syscall'ы. То есть:

Seccomp (secure computing mode) позволяет ограничить системные вызовы, доступные процессу. Режимы:
- `SECCOMP_MODE_STRICT` — только `read`, `write`, `_exit`, `sigreturn`.
- `SECCOMP_MODE_FILTER` — настраиваемый фильтр BPF (Berkeley Packet Filter).

```bash
man 2 seccomp
```

Если процесс попытается вызвать запрещенный, то словим `SIGSYS` - Bad system call

Используется в контейнерах (Docker), sandbox-ах (Chrome, Firefox).

**Пример:**
```cpp
#include <iostream>
#include <seccomp.h>
#include <unistd.h>
#include <sys/prctl.h>

void setup_seccomp() {
	// Create a new seccomp filter
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW); // Default action is to allow
	if (ctx == nullptr) {
		perror("seccomp_init");
		exit(1);
	}
	
	// Add rules to deny specific syscalls
	seccomp_rule_add(ctx, SCMP_ACT_KILL_PROCESS, SCMP_SYS (execve), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS (fork), 0);
	
	// Load the filter into the kernel
	if (seccomp_load(ctx) < 0) {
		perror("seccomp_load");
		exit(1);
	}
	
	// Release the filter context
	seccomp_release(ctx);
}

int main() {
	setup_seccomp();
	
	std::cout << "Process is running with restricted syscalls." << std::endl;
	
	char *args[] = {"/bin/ls", NULL};
	execve(args[0], args, NULL); // This will fail
	
	while (true) {}	
	return 0;
}
```
- Хотя в идеале, провять `seccomp_rule_add` на наличие ошибок.
