- Они же - потоки
- Знаем про процессы, а есть еще и потоки
	- Разница небольшая есть

Поток (thread) — это легковесный процесс, который разделяет с другими потоками того же процесса адресное пространство, файловые дескрипторы и другие ресурсы. Процессы изолированы, потоки — нет.

# 1. На уровне C++
Thread на уровне программы — это как бы отдельная сущность, исполняющая команды параллельно.

```cpp
#include <iostream>
#include <thread>

void f() {
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		std::cout << i << std::endl;
	}
}

int main() {
	std::thread t(f);  // Создает второй поток, работает "параллельно"

	for (int i = 0; i < 1'000'000; ++i) {
		std::cout << i << std::endl;
	}

	return 0;
}
```

Понятие "параллельно" — только на уровне программы. На деле, процессор просто шедулит процессы
Если запустим, увидем, что ОС переключает контекст - сначала пачка с  `main`'а, затем пачка с `f`. `std::endl` иногда выводится дважды, а иногда числа конкатенируются. То есть иногда второй вывод (`std::endl`) обрабатывается после смены контекста.

**Пример ниже — это UB:**
- Вектор во время реаллокации могут прервать
- Double free corruption ИЛИ Segmentation Fault
- Недетерминированно
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

	return 0;
}
```

**Классический уже полезный пример — сумма чисел:**
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
		t.join();  // а-ля wait
	}
	
	long long result = 0;
	for (int i = 0; i < 8; ++i) {
		result += arr[i];
	}
	std::cout << result << std::endl;

	return 0;
}
```

## 1.1. Про `detach`
Если мы выйдем из программы, пока thread'ы еще будут выполняться, то вызовется `std::terminate`.
Но можно Deattach'ить. `detach` позволяет потоку работать независимо от основного. После `detach` поток становится "демоном", и его ресурсы автоматически освобождаются по завершении.
```cpp
std::thread t(f);
t.detach();
// main может завершиться, поток продолжит работу
```

# 2. На уровне Linux
На Linux конпепция потоков существуют. Конечно, в далеком прошлом поток был равен процессу. Сейчас это немного другая концепция.
Но что осталось самым важным — потоки разделяют одно и то же адресное пространство (maps один и тот же), также у них одинаковые Signal Handler'ы.

В Linux потоки реализованы как легковесные процессы (Lightweight Processes, LWP), которые используют `clone()` с флагами, указывающими на общие ресурсы.

В Top'е можно включить `TGID` (thread group id), который показывает ID процесса, который породил все свои потоки

## 2.1. `clone`
Есть syscall `clone`, который обобщает `fork`
- На самом деле, `fork` - это частный случай `clone`

`clone` — универсальный системный вызов для создания процессов и потоков. Флаги:
- `CLONE_VM` — общее адресное пространство.
- `CLONE_FS` — общая информация о файловой системе.
- `CLONE_FILES` — общие файловые дескрипторы.
- `CLONE_SIGHAND` — общие обработчики сигналов.
- `CLONE_THREAD` — поток в той же группе потоков.

Если сделать `strace` на прошлую программу, увидим, что был вызван `clone3` со множеством флагов.
А ДО ЭТОГО мы создали стек через `mmap` и передали его в `clone3` вторым аргументом (на самом деле, конец стека - стек растет вверх).
НО САМОЕ ГЛАВНОЕ — это флаг `CLONE_THREAD` - который говорит OS, что планируется создать thread (т.е. ребенок добавляется в одну thread-группу, разделяет один PID (на самом деле, PID другой, но `getpid` возвращает `TGID`) с родителем) (если хотим сделать `THREAD`, то обязательно копировать `VM` + `SIGHAND`).

**Пример создания потока через `clone`:**
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

### 2.1.1. `tgkill`
`tgkill` отправляет сигнал конкретному потоку в группе.
```cpp
syscall(SYS_tgkill, tgid, tid, sig);
```
где `tgid` — thread group ID (PID процесса), `tid` — thread ID (возвращается `gettid`).

# 3. Реализация `std::thread`

```cpp
#include <pthread.h>
#include <memory>
#include <stdexcept>

class Thread {
    pthread_t thread;
    bool joined;
    
    static void* starter(void* arg) {
        auto func = static_cast<std::function<void()>*>(arg);
        (*func)();
        delete func;
        return nullptr;
    }
    
public:
    template<typename Callable>
    Thread(Callable func) : joined(false) {
        auto func_ptr = new std::function<void()>(func);
        if (pthread_create(&thread, nullptr, starter, func_ptr) != 0) {
            delete func_ptr;
            throw std::runtime_error("pthread_create failed");
        }
    }
    
    void join() {
        if (!joined) {
            pthread_join(thread, nullptr);
            joined = true;
        }
    }
    
    ~Thread() {
        if (!joined) {
            pthread_detach(thread);
        }
    }
};
```

# 4. POSIX Thread'ы
Thread'ы в C++ появились только в 11 стандарте.
`clone` - это не POSIX.

Есть также POSIX Threads - стандарт того, как должны быть реализованы потоки
- На Linux'е они тоже появились, и отныне есть более высокоуровневые методы создания потоков
- Это уже не сисколлы, это функции библиотеки

```bash
man 7 pthreads
```
- Все функции имеют вид `pthread_*`.

**Пример умножения матриц с использованием `pthread`:**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define N 100
#define THREADS 4

int A[N][N], B[N][N], C[N][N];

typedef struct {
    int start;
    int end;
} thread_data;

void* multiply(void* arg) {
    thread_data* data = (thread_data*)arg;
    for (int i = data->start; i < data->end; ++i) {
        for (int j = 0; j < N; ++j) {
            C[i][j] = 0;
            for (int k = 0; k < N; ++k) {
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
    return NULL;
}

int main() {
    // инициализация A, B
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            A[i][j] = rand() % 10;
            B[i][j] = rand() % 10;
        }
    }
    
    pthread_t threads[THREADS];
    thread_data data[THREADS];
    int chunk = N / THREADS;
    
    for (int t = 0; t < THREADS; ++t) {
        data[t].start = t * chunk;
        data[t].end = (t == THREADS-1) ? N : (t+1) * chunk;
        pthread_create(&threads[t], NULL, multiply, &data[t]);
    }
    
    for (int t = 0; t < THREADS; ++t) {
        pthread_join(threads[t], NULL);
    }
    
    printf("Multiplication done\n");
    return 0;
}
```

# 5. Синхронизации. Mutex

**Вернемся к раннему примеру:**
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

	return 0;
}
```

Функции не потокобезопасные (thread-safe(ty)). То есть ее нельзя выполнять из нескольких потоков. Самый безопасный способ сделать код более-менее потокобезопасным — это сделать переменные способными к блокировкам

`Mutex` (mutual exclusion) - объект, который имеет 2 метода: `lock`, `unlock`

**Псевдо-решение проблемы:**
```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>

std::vector<int> v;

// Mutual exclusion
std::mutex m;

void f() {
	m.lock();  // Занимаем
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		v.push_back(i);
	}
	m.unlock();  // Освобождаеи
}

int main() {
	std::thread t(f);

	m.lock();  // Если свободно, то занимаем. Иначе ждем, пока освободится и тут же занимаем
	for (int i = 0; i < 1'000'000; ++i) {
		v.push_back(i);
	}
	m.unlock();

	t.join();  // Не забываем
	std::cout << v.size();   // 2'000'000
	
	return 0;
}
```

Вообще, это неправильно. Если будет исключение или еще что, или просто забудем написать `m.unlock();`, то будет грустно.

Вспоминаем RAII. Есть RAII структура - `std::lock_guard<T>`
- Требует у типа `lock` и `unlock`, у самого есть только конструктор и деструктор
Есть также `std::unique_lock` и `std::shared_lock` - там уже есть `unlock`.

**Решение проблемы:**
```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>

std::vector<int> v;

// Mutual exclusion
std::mutex m;

void f() {
	std::lock_guard<std::mutex> lg(m);  // требует lock и unlock, у самого есть только конструктор и деструктор
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		v.push_back(i);
	}
}

int main() {
	std::thread t(f);

	m.lock();
	for (int i = 0; i < 1'000'000; ++i) {
		v.push_back(i);
	}
	m.unlock();

	t.join();
	std::cout << v.size();   // 2'000'000
	
	return 0;
}
```

## 5.1. Dead lock
Это ситуация в, когда два или более процесса/потока бесконечно ждут друг друга, так как каждый захватил ресурс, необходимый другому, и ни один не может завершить свою работу.

В примере ниже с небольшой вероятностью будем бесконечно долго ждать
- В этом случае: `main` заблочит `m1`, `f` заблочит `m2`. Далее `main` будет ждать `m2`, а `f` — `m1`

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>

std::vector<int> v;

// Mutual exclusion
std::mutex m1;
std::mutex m2;

void f() {
	std::unique_guard<std::mutex> lg(m2);
	std::unique_guard<std::mutex> lg2(m1);
	for (int i = 1'000'000; i < 2'000'000; ++i) {
		v.push_back(i);
	}
}

int main() {
	std::thread t(f);

	m1.lock();
	m2.lock();
	for (int i = 0; i < 1'000'000; ++i) {
		v.push_back(i);
	}
	m1.unlock();
	m2.unlock();

	t.join();
	std::cout << v.size();   // 2'000'000
	
	return 0;
}
```

**Решение** — `std::lock(lockable_1, lockable_2, ...)`.
`std::lock` использует алгоритм избежания deadlock (например, алгоритм Дейкстры). Все мьютексы блокируются атомарно.
```cpp
std::unique_lock<std::mutex> lock1(m1, std::defer_lock);
std::unique_lock<std::mutex> lock2(m2, std::defer_lock);
std::lock(lock1, lock2);
```

## 5.2. Реализация `std::mutex`
Вкраце, с точки зрения syscall'ов.

**Раньше mutex реализовывались посредством семафоров:**
```bash
man 2 semop
```
Допускает исполнение не более чем $N$ потоков, где $N$ задается пользователем. То есть более гибкая настройка. Однако это устарело.

**Сейчас же используется syscall `futex` - fast userspace mutex**
`futex` (fast userspace mutex) — системный вызов для реализации примитивов синхронизации. Он работает в пользовательском пространстве, пока нет contention, и переходит в ядро при необходимости ожидания.

```bash
man 2 futex
```

**В libc обертки `futex` нет, поэтому приходится вызывать syscall явно:**
```cpp
syscall(SYS_futex, ...);
```

**Базовая реализация mutex через `SYS_FUTEX`:**
```cpp
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <atomic>
#include <climits>

class FutexMutex {
    std::atomic<int> val; // 0 - свободен, 1 - занят
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

Вся магия в том, что `mutex` потокобезопасный и, пока мы будем проверять 0 или 1 и ставить новое значение, нас никто не обгонит, заключается в `std::atomic` (а он работает благодаря существованию специальной инструкции у процессора, позволяющая в рамках ВСЕГО ОДНОЙ процессорной операции исполнить **обмен** значений (считать + заменить))

`std::atomic` гарантирует атомарность операций через процессорные инструкции (например, `lock cmpxchg` на x86). Это основа для lock-free алгоритмов.
