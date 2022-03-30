# Лекция 18

Дебаг (debugger) — компилятор генерирует программу в таком формате, который проще всего отлаживать;

Дополняется код, чтобы мы в интерфейсе дебаггера видели, в каком порядке у нас происходит работа с переменными и много лишних действий, медленно

## Как можно сделать так, чтобы программа быстрее?

1. Собирать программу в режиме *Release (далее — релиз)*
    1. В консоли добавить ключ “-r”; в CLion: добавляется в настройках проекта — Build, Execution, Deployment → CMake → ➕ (add) профиль Release, далее можно наверху в проекте запускать бинарный файл Release;
    2. В рели включаются оптимизации компилятора, на порядок быстрее, чем в дебаге
    3. Компиляторы умеют:
        - Убирать вычисление выражения, которое не используется в программе, если “снаружи” нет разницы между кодом с этими выражениями или нет
        - Удалить векту, если она приводит к UB, то есть в ней может происходить все, что угодно — “ничего не делать” равносильно UB;
        - Переставлять инструкции местами, если компилятор считает, что они независимы по данным (перестановка инструкций не меняет внешние проявления кода в сравнении с кодом до перестановки)
        - Применять векторизованные оптимизации процессора (для операций с большим количеством даблов);
            
            Например, если нужно посчитать скалярное произведение векторов `a` и `b`, то компилятор может за одну операцию процессора перемножить первые 15 (условно, зависит от версии инструкций) элементов векторов 
            
        - Удалять копирования, если они не нужны и много чего другого
    4. Производительность проекта замеряем в релизе
    5. Тестируем на настоящих данных, иначе компилятор может соптимизировать синтетический тест;

Научимся измерять производительность кода

### Namespace `std::chrono`

Добавим header `<chrono>`

Внутри лежат классы, структуры, литералы — таймеры для работы со временем:

- `std::chrono::system_clock` — системный таймер;
    - Недостаток: может переводить время назад (например, при изменении часового пояса на компьютере);
- `std::chrono::steady_clock` — монотонный таймер, никогда не убывает;
    - Чаще всего, реализация — время с загрузки ОС;
- `std::chrono::high_resolution_clock` — использует наименьший измеряемый процессором интервал;
    - Платформенно зависимый код (нельзя использовать в портируемых программах);
- std::chrono::timepoint — момент во времени;
- std::chrono::duration — продолжительность в единицах времени;

Создадим вектор типа `int` заполним его большим количеством чисел и 

```cpp
#include <chrono>

int main() {
    std::vector<int> values;
	
    for (size_t i = 0; i < 100000000; ++i) {
        values.push_back(i);
    }
	
	 
    std::cout << std::accumulate(values.begin(), values.end(), 0) << std::endl; // функция, которая применяет к начальному значению (в нашем случае - 0) оператор + с элементами вектора
    return 0;
}
```

Измерим, сколько работает этот код

```cpp
#include <chrono>

int main() {
    std::vector<int> values;
	
    auto start = std::chrono::steady_clock::now();
    for (size_t i = 0; i < 100000000; ++i) {
        values.push_back(i);
    }
    auto end = std::chrono::steady_clock::now();
		
    std::cout << (end - start).count() << std::endl;  // так получаем продолжительность кода, метод .count() возвращает количество условных единиц duration
	
    std::cout << std::accumulate(values.begin(), values.end(), 0) << std::endl; // функция, которая применяет к начальному значению (в нашем случае - 0) оператор + с элементами вектора
    return 0;
}
```

`std::chrono::steady_clock::now()` возвращает шаблонизированную структуру `std::chrono::time_point,` которая параметризована типом таймера и количеством прошедшего времени:

```cpp
template<
    class Clock,
    class Duration = typename Clock::duration
> class time_point;
```

### std::chrono::duration

Поймем, как представлен `std::chrono::duration`

Это шаблонная структура с двумя параметрами — тип данных, в которых хранятся единицы времени, и структура `std::ratio<>` — представляет простую дробь, имеет числитель и знаменатель

```cpp

int main() {
    std::vector<int> values;
	
    auto start = std::chrono::steady_clock::now();
    for (size_t i = 0; i < 100000000; ++i) {
        values.push_back(i);
    }
    auto end = std::chrono::steady_clock::now();
		
    //std::ratio<1, 1> - секунда, числитель 1, знаменатель 1
    //std::ratio<1, 1000> - миллисекунда, числитель 1, знаменатель 1000
    //std::ratio<1, 1000000> - микросекунда, числитель 1, знаменатель 1000000
	
    //std::chrono::duration<int64_t, std::milli> elapsed = end - start; std::milli - то же самое , что и std::ratio<1, 1000>
    //выдаст ошибку, так как при преобразовании дробей с большим знаменателем в дроби с меньшим знаменателем дает большую погрешность
	
    std::chrono::duration<int64_t, std::nano> elapsed = end - start;
	
    std::cout << elapsed.count() << "ns" << std::endl;  // 
	 
	  
    std::cout << std::accumulate(values.begin(), values.end(), 0) << std::endl;
    return 0;
}
```

Для удобства напишем класс `Timer`

```cpp
class Timer {
public:
    Timer(std::string message) : start_(std::chrono::steady_clock::now()), message_(std::move(std::string message_)) {}
	// передаем по значению, так как все равно хотим скопировать в приватное поле message_
	// если передачем по значению, то используем std::move. если передадут const std::string& на строку, то она скопируется в конструкторе
	// и std::move копию за-move-ит, а если временный объект, то сразу за-move-ится в message

    ~Timer() {
        auto end = std::chrono::steady_clock:::now();
        std::chrono:duration<int64_t, std::nano> elapsed = end - start;
        std::cout << message_ << " " << elapsed.count() << " ns" << std::endl;
    }

private:
	std::chrono::steady_clock::time_point start_;
	std::string message_;
};
```

Основная функция теперь выглядит так:

```cpp
using namespace std::chrono_literals;

int main() {
    std::vector<int> values;

    {
        Timer timer("for");
        for (size_t i = 0; i < 100000000; ++i) {
            values.push_back(i);
            std::this_thread::sleep_for(10ms); // из namespace std::chrono_literals;
            // или так: std::this_thread::sleep_until(std::chrono::steady_clock::now() + 10ms);
        }
    }

    std::cout << std::accumulate(values.begin(), values.end(), 0) << std::endl;
    return 0;
}

// Вывод:
// for 10071346921 ns
// 499500
```

## Асинхронность и многопоточность

В какой-то момент процессоры перестали увеличивать частоту (из-за физических ограничений), а стали наращивать количество ядер.

> По сути, ядро процессора —- это отдельный мини-процессор.
> 

Друг с другом ядра одного процессора объединены общим корпусом и кэшем.  

- Для процессора оперативная память очень медленная. Для многих вычислений достаточно только, например, регистров в процессоре, а данные в оперативной памяти меняются не так часто.
Поэтому у процессоров есть регистры, затем идут L1-кэш (маленькое количество памяти, находящееся на ядре процессора, к которому очень быстрый доступ), после L2-кэш (размер памяти чуть побольше, но с более медленным доступом, чем у L1-кэша) и, наконец, L3-кэш (ещё больше памяти и медленней доступ к ней, но всё ещё быстрее, чем к оперативной памяти). Процессор работает с данными не напрямую, а через кэш.
- Hyperthreading — например, деталей, которые отвечают за вычисления 4 комплекта, а деталей, которые отвечают за хранение текущего состояния в два раза больше. Большую часть времени занимают не вычисления, а перекладывание данных из памяти в кэш, из кэша в регистр и т.д.  У ядра с hyperthreading одна часть может заниматься вычислениями, а другая —- вводом-выводом в память.

Иногда задачу можно разделить на независимые подзадачи и работать с ними параллельно (например, в изображении обрабатывать каждый пиксель отдельно).

Обычно программы выполняются синхронно (строчка за строчкой). Для того, чтобы запустить асинхронную задачу, в стандартной библиотеке есть функция `std::async()`.

```cpp
using namespace std::chrono_literals;

int Calc(int x) {
    int result = x * x;
    std::cout << result << std::endl;
    std::this_thread::sleep_for(1s);
    return result;
}

int main() {

    auto future = std::async(Calc, 10); // передаём указатель на функцию и набор параметров для неё
    // std::future<int> future = std::async(Calc, 10); // функция std::async возвращается специальный объект, который имеет тип std::future
	
    // future.wait() - метод, который "говорит дождаться", пока во future не положат результата
    // future.get() - позволяет получить результат, который лежит во future
    // Оба метода являются "блокирующими", то есть программа будет ждать, пока асинхронная функция не закончит свои вычисления и не положит результат во future.

    std::cout << "Result: " << future.get() << std::endl;
    return 0;
}

// Вывод:
// Result: 100
// 100
```

`std::future` — что-то вроде указателя на будущий результат, который будет получен в результате выполнения асинхронной функции. Параметризован типом возвращаемого значения.

<aside>
💡 `std::future` может принимать в качестве параметра и void

</aside>

Добавим параллелизма: 

```cpp
using namespace std::chrono_literals;

int Calc(int x) {
    int result = x * x;
    return result;
}

int main() {

    std::future<int> future1 = std::async(Calc, 10);
    std::future<int> future2 = std::async(Calc, 20);

    std::cout << "Result: " << future1.get() << std::endl;
    std::cout << "Result: " << future2.get() << std::endl;

    return 0;
}

// Вывод:
// Result: 100
// Result: 400
```

Вычисления `Calc(10)` и `Calc(20)` будут проходить параллельно. `std::future` нельзя копировать, но можно, например, сложить в вектор. 

```cpp
using namespace std::chrono_literals;

int main() {

    // с помощью future можно достать будущий результат
    // а с помощью promise этот результат можно положить
    std::promise<int> promise;
    auto future = promise.get_future();

    // wait_for позволяет ждать результата определённое время
    if (future.wait_for(1s) == std::future_status::timeout) {
        std::cout << "Timeout" << std::endl;
    }
		
    // устанавливает результат в future 
    promise.set_value(100);

    if (future.wait_for(1s) == std::future_status::ready) {
        std::cout << "Ready" << future.get() << std::endl;
    }

    return 0;
}

// Вывод:
// Timeout
// Ready 100
```

Здесь мы не создавали потоки явно (всё могло выполниться и в одном потоке).

Потоки можно создавать следующим образом, напрямую:

```cpp
using namespace std::chrono_literals;

int main() {
    std::cout << std::thread::hardware_concurrency() << std::endl;
    // выводит физическое количество потоков (ядер)
		
    std::thread thread1([]() {
        for (size_t i = 0; i < 10; ++i) {
            std::this_thread::sleep_for(100ms);
            std::cout << "Thread 1: " << i << std::endl;
        }
    });

    std::thread thread2([]() {
        for (size_t i = 0; i < 10; ++i) {
            std::this_thread::sleep_for(200ms);
            std::cout << "Thread 2: " << i << std::endl;
        }
    });

    thread1.join();
    thread2.join();
    // join() "заставляет дождаться", пока thread1 и thread2 не закончат работу, и позволяет основному потоку увидеть изменения данных в вызванных потоках

    return 0;
}
```

Сделаем так, чтобы два потока изменяли общий участок памяти:

```cpp
using namespace std::chrono_literals;

int TOTAL_ITERATIONS = 0;

int main() {

    std::thread thread1([]() {
        for (size_t i = 0; i < 10; ++i) {
            ++TOTAL_ITERATIONS;
        }
    });

    std::thread thread2([]() {
        for (size_t i = 0; i < 10; ++i) {
            ++TOTAL_ITERATIONS;
        }
    });

    thread1.join();
    thread2.join();

    std::cout << TOTAL_ITERATIONS << std::endl;

    return 0;
}

// Вывод:
// 20
```

Вывод верный, но на самом деле код написан **неправильно**!

> Потоки `thread1` и `thread2` могут не увидеть изменения, сделанные в другом потоке.
> 

Такая ситуация называется гонкой (race condition) и возникает именно при работе нескольких потоков с одной областью памяти. 

Допустим, у нас есть переменная со значением 100. Один поток должен прибавить к нему 10, а второй —- вычесть 20. Может случиться так, что второй поток “вклинится” между началом и концом работы первого потока (это возможно, так как наши операции не атомарные, а занимают какое-то время). Результатом работы первого потока будет 90, второго — 80. Во-первых, ни один из этих результатов не является верным. Во-вторых, мы даже не можем гарантировать, какое итоговое значение получим в конце.

### Что можно сделать?

1.  Объявить переменную `TOTAL_ITERATIONS` как `atomic`:  `std::atomic<int> TOTAL_ITERATIONS = 0`.
    
    Тогда в одни операции с `TOTAL_ITERATIONS` не могут “вклиниться” другие операции.  `atomic` подходит, когда нужно сделать какой-то счётчик или ещё какой-то примитивный тип.
    
2.  Использовать `std::mutex`. Он позволяет обеспечить упорядоченный доступ к `TOTAL_ITERATIONS` (в один момент времени доступ к `TOTAL_ITERATIONS` будет иметь только один поток).
    
    ```cpp
    int TOTAL_ITERATIONS = 0;
    std::mutex MUTEX; // должен быть доступен во всех потоках
    ...
    int main() {
        std::thread thread1([]() {
            MUTEX.lock();
            ...
            MUTEX.unlock(); // нужно не забыть позвать unlock() в этом же потоке, иначе получим deadlock (когда ни один поток не может "захватить блокировку")
            // это может произойти и из-за вызова исключения или return
        });
        ...
        std::thread thread2([]() {
            MUTEX.lock(); // если первый поток уже начал и ещё не закончил выполнять свои операции, второй поток не сможет начать выполнять свои
            ...
    	    MUTEX.unlock();
    	});
        ...
    }
    ```
    
    При этом нет никакой гарантии, какой поток “выполнится” первым. 
    
    Для избегания deadlock есть `std::lock_guard` 
    
    ```cpp
    ...
    std::thread thread1([]() {
        std::lock_guard lock(MUTEX); // в конструкторе зовёт Mutex.lock(), в деструкторе зовёт Mutex.unlock()
        ...
    });
    ...
    ```