Сокр. IPC.

- [Статья на Wiki с хорошей табличкой](https://en.wikipedia.org/wiki/Inter-process_communication)

IPC (Inter-Process Communication) — механизмы обмена данными между процессами. Основные методы:
1. Сигналы (асинхронные уведомления)
2. Каналы (pipes) и именованные каналы (FIFO)
3. Разделяемая память (shared memory) (mmap)
4. Очереди сообщений (message queues)
5. Семафоры (semaphores)
6. Сокеты (sockets) — локальные и сетевые

Про пайпы говорилось раннее. Сейчас рассмотрим другие методы взаимодействия между процессами.

# 1. Разделяемая память (Shared Memory)
## 1.1. System V Shared Memory

Shared memory (разделяемая память) хранится в оперативной памяти (RAM) компьютера. Технически это специально выделенная область в ядре операционной системы, которая затем отображается в адресные пространства нескольких процессов.

```bash
man 2 shmop
```
- `shmat` - shared memory attach
- `shmdt` - shared memory detach

**Тезис:** Эти системные вызовы обращаются к подсистеме IPC (Inter-Process Communication) ядра Linux.

**Пример:**
```cpp
// shm_read.cpp

#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

#define SHM_SIZE 1000000 // Size of the shared memory segment

int main() {
	int shm_id;
	char *shm_ptr;
	
	// Create a unique key for the shared memory segment
	/*
	key_t key = ftok("shmfile", 65);
	if (key== -1) {
		perror("ftok");
		exit(1);
	}*/
	
	// Access the shared memory segment
	shm_id = shmget(12222, SHM_SIZE, 0666);
	if (shm_id == -1) {
		perror("shmget");
		exit(1);
	}
	
	// Attach the shared memory segment to the process's address space
	shm_ptr = (char*) shmat(shm_id, NULL, 0);
	if (shm_ptr == (char *)-1) {
		perror("shmat");
		exit(1);
	}
	
	// Read from shared memory
	printf("Reader: Read from shared memory: %s\n", shm_ptr);
	
	// Detach from shared memory
	shmdt(shm_ptr);
	
	// Optionally, remove the shared memory segment
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

#define SHM_SIZE 1000000 // Size of the shared memory segment

int main() {
	int shm_id;
	char *shm_ptr;
	// Create the shared memory segment
	shm_id = shmget(12222, SHM_SIZE, IPC_CREAT | 0666);
	if (shm_id == -1) {
		perror("shmget");
		exit(1);
	}
	
	// Attach the shared memory segment to the process's address space
	shm_ptr = (char*)shmat(shm_id, NULL, 0);
	if (shm_ptr == (char *)-1) {
		perror("shmat");
		exit(1);
	}
	
	// Write to shared memory
	const char *message = "Hello from writer!";
	sprintf(shm_ptr, "%s", message);
	printf("Writer: Wrote to shared memory: %s\n", message);
	
	getchar();
	// Detach from shared memory
	shmdt(shm_ptr);
	return 0;
}
```

Без `getchar()` процесс `shm_write` может завершиться до того, как `shm_read` прочитает данные. Для корректной работы нужна явная синхронизация (например, через семафоры или ожидание).

Если в Reader'е не вызывать `shmctl(..., IPC_RMID, ...)`, сегмент shared memory останется в системе до перезагрузки (или пока не будет удален вручную командой `ipcrm`).

**Рассмотрим карту памяти `shm_write`:**
```bash
cat /proc/$(pgrep shm_write)/maps
```
Будет интересный отрезок `/SYSV0...` - это и есть кусок разделяемой памяти

**Посмотреть глобально, сколько есть отрезков на shared memory:**
```bash
man ipcs
```

```bash
ipcs -m
```

**Как выбирать ключ так, чтобы не было коллизий:**
```bash
man 3 ftok
```
По имени файла (может быть, рандомного) выдает ключ
- Этакое хеширование

На самом деле, механизм с разделяемой памяти довольно старый.

# 2. Другие методы
## 2.1. Message queue
Очереди сообщений позволяют передавать структурированные данные.
```bash
man 7 sysvipc
```
- Это стандарт SYSTEM V - очень старый

Там есть message queue. По ключу отправляем сообщение: `msgsnd(_)` и получаем. Сообщения получаются по очереди (на то и queue).

```cpp
// sender.cpp

#include <sys/ipc.h>
#include <sys/msg.h>
#include <cstring>
#include <cstdio>

struct message {
    long mtype;
    char mtext[100];
};

int main() {
    key_t key = ftok("/tmp", 'B');
    int msgid = msgget(key, IPC_CREAT | 0666);
    
    message msg;
    msg.mtype = 1;
    strcpy(msg.mtext, "Hello Message Queue!");
    
    msgsnd(msgid, &msg, sizeof(msg.mtext), 0);
    
    printf("Sender: sent message\n");
    getchar();
    
    // msgctl(msgid, IPC_RMID, NULL);
    return 0;
}
```

```cpp
// receiver.cpp

#include <sys/ipc.h>
#include <sys/msg.h>
#include <cstring>
#include <cstdio>
#include <cstdlib>

struct message {
    long mtype;
    char mtext[100];
};

int main() {
    // Получаем тот же ключ, что и в отправителе
    key_t key = ftok("/tmp", 'B');
    if (key == -1) {
        perror("ftok failed");
        return 1;
    }
    
    // Получаем идентификатор очереди сообщений
    int msgid = msgget(key, 0666);
    if (msgid == -1) {
        perror("msgget failed");
        return 1;
    }
    
    // Буфер для приема сообщения
    message msg;
    
    printf("Receiver: waiting for message...\n");
    
    // Принимаем сообщение
    // Последний параметр 0 означает получение первого сообщения в очереди
    if (msgrcv(msgid, &msg, sizeof(msg.mtext), 0, 0) == -1) {
        perror("msgrcv failed");
        return 1;
    }
    
    printf("Receiver: received message type %ld: %s\n", msg.mtype, msg.mtext);
    
    // Удаляем очередь сообщений (опционально)
    // msgctl(msgid, IPC_RMID, NULL);
    
    return 0;
}
```

## 2.2. Семафоры
Есть еще семафоры. Это механизм синхронизации
- Когда нужна гарантия, что только один процесс обращается к разделяемой памяти
- Но это очень древний способ (сейчас используют мьютексы).

Семафоры — это счетчики, позволяющие ограничить доступ к ресурсу. В Linux есть:
- Семафоры System V (`semget`, `semop`)
- POSIX семафоры (`sem_init`, `sem_wait`, `sem_post`)

```cpp
#include <sys/sem.h>
#include <cstdio>

int main() {
    // 1. СОЗДАНИЕ КЛЮЧА
    // Генерируем уникальный ключ для идентификации семафора
    // "/tmp" - путь к существующему файлу
    // 'C' - произвольный символ-идентификатор
    key_t key = ftok("/tmp", 'C');
    
    // 2. СОЗДАНИЕ/ПОЛУЧЕНИЕ СЕМАФОРА
    // semget - создает или получает идентификатор набора семафоров
    // key - уникальный ключ
    // 1 - количество семафоров в наборе (нам нужен один)
    // IPC_CREAT | 0666 - создать семафор, если не существует, с правами доступа rw-rw-rw-
    int semid = semget(key, 1, IPC_CREAT | 0666);
    
    // 3. ИНИЦИАЛИЗАЦИЯ СЕМАФОРА
    // Устанавливаем начальное значение семафора в 1
    // semid - идентификатор набора семафоров
    // 0 - номер семафора в наборе (индекс, начинается с 0)
    // SETVAL - команда "установить значение"
    // 1 - начальное значение (бинарный семафор, 1 = доступ свободен)
    semctl(semid, 0, SETVAL, 1);
    
    // 4. ЗАХВАТ РЕСУРСА (P-ОПЕРАЦИЯ)
    struct sembuf op;           // Структура для операции над семафором
    op.sem_num = 0;            // Номер семафора в наборе
    op.sem_op = -1;            // P-операция: уменьшить значение на 1
    op.sem_flg = 0;            // Флаги (0 - обычная операция)
    
    // Выполняем операцию над семафором
    // Блокируемся, если значение семафора <= 0
    semop(semid, &op, 1);
    
    // КРИТИЧЕСКАЯ СЕКЦИЯ
    // С этого момента только один процесс может выполнять этот код
    printf("Critical section start\n");
    getchar();  // Ожидаем ввод пользователя (имитация работы в критической секции)
    
    // 5. ОСВОБОЖДЕНИЕ РЕСУРСА (V-ОПЕРАЦИЯ)
    op.sem_op = 1;             // V-операция: увеличить значение на 1
    semop(semid, &op, 1);      // Разблокируем семафор
    
    // 6. УДАЛЕНИЕ СЕМАФОРА (закомментировано)
    // semctl(semid, 0, IPC_RMID); // Удаляем семафор из системы
    
    return 0;
}
```

## 2.3. POSIX'ный 
- Вообще, есть еще метод создания разделяемой памяти - от POSIX

```bash
man 7 shm_overview
```
- `shm_open` - создание и открытие

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>     // For 0_* constants
#include <sys/mman.h>  // For shm_open and mmap
#include <unistd.h>    // For close

#define SHM_NAME "/my_shm"
#define SHM_SIZE 256

int main() {
	int shm_fd;
	char *ptr;
	
	// Create shared memory object
	shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
	if (shm_fd == -1) {
		perror("shm_open");
		exit(1);
	}
	
	// Configure the size of the shared memory object
	ftruncate(shm_fd, SHM_SIZE);
	
	// Map the shared memory object in the address space
	ptr = mmap(0, SHM_SIZE, PROT_WRITE, MAP_SHARED, shm_fd, 0);
	if (ptr == MAP_FAILED) {
		perror("mmap");
		exit(1);
	}
	
	// Write to shared memory
	const char *message = "Hello from writer!";
	sprintf(ptr, "%s", message);
	printf("Writer: Wrote to shared memory: %s\n", message);
	
	getchar();
	
	// Clean up
	munmap(ptr, SHM_SIZE);
	close(shm_fd);
	shm_unlink(SHM_NAME);
	
	return 0;
}
```

POSIX shared memory использует файлы в `/dev/shm`. Это более современный и гибкий механизм по сравнению с System V.
```bash
cd /dev/shm
ls -l
```

# 3. Сравнение IPC-механизмов
| Механизм       | Пропускная способность | Сложность | Синхронизация     | Тип коммуникации          |
| -------------- | ---------------------- | --------- | ----------------- | ------------------------- |
| Сигналы        | Низкая                 | Низкая    | Нет               | Асинхронный               |
| Pipes          | Средняя                | Средняя   | Да (блокировки)   | Синхронный                |
| FIFO           | Средняя                | Средняя   | Да                | Синхронный                |
| Shared Memory  | Высокая                | Высокая   | Требует семафоров | Синхронный / Асинхронный* |
| Message Queues | Средняя                | Высокая   | Встроенная        | Синхронный / Асинхронный* |
| Сокеты         | Средняя/Высокая        | Высокая   | Да                | Синхронный / Асинхронный* |
`*` — зависит от режима использования (например, неблокирующий ввод-вывод, callback-механизмы).
