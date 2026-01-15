# 1. SIMD (Single Instruction Multiple Data)
С точки зрения ассемблера, у чисел с плавающей точкой другая арифметика, им нужны отдельные инструкции (про `add` и `mov` можно забыть).
Для них заведены отдельные регистры: `xmm0`, `xmm1`, ..., `xmm15` (см. page 22 в стандарте ABI).

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
        movss   xmm0, DWORD PTR .LC0[rip]  ; в LC0 лежит константа 3.14
        movss   DWORD PTR [rbp-4], xmm0    ; новый суффикс - scalar single
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

Для `double`'ов то же самое, но суффикс инструкций не `ss`, а `sd`.

## 1.1. Векторные регистры и интринсики
Вообще говоря, регистры `xmm` обладают мощной особенностью, о которой сейчас пойдет речь.
Раньше для оперирования с `float`'ами были регистры `mmx{0-7}`. Только потом, когда процессоры стали 64-битными, уже появились `xmm`. Однако у современных процессоров эти регистры 128-битные.
- Референс: Intel Intrinsics Guide.

Сейчас существуют целое семейство инструкций, позволяющее оперировать с 128 битами за 1 инструкцию! Это называется векторизация.
```cpp
#include <emmintrin.h>
```

Интринсики (intrinsics) — это функции, которые компилятор заменяет на соответствующие SIMD-инструкции. Они предоставляют низкоуровневый доступ к векторным операциям, оставаясь в синтаксисе C.

Иначе говоря, **интринсики** — функции C, компилируемые в SIMD-инструкции.

**Пример: скалярное произведение с SSE (использованием 128-битными инструкциями)**
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

## 1.2. Определение SIMD
SIMD (Single Instruction Multiple Data) — архитектурная особенность процессора, позволяющая выполнять одну операцию над несколькими данными одновременно. Это достигается за счёт широких регистров 128, 256, 512 бит, которые могут хранить векторы из 4, 8 или 16 значений соответственно.

**Семейства SIMD-инструкций:**
- **SSE** (Streaming SIMD Extensions) — 128 бит, регистры `xmm`.
- **AVX** (Advanced Vector Extensions) — 256 бит, регистры `ymm`.
- **AVX-512** — 512 бит, регистры `zmm`.

### 1.2.1. Пример инструкций:
- `addps` — сложение упакованных single-precision значений.
- `mulpd` — умножение упакованных double-precision значений.

## 1.3. Векторизация в компиляторах и стандартной библиотеке
Компиляторы плохо работают с подобными инструкциями. Но можно вызывать их явно (как в примере выше) или работать с уже готовыми алгоритмами, использующими это. Например, с C++17 в `algorithms` можно указывать `execution policy`. Их 4 вида: `sequenced_policy` (по умолчанию), `parallel_policy` (многопоточка), `parallel_unsequenced_policy`, `unsequenced_policy`. Иногда  `parallel_unsequenced_policy` под капотом использует векторизацию (то есть чтение и обработка не поэлементно, а пачками).
- Также с C++26 будет возможность использовать SIMD как часть стандартной библиотеки, не вызывая интринсики явно.

## 1.4. AVX (Advanced Vector Extensions)
- Современнее, чем SSE
- Позволяет оперировать уже не с 128 битами, а с 256 и даже 512
- Свой заголовочный файл `<immintrin.h>`, а компилировать обязательно с флагом `-mavx2`

```cpp
#include <immintrin.h>

// Функция с использованием AVX2 intrinsics
void vector_add_avx2(float* a, float* b, float* result, size_t n) {
	size_t i;
	for (i = 0; i + 8 <= n; i += 8) {
		// Загружаем 8 значений из массива а
		__m256 va = _mm256_loadu_ps(&a[i]);
		
		// Загружаем 8 значений из массива b
		__m256 vb = _mm256_loadu_ps(&b[i]);
		
		// Вычисляем сумму векторов
		__m256 vsum = _mm256_add_ps(va, vb);
		
		// Сохраняем результат
		_mm256_storeu_ps(&result[i], vsum);
	}
	
	// Обрабатываем оставшиеся элементы (скалярно)
	for (; i < n; ++i) {
		result[i] += a[i] + b[i];
	}
}
```

Если через `objdump` посмотрим на ассемблер, увидим, что здесь используются регистры `ymm`.
- [Source](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)

The width of the SIMD registers is increased from 128 bits to 256 bits, and renamed from XMM0–XMM7 to YMM0–YMM7 (in X86-64) mode, from XMM0–XMM15 to YMM0–YMM15). The legacy SSE instructions can still be utilized via the VEX prefix to operate on the lower 128 bits of the YMM registers.
- То есть YMM - лишь расширения регистров XMM. То есть XMM0 лежит в первых 128 битах YMM0 и тд.

А начиная с процессоров Skylake 2017, появились AVX-512. Соответственно, регистры стали называться ZMM0-ZMM15.

# 2. Branch prediction (Предсказание ветвлений)

**Начнем с примера:**
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

Без сортировки работает 26 секунд
С сортировкой - 8 секунд.

```cpp
// Без сортировки: 26 секунд
// С сортировкой:   8 секунд
if (data[c] >= 128) sum += data[c];
```

Казалось бы, ассемблерный код у обеих программ одинаковый
- [Развернутый ответ на вопрос](https://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-processing-an-unsorted-array)
- Работа процессора состоит из нескольких стадий (базовая модель - [5 стадий](https://en.wikipedia.org/wiki/Instruction_pipelining)), это много, поэтому он, одновременно работает с несколькими инструкциями: пока вторая инструкция декодируется (ID), первая фетчится (IF) и тд. А команда `jump` замедляет работу: приходится сбрасывать pipeline и считать заново. Изначально процессор не знает, какая ветка наиболее вероятная. Но в процессоре есть [Branch Predictor](https://en.wikipedia.org/wiki/Branch_predictor) - устройство, которое запоминает, какая ветка чаще всего срабатывала. Соответственно, он и выбирает, какую ветку подгружать.

## 2.1. Принцип работы конвейера (Pipeline)
На самом деле, современные процессоры используют глубокий конвейер (pipeline) из 14-19 стадий (например, в Intel Skylake). Основные стадии:
1. Fetch (IF) — загрузка инструкции.
2. Decode (ID) — декодирование.
3. Execute (EX) — выполнение.
4. Memory (MEM) — доступ к памяти.
5. Writeback (WB) — запись результата.

При неправильном предсказании ветвления (branch misprediction) конвейер сбрасывается (pipeline flush), что стоит 10-20 тактов.

![[cpu_pipeline.png|600]]

### 2.1.1. Branch Predictor
Branch Predictor — устройство, которое запоминает, какая ветка чаще всего срабатывала. Соответственно, он и выбирает, какую ветку подгружать.

Работа ПРОСТЕЙШЕГО (2-х битного) Branch Predictor'а может быть представлена в виде конечного автомата из 4-х состояний. Грубо говоря, чтобы Predictor поменял свое мнение, надо, чтобы 2 раза подряд он ошибся.
![[branch_predictor_automation.png]]

Современные же Branch Predictor'ы устроены очень тяжело и используют:
- Историю глобальных ветвлений (Global History Register).
- Таблицы предсказаний (Branch Target Buffer).
- Нейронные сети (в Apple M1).

Ситуация, когда Branch Predictor ошибается, называется **branch misprediction**. Количество таких ошибок называется **Penalty (штрафом)**.

### 2.1.2. Атрибуты `likely`/`unlikely` (C++20)
`[[likely]]` и `[[unlikely]]` — атрибуты, указывающие компилятору, какая ветка более вероятна.
```cpp
// C++20
if (condition) [[likely]] {
    // вероятная ветка
}
if (condition) [[unlikely]] {
    // маловероятная ветка
}

// GCC/Clang
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)
```
Компилятор может переупорядочить код, поместив вероятную ветку сразу после проверки, уменьшая penalty при предсказании.

# 3. Latency Numbers Every Programmer Should Know
- Начнем с "Latency Numbers Every Programmer Should Know"

| Operation                          | time, ns       | time, us   | time, ms |                             |
| ---------------------------------- | -------------- | ---------- | -------- | --------------------------- |
| L1 cache reference                 | 0.5 ns         |            |          |                             |
| Branch mispredict                  | 5 ns           |            |          |                             |
| L2 cache reference                 | 7 ns           |            |          | 14x L1 cache                |
| Mutex lock/unlock                  | 25 ns          |            |          |                             |
| Main memory reference              | 100 ns         |            |          | 20x L2 cache, 200x L1 cache |
| Compress 1K bytes with Zippy       | 3,000 ns       | 3 us       |          |                             |
| Send 1K bytes over 1 Gbps network  | 10,000 ns      | 10 us      |          |                             |
| Read 4K randomly from SSD*         | 150,000 ns     | 150 us     |          | ~1GB/sec SSD                |
| Read 1 MB sequentially from memory | 250,000 ns     | 250 us     |          |                             |
| Round trip within same datacenter  | 500,000 ns     | 500 us     |          |                             |
| Read 1 MB sequentially from SSD*   | 1,000,000 ns   | 1,000 us   | 1 ms     | ~1GB/sec SSD, 4X memory     |
| Disk seek                          | 10,000,000 ns  | 10,000 us  | 10 ms    | 20x datacenter roundtrip    |
| Read 1 MB sequentially from disk   | 20,000,000 ns  | 20,000 us  | 20 ms    | 80x memory, 20X SSD         |
| Send packet CA->Netherlands->CA    | 150,000,000 ns | 150,000 us | 150 ms   |                             |
```
1 ns = 10^-9 seconds
1 us = 10^-6 seconds = 1,000 ns
1 ms = 10^-3 seconds = 1,000 us = 1,000,000 ns
```

![[cpu_operations.png]]

Комментариев требует "Right"/"Wrong" branch of "if". Если ветка правильна отгадана, то переход займет 1-2 цикла, иначе в 10 раз больше! Тут как раз написана цена сброса pipeline.
Тут много занятного: прямой вызов функции в 1.5-2 раза быстрее, чем вызов функции через указатель.
Вызов syscall'а занимает 1000-1500 тактов.
- Но есть DSO (Direct System Call Optimization). Некоторые сисколлы могут быть вызваны без полного перехода в ядро через механизм VDSO (Virtual Dynamic Shared Object). Например, `clock_gettime` реализован в VDSO и работает из пользовательского пространства.

# 4. Кеши процессора

## 4.1. Иерархия кешей
На самом деле, помимо TLB-кеша, есть 3 уровня кешей: L1, L2, L3.
Это небольшие куски памяти на самом процессоре, позволяющие не обращаться каждый раз к оперативной памяти.
Особенность L3-кеша заключается в том, что он шарится на все ядра: пока L1 и L2 кеши на каждое ядро свои, L3 у всех общий.

| Memory        | CPU cycles | Ryzen 7 7800X3D                                                                                                                        |
| ------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| L1 read       | 3-4        | 64KB (per core) (which breaks down to 32KB of L1 Data cache and 32KB of L1 Instruction cache per core, with 8 cores total for the CPU) |
| L2 read       | 10-12      | 1 MB (per core) (8MB total / 8 cores)                                                                                                  |
| L3 read       | 30-70      | 96 MB (shared across all cores)                                                                                                        |
| Main RAM read | 100-150    |                                                                                                                                        |

| ![[cpu_caches.png\|250]] | ![[cpu_caches_crystal.webp\|400]] |
| ------------------------ | --------------------------------- |
### 4.1.1. Почему L3 общий?
- Экономия площади кристалла.
- Упрощение поддержания когерентности кэша.
- Обмен данными между ядрами без обращения к RAM.

### 4.1.2. Почему есть разделение кеша L1 на инструкционный (L1i) и кеш данных (L1d)

Это, к слову, Гарвардская модель.
В процессорном конвейере на разных стадиях одновременно происходят:
- **IF (Instruction Fetch):** Выборка следующей инструкции.
- **MEM (Memory Access):** Чтение или запись данных для текущей инструкции (например, `LOAD` или `STORE`).

Если бы был один унифицированный кеш L1, эти две операции конкурировали бы за один порт доступа, что понизило бы производительность.
Также раздельные кеши проще в проектировании. Управляющая логика для инструкций (предсказание ветвлений, предвыборка) и для данных (управление записью, когерентность) становится более независимой и локализованной.

В L2 и L3 используют более гибкую и экономичную модель Фон Неймана. После L1d и L1i поток инструкций и данных уже отфильтрован. Объединенный кеш L2 действует как буфер для промахов из обоих кешей первого уровня, эффективно используя общий объем памяти. Не нужно заранее резервировать фиксированное пространство под инструкции или данные — оно распределяется динамически. К тому же, L2 и L3 медленнее L1, поэтому одновременный доступ к ним не так критичен для конвейера. Их основная задача — уменьшить количество обращений к медленной оперативной памяти (RAM).

### 4.2. Алгоритм работы кешей
При разыменовании указателя процессор сначала ищет объект в L1. Не нашел — идет в L2. Снова не нашел — в L3. Если и там нет, то идет в оперативную память (там-то точно все есть). Когда же процессор на каком-то уровне нашел запрашиваемую память, он записывает данные в кеши всех уровней выше (вытесняя старые данные).
- Например, если объект лежал в L3, процессор после успешного поиска запишет его в L2 и L1
- А-ля Splay-дерево
- С этим связано много трюков в HFT. Если хочется уметь вызывать какую-либо функцию очень быстро, ее часто в цикле вызывают с параметром `really_do_actions=false`. Так функция всегда в L1 и кеш "держится горячим".
- Т.е. частый вызов функции с `false`-флагом держит её код в L1-iCache.
- Данные в L1-dCache.

### 4.3. Кеш-линия (Cache Line)
Есть понятие Cache Line - отрезки, по которым индексируется кеш. Размер кеш линии - 64 байта. Соответственно, обращаясь к адресу в памяти, все выравнивается до 64 байт.

Хорошей демонтрацией работы кеша выступает [Gallery of Processor Cache Effects](https://igoro.com/archive/gallery-of-processor-cache-effects/). 

#### 4.3.1. Пример 1: Эффект кеш-линии
Первый пример демонстрирует, насколько загрузка из памяти может быть дольше, чем само исполнение программы.
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

> The first loop multiplies every value in the array by 3, and the second loop multiplies only every 16-th. The second loop only does about **6% of the work** of the first loop, but on modern machines, the two for-loops take about the same time**:** **80** and **78** **ms** respectively on my machine.
> The reason why the loops take the same amount of time has to do with memory. The running time of these loops is dominated by the memory accesses to the array, not by the integer multiplications. And, as I’ll explain on Example 2, the hardware will perform the same main memory accesses for the two loops.


#### 4.3.2. Пример 2: Обход матрицы по строкам vs по столбцам
Другой пример. Инкрементируем все элементы матрицы. Сначала проходимся первично по строкам, затем — по столбцам.
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
// Row-major: быстро
for (int a = 0; a < d; ++a)
    for (int b = 0; b < d; ++b)
        A[a][b]++;

// Column-major: медленно
for (int b = 0; b < d; ++b)
    for (int a = 0; a < d; ++a)
        A[a][b]++;
```

Запуск показывает, что второй вариант (первичные - столбцы) работает медленнее, даже с учетом того, что вычисления производятся на памяти, которая во время первого обхода уже успела загрузиться в какой-то уровень кеша (L2 или L3). Если же порядок поменять, разница будет еще существеннее.
- При обходе по строкам обращение к памяти происходит последовательно, что хорошо для префетчера (предзагрузки данных). При обходе по столбцам каждый доступ — это новый кеш-промах, так как элементы находятся в разных строках (и разных кеш-линиях).

Оптимизация (`-O2`), к слову, проход по строкам делает быстрее, а по столбцам - медленнее.

Говорят, что row-major использует spatial locality, то есть принцип, подразумевающий стремление располагать данные и инструкции близко друг к другу в памяти.

# 5. Instruction-level parallelism (ILP)

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
Имеется 2 цикла. Вопрос: какой выполнится быстрее? Тесты показывают, что второй цикл выполняется в ~1.5 раза быстрее. Во втором цикле, поскольку инкрементировать нужно две разные ячейки, процессор может делать это параллельно (на одной ядре!).

Причина — out-of-order execution (внеочередное исполнение) и наличие нескольких исполнительных устройств (ALU). В первом цикле зависимость по данным (`a[0]++` дважды) ограничивает параллелизм. Во втором цикле операции независимы и могут выполняться одновременно.

## 5.1. Out-of-Order Execution
- Современные процессоры могут переупорядочивать инструкции для лучшего использования ресурсов.
- Условия: отсутствие зависимостей по данным (data hazards).
- Результат сохраняется в порядке программы благодаря reorder buffer.

**Пример неожиданного эффекта:**
```cpp
// Из-за out-of-order может показаться, что b станет 42 раньше, чем a станет 10,
// но для наблюдателя из программы порядок сохраняется.
int a = 0, b = 0;
void thread1() { a = 10; b = 42; }
void thread2() { while(b != 42); std::cout << a; } // всегда выведет 10.
```

## 5.2. Таблицы задержек инструкций
Есть [статья](https://www.agner.org/optimize/instruction_tables.pdf), в которой перечислено время, необходимое тому или иному процессору выполнить ту или иную инструкцию.
- Latency показывает, сколько тактов процессора нужно, чтобы выполнить инструкцию
- Reciprocal throughput показывает, сколько независимых инструкций такого же типа может выполняться за такт одновременно

# 6. Speculative execution (Спекулятивное исполнение)
Процессор, как уже выяснилось, старается всегда выполнять инструкции наперед.
С тем же Branch Prediction'ом, когда до конца не известно, надо ли будет выполнить инструкцию или нет, процессор уже может начать ее выполнять. Это называется [Speculative execution](https://en.wikipedia.org/wiki/Speculative_execution)

> Speculative execution is an optimization technique where a computer system performs some task that may not be needed. Work is done before it is known whether it is actually needed, so as to prevent a delay that would have to be incurred by doing the work after it is known that it is needed. If it turns out the work was not needed after all, most changes made by the work are reverted and the results are ignored.

C этим связано немало интересных фактов.
Так, "конъюнкция высчитывается лениво" становится уже не совсем верным утверждением.
```cpp
if (num != 0 && 10 / num == 3) {
	// ...
}
```
В теории, процессор сможет начать исполнять `10 / num == 3`, не проверив `num != 0`. Если это сгенерирует ему исключение, он будет держать его до тех пор, пока не убедится, что Branch Prediction был правильным.

## 6.1. Уязвимость Spectre
Есть реальный пример, когда хакеры использовали это как уязвимость. Ее нашли только в 2018 году, а пользовались более 20 лет.
Если вкраце: мы хотим прочитать значение из ячейки памяти, которая нам не принадлежит/не доступна. Мы будем заставлять читать эту ячейку памяти спекулятивно.

План: тренировать Branch Prediction на if'е, который всегда срабатывает на `true`. Затем заставляем процессор обратиться к памяти спекулятивно (процессор будет выполнять, потому что уже натренирован на `true`). Он поймет, что "обжегся", заготовит SegFault, но выдавать его не будет. Вдруг окажется, что Branch Predictor ошибся, поэтому SegFault не вызовется.

Остается продумать, как нам получить само значение. Делаем так, чтобы в if'е значение использовалось в качестве индекса. Значение поменять не получится, но обращаясь по индексу, эта ячейка (вернее, ее кеш-линия) загрузится в кеш. После всей махинации, останется посмотреть, к какой ячейке доступ окажется быстрее.
- Техника аттаки посредством сверки по времени называется [Timing attack](https://en.wikipedia.org/wiki/Timing_attack) (раннее обсуждалось)

**Меры защиты**: обновления микрокода, изоляция ядер.
Ссылки: [Spectre vulnerability](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)), [Code example](https://gist.github.com/ErikAugust/724d4a969fb2c6ae1bbd7b2a9e3d4bb6)

## 6.1. Более точное описание Spectre v1:
1. Жертва имеет код: `if (idx < array_size) { value = array[idx]; ... }`
2. Атакующий тренирует Branch Predictor, чтобы он предсказывал `true`.
3. Затем вызывает код с `idx` за пределами массива (но в пределах адресного пространства).
4. Процессор спекулятивно загружает `array[idx]` и использует его для доступа к другой области памяти (например, `probe[array[idx] * 4096]`), что загружает соответствующую кеш-линию.
5. После отката спекулятивного выполнения кеш-линия остаётся в кеше.
6. Атакующий измеряет время доступа к каждому элементу `probe` и определяет, какой элемент загружен быстрее (тот, что остался в кеше), восстанавливая значение `array[idx]`.
