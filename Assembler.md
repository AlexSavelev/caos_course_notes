# 1. Процессор
- Вообще говоря, современные процессоры очень наворочены.
## 1.1. Архитектуры процессоров
Процессоры делятся на CISC- и RISC- процессоры:
- **CISC** (Complex Instruction Set Computing) — архитектура с большим набором сложных инструкций, каждая из которых может выполнять несколько низкоуровневых операций. Пример: x86/x86-64.
- **RISC** (Reduced Instruction Set Computing) — архитектура с сокращённым набором простых инструкций, выполняющихся за один такт. Пример: ARM, RISC-V.

Имеют разные архитектуры:
- `x86-64` - БОЛЬШИНСТВО
	- Подразумевается, что за 1 процессорную инструкцию можно соперировать над 64-битным числом
	- `AMD64` - синоним
- `ARM` - Android, iMac's

В КАЖДОМ процессоре есть свои ячейки памяти, именуемые регистрами. Процессор читает память так: по шине памяти, с помощью MMU (memory management unit) обращается к ОЗУ, значение записывается в регистр.

Также у процессора есть наборы инструкций - для каждой архитектуры список свой.

# 2. Основные инструкции x86-64
**Ассемблер** — низкоуровневый язык программирования, мнемоническое представление машинного кода.

**Первичный список:**
- `mov dest, src` — копирование.
- **Арифметика**: `add`, `sub`, `imul`, `idiv`.
- **Логические**: `and`, `or`, `xor`, `not`, `shl`/`shr`, `sal`/`sar`.

Напишем фунцию и скомпилируем её в [Godbolt](https://godbolt.org):
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

Выжимка:
```asm
sum:
        mov     eax, edi    ; a -> eax
        add     eax, esi    ; eax += b
        ret
```

`mov` - положи значение в регистр (из другого регистра или вообще из памяти).

```asm
mov dest src

; возьми то, что находится под rbp-20 и положи в edx
mov edx, DWORD PTR [rbp-20]

; DWORD PTR [rbp-20] - разыменование указателя
; DWORD - Double Word - удвоенный размер машинного слова - 2 * 16 = 32
; Если перепишем на uint64, то будет QWORD - Quad Word - 4 * 16 = 64
```

`add`, `sub`, `imul` — арифметика — `+=`, `-=`, `*=` — значение сохраняется в первый операнд.

```nasm
add eax, 5
sub DWORD PTR [rbp-12], eax
imul eax, DWORD PTR [rbp-20]
```

`inc`, `dec` - инкремент и декремент.

## 2.1. Деление и оптимизации
С `idiv` сложнее — она принимает один операнд. Делимое подразумевается в регистре `rax` (для 64-бит), а результат (частное) кладётся в `rax`, остаток — в `rdx`.
```asm
mov eax, edx   ; подготавливаем делимое
cdq            ; расширяем eax в edx:eax для знакового деления
idiv DWORD PTR [rbp-20] ; делим edx:eax на значение из памяти
```

Дело в том, что `idiv` — очень дорогая операция (латентность до 40-90 тактов). Компиляторы часто заменяют деление на константу умножением и сдвигами (магические числа).
Написав, например, `d /= 35;`,  мы увидим манипуляции со сдвигами и пр.

```asm
; Деление на 35
mov     eax, edi
mov     edx, 4908534051    ; магическая константа
imul    rax, rdx
shr     rax, 35
```

## 2.2. Побитовые сдвиги
`sal`/`sar` — сделать арифметический побитовый сдвиг влево/вправо (т.е. старший бит, отвечающий за знак, не трогается)
`shl`/`shr` — логический сдвиг (сдвигается все)

```nasm
sal DWORD PTR [rbp-8], 2   ; умножение на 4
sar DWORD PTR [rbp-8], 1   ; деление на 2 с учётом знака
```

Например, деление на 2 сделано с помощью `sal 2`+`sar 3` (учитывается знаковость `int`'а)

## 2.3. Регистры: историческая перспектива
Раньше процессоры были 16-битными с регистрами `ax`, `bx`, `cx`, `dx`.
Потом — 32-битные: `eax`, `ebx`, ... (`e`=extended).
А когда появились 64-ти битные процессоры - `rax`, ... (`r` = register)
- Однако совместимость с `eax` осталась - работая с `int`, можем работать с `eax` как первой половиной `rax`

**Список регистров:**
- **Общего назначения**: `rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, `rbp`, `rsp`, `r8`–`r15`.
  - `eax` — младшие 32 бита `rax`, `ax` — 16 бит, `al` — 8 бит.
- **Специальные**:
  - `rip` — Instruction Pointer.
  - `rflags` — регистр флагов (ZF, CF, SF, OF).
  - `xmm0`–`xmm15`, `ymm0`–`ymm15`, `zmm0`–`zmm31` — для SIMD.

## 2.4. Условные конструкции и переходы
```cpp
int sum(int a) {
	if (a > 25) {
		e++;
	}
	return a;
}
```

```nasm
	cmp DWORD PTR [rbp-24], 25
	jle .L2
	add QWORD PTR [rbp-24], 1
.L2:
	; ...
	ret

```
Есть специальный регистр — `EFLAGS` (`RFLAGS` в 64-битном режиме), в котором в одном из битов лежит результат сравнения.
Инструкция `cmp` вычитает операнды и устанавливает флаги, не сохраняя результат.
`jle` — jump less equal. Использует флаги ZF (zero) и SF (sign), установленные `cmp`.
`.L2:` — метка (буквально `label:`, используемый в `goto` на C++).

### 2.4.1. Регистр флагов `rflags`
- **ZF** (Zero Flag) = 1 если результат ноль.
- **CF** (Carry Flag) = 1 если был перенос.
- **SF** (Sign Flag) = 1 если результат отрицательный.
- **OF** (Overflow Flag) = 1 если переполнение.

### 2.4.2. Условные переходы
- `je`/`jz` (jump if equal/zero), `jne`/`jnz`.
- `jg` (знаковое >), `ja` (беззнаковое >).
- `jl`, `jge`, `jle`, `jb`, `jbe`.

## 2.5. Циклы на ассемблере
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

## 2.6. Хранение переменных
Переменные на самом деле могут лежать не на стеке (automatic storage duration), а в регистре.

До C++17 было ключевое слово `register` — подсказка компилятору положить переменную в регистр
- С C++17 `register` удалено из стандарта, но компиляторы сами отлично распределяют регистры.
- Современные компиляторы используют алгоритмы окраски графов (graph coloring) для распределения регистров. Если регистров не хватает, переменные вытесняются на стек (spilling).

**Рассмотрим код (`-std=c++11`):**
```cpp
register int i = 0;
for (; i < a; ++i) {
	register int j = 0;
	for (; j < a; ++j) {
	}
}
```
Положит в какие-то регистры (например, `edx`)

# 3. Связывание ассемблерного кода с C++
## 3.1. Линкуем assembler'ный код
```nasm
; is_prime.asm:

section .text
global is_prime

is_prime:
	cmp edi, 2
    jl .not_prime
    ; ... алгоритм проверки ...
    mov eax, 1
    ret
.not_prime:
    mov eax, 0
    ret
```

Можем использовать функцию на C++
```cpp
// prime.cpp

extern "C" int is_prime(int n);

int main() {
	// ...
}
```
`extern "C"` говорит о том, что искать функцию надо по правилам C — то есть без name mangling.

## 3.2. Компиляция и отладка
Компилировать Assembler будем через утилиту `nasm`
```bash
man nasm
nasm -h
```

```bash
nasm -f elf64 -o is_prime.o is_prime.asm

readelf -a is_prime.o
```

```bash
g++ prime.cpp is_prime.o -o prime
```

**В gdb:**
```bash
g++ -g is_prime.cpp is_prime.o -o prime
gdb a.out
```

```
disassemble is_prime   ; посмотреть ассемблерный код функции
break *0x...           ; поставить точку останова по адресу
stepi                  ; выполнить одну инструкцию (si)
info registers         ; показать регистры
```

# 4. Вызов функций: `call`, `ret`, стековый фрейм
Есть инструкция `call`

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

```nasm
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

## 4.1. Устройство стека
- `x86-64-abi-0.99.pdf` — System V standard на linuxbase.org

![[stack_table.png]]

- Стек растёт вниз (к младшим адресам).
- `rsp` (stack pointer) — указывает на вершину стека.
- `rbp` (base pointer) — указывает на начало текущего стекового фрейма.

При вызове `call` появляется новый Stack Frame определенного на этапе компиляции размера. Что происходит:
1. Адрес возврата (next instruction) кладётся на стек.
2. `rsp` уменьшается.
3. Управление передаётся на адрес функции.

В начале функции (`push rbp; mov rbp, rsp`) сохраняется предыдущий `rbp` и устанавливается новый фрейм.
- `leave` (эквивалент `mov rsp, rbp; pop rbp`) восстанавливает предыдущий фрейм.
- `ret` извлекает адрес возврата и прыгает на него.

**На стеке создается много переменных:**
```asm
; Пример функции с массивом из 1000 int (4000 байт)
func:
    push rbp
    mov rbp, rsp
    sub rsp, 4000          ; выделяем место на стеке
    ; ... работа с массивом
    leave                  ; эквивалентно mov rsp, rbp; pop rbp
    ret
```
На деле, ассемблер просто вычитает из `rsp` 4000 байт. А перед выходом выполняет `leave`, эквивалентную 2-м коммандам: `mov rsp, rbp` и `pop rbp`

**Recap:**
- `call <адрес>`
    1. `push eip` — Сохраняет адрес следующей за `call` инструкции (адрес возврата) на стек.
    2. `jmp <адрес>` — Передает управление по адресу функции.
- `push <операнд>`
	- Уменьшает `ESP` (например, на 4 байта в 32-битном режиме), затем помещает значение операнда в память по новому адресу `[ESP]`.
- `pop <операнд>`
	- Берет значение с вершины стека (`[ESP]`) и загружает его в операнд, затем увеличивает `ESP` (освобождая память).
- `leave`
	1. `mov esp, ebp` — "Очищает" локальные переменные (сбрасывает ESP на базу фрейма).
    2. `pop ebp` — Восстанавливает предыдущее значение EBP вызывающей функции.
- `ret`
    - Она обратна `call`. Берет адрес возврата с вершины стека (`pop eip`) и передает управление по этому адресу.
    - `ret <число>` (например, `ret 8`) — делает то же самое, но дополнительно увеличивает `ESP` на указанное число. Это нужно, чтобы очистить переданные в функцию аргументы со стека

### 4.1.1. Red Zone
- В System V ABI для x86-64 существует **red zone** — область памяти размером 128 байт ниже `rsp`, которую функция может использовать без явного выделения на стеке для своих нужд. Это оптимизация для листовых функций (leaf functions), которые не вызывают другие функции.

### 4.1.2. Про флаг `-fno-omit-frame-pointer`
Frame Pointer Omission (FPO).
Эта опция компилятора запрещает использовать `rbp` (`ebp`) как регистр общего назначения, сохраняя его как указатель на фрейм. Упрощает отладку, но может немного замедлить код.


## 4.2. Передача аргументов: System V ABI
**Аргументы с 7-го передаются через стек:**
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
Начинается момент, когда аргументов настолько много, что регистров уже не хватает и компилятор ставит аргументы на стек.
- В System V ABI AMD64 порядок клади аргументов в регистр согласован: Figure 3.4: Register Usage.

Также согласно таблице, `rax` и `rdx` используются как first и second return register соответственно. Если же возвращаемый объект на них не поместится, то возврат положится на стек.

**Итог:**
1. Первые 6 целочисленных/указательных аргументов: `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`.
2. Первые 8 аргументов с плавающей точкой: `xmm0`–`xmm7`.
3. Остальные аргументы передаются через стек (справа налево).
4. Возвращаемое значение: `rax` (целое), `xmm0` (float), `rdx:rax` (128-битное).

## 4.3. Возврат структур
- Если структура помещается в два регистра (до 16 байт), возвращается через `rdx:rax`.
- Иначе — вызывающий выделяет память и передаёт указатель как скрытый первый аргумент (`rdi`).

```cpp
struct Small { int a, b; };  // Возврат в rax:rdx
struct Large { int a[10]; }; // Возврат через указатель
```

## 4.4. Инструкция LEA
- `lea` (Load Effective Address) — вычисляет эффективный адрес (например, `rbp-20`) и кладёт его в регистр, **не обращаясь к памяти**.
- Часто используется для арифметики без изменения флагов: `lea eax, [rdi + rsi*2]`.

## 4.5. `-fno-omit-frame-pointer`
- Запрещает использовать `rbp` как регистр общего назначения.
- Упрощает отладку, но слегка замедляет код.

Эта опция полезна при отладке, так как сохраняет цепочку фреймов стека. Без неё компилятор может использовать `rbp` для других целей, что усложняет анализ стека.

## 4.6. Условное перемещение (CMOV)
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
`cmovle` — условный (conditional) `mov`. Позволяет избежать ветвления, что полезно для предсказания переходов.

**Все это было лишь каплей в море**. На деле инструкций в современных процессорах ОЧЕНЬ МНОГО. Поэтому в 21 веке писать на ассемблебе приходится очень редко, ибо современные компиляторы в 99.99+% случаев знают что делать куда лучше вас.

# 5. Ассемблерные вставки
GCC позволяет делать [вставки ассемблера](https://gcc.gnu.org/onlinedocs/gcc-5.5.0/gcc/Extended-Asm.html) через синтаксис:
```cpp
asm [volatile] ( AssemblerTemplate 
                 : OutputOperands 
                 [ : InputOperands
                 [ : Clobbers ] ])
```

```cpp
// This code copies `src` to `dst` and add 1 to `dst`.
int src = 1;
int dst;   

asm ("mov %1, %0\n\t"
    "add $1, %0"
    : "=r" (dst) 
    : "r" (src));

printf("%d\n", dst);
```
- Здесь: `%0`, `%1` - номера аргументов, перечисляемые далее. `r` - register. 
- `OutputOperands`: `"=r" (dst)` — `=` означает выходной операнд, `r` — регистр.
- `InputOperands`: `"r" (src)`.
- `Clobbers`: `"cc"` — флаги изменены, `"memory"` — память изменена.

- Clobbers - список регистров, которые внутри вставки могут "испортится". Это нужно компилятору для того, чтобы на определенный список регистров он не полагался и слелал нужные ему копии. Чаще всего указывается `"cc"` - condition codes - список флагов. Можно указать `"memory"`, `"r0"` и тд.
- Про `volatile`. В C++ ключевое слово позволяло переменную "меньше оптимизировать"

```cpp
int x;
x = 4;  // Этот код, возможно, не исполнится
x = 5;

volatile int y;
y = 4;  // Честно исполнит
y = 5;
```

```cpp
int a;
a = 10;  // Загрузится в регистр
std::cout << a;  // Прочтет из регистра, а не из стека!

volatile int b;
b = 10;  // Загрузится в регистр
std::cout << b;  // Прочтет снова из регистра
```

Здесь также: `volatile` запрещает компилятору переносить, удалять или кэшировать вставку. Без `volatile` компилятор может решить, что вставка не имеет побочных эффектов, и удалить её.

## 5.1. Измерение тактов процессора (RDTSC)
Пример ниже считает время, используя ассемблерную инструкцию `rdtsc`.
- Остальные методы (`std::chrono`, syscall'ы) используют куда больше накладных ресурсов
- `rdtsc` смотрит на процессорный счетчик тактов
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
> The following example demonstrates a case where you need to use the `volatile` qualifier. It uses the x86 `rdtsc` instruction, which reads the computer’s time-stamp counter. Without the `volatile` qualifier, the optimizers might assume that the `asm` block will always return the same value and therefore optimize away the second call.

# 6. Timing Attack и константное время
Этот тип атаки часто фигурирует в криптографии.
- Timing attack is a side-channel attack in which the attacker attempts to compromise a cryptosystem by analyzing the time taken to execute cryptographic algorithms

**Пример: функция сложения по модулю с ветвлением может выдавать разное время в зависимости от входных данных:**
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
Если скомпилировать по-обычному, без оптимизаций, в ассемблере будет branch. Соответственно, в зависимости от входных данных функция будет отрабавать разное количество времени. Таким образом, зная время выполнения, можно понять, большие ли `a` и `b`.
- Если же скомпилировать с оптимизациями, все равно не будет полной уверенности в branchless-факторе.

Хотим branchless-код:
- Возможно, работает дольше, чем исначальная реализация
- Но непоколебима с точки зрения Timing Attacking'а
```cpp
#include <inttypes.h>

uint32_t constant_time_add_mod(uint32_t a, uint32_t b, uint32_t mod) {
	uint32_t result, temp;
	
	asm volatile (
		"addl %2, %1\n\t"    // a + b
		"movl %1, %0\n\t"    // result = sum
		"subl %3, %1\n\t"    // sum - mod
		"cmovbl %1, %0\n\t"  // if (CF==1) result = SUM mod
		: "=&r" (result), "=&r" (temp)
		: "r" (a), "r" (b), "r" (mod)
		: "cc"
	);
	
	return result;
}
```

**Можно, кстати, обойтись без ассемблера:**
```cpp
#include <stdint.h>

uint32_t constant_time_add_mod(uint32_t a, uint32_t b, uint32_t mod) {
	uint64_t sum = (uint64_t)a + b;
	uint32_t result = (uint32_t)sum;
	uint32_t overflow = (uint32_t)(sum >> 32) | (result >= mod ? 1 : 0);
	result -= mod & -overflow; // вычитаем mod, если был перенос или result >= mod
	return result;
}
```

# 7. Атака переполнения буфера и защита

- Stack Smashing

Если переполнить буфер на стеке, можно перезаписать адрес возврата и перехватить управление (выполнение произвольного кода).


## 7.1. Защитные механизмы
1. **Stack Protector** (Stack Canary)
   - Случайное значение перед адресом возврата.
   - Проверка перед `ret`.
   - При нарушении: вызов `__stack_chk_fail` и `*** stack smashing detected ***`.
   - Флаги: `-fstack-protector` для включения, `-fno-stack-protector` для отключения.

2. **ASLR** (Address Space Layout Randomization)
   - Рандомизация адресного пространства: адреса стека, кучи, библиотек меняются при каждом запуске.
   - Отключение: `echo 0 | sudo tee /proc/sys/kernel/randomize_va_space`.

## 7.2. Пример ошибки Stack Smashing
```cpp
#include <cstring>

void vulnerable() {
	char buf[16];
	strcpy(buf, "very long string that overflows buffer"); // переполнение
}

int main() { vulnerable(); }
```
- Компиляция: `g++ -fstack-protector -o test test.cpp`
- Выполнение: `./test` → вывод: `*** stack smashing detected ***`
