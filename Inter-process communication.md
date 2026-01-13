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

## Разделяемая память (Shared Memory)
### System V Shared Memory
```bash
man 2 shmop
```
- `shmat` - shared memory attach
- `shmdt` - shared memory detach

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

Без `getchar()` процесс `shm_write` может завершиться до того, как `shm_read` прочитает данные. Это приведёт к вызову `shmctl(shm_id, IPC_RMID, NULL)` и удалению сегмента до его чтения. Для корректной работы нужна явная синхронизация (например, через семафоры или ожидание).

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

На самом деле, этот механизм довольно старый.

# Другие методы
## Message queue
Очереди сообщений позволяют передавать структурированные данные.
```bash
man 7 sysvipc
```
- Это стандарт SYSTEM V - очень старый

Там есть message queue. По ключу отправляем сообщение: `msgsnd(_)` и получаем. Сообщения получаются по очереди (на то и queue).

```cpp
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

## Семафоры
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
    key_t key = ftok("/tmp", 'C');
    int semid = semget(key, 1, IPC_CREAT | 0666);
    
    // инициализация семафора значением 1
    semctl(semid, 0, SETVAL, 1);
    
    struct sembuf op;
    op.sem_num = 0;
    op.sem_op = -1; // P-операция (захват)
    op.sem_flg = 0;
    
    semop(semid, &op, 1);
    printf("Critical section start\n");
    getchar();
    
    op.sem_op = 1; // V-операция (освобождение)
    semop(semid, &op, 1);
    
    // semctl(semid, 0, IPC_RMID);
    return 0;
}
```

# POSIX'ный 
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

# Сравнение IPC-механизмов
| Механизм       | Пропускная способность | Сложность | Синхронизация     | Тип коммуникации          |
| -------------- | ---------------------- | --------- | ----------------- | ------------------------- |
| Сигналы        | Низкая                 | Низкая    | Нет               | Асинхронный               |
| Pipes          | Средняя                | Средняя   | Да (блокировки)   | Синхронный                |
| FIFO           | Средняя                | Средняя   | Да                | Синхронный                |
| Shared Memory  | Высокая                | Высокая   | Требует семафоров | Синхронный / Асинхронный* |
| Message Queues | Средняя                | Высокая   | Встроенная        | Синхронный / Асинхронный* |
| Сокеты         | Средняя/Высокая        | Высокая   | Да                | Синхронный / Асинхронный* |
`*` — зависит от режима использования (например, неблокирующий ввод-вывод, callback-механизмы).
