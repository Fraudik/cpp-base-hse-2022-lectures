# Лекция 7 — 04.02.2022

<aside>
🧷 Константность → входит в сигнатуру метода, константный и неконстантный методы — разные;

</aside>
<br>
</br>

<aside>
🧷 Оператор — метод с более красивым синтаксисом → чтобы перегрузить оператор, надо определить соответствующий метод;

  
</aside>
<br>
</br>

<aside>
🧷 Оператор должен быть константным если ему не надо ничего менять в состоянии объекта;

</aside>
<br>
</br>

<aside>
🧷 Ключевое слово `friend` позволяет объявить метод/оператор “другом” класса и данный оператор будет иметь доступ к приватным/protected членам класса:

- `friend` пишем всегда внутри класса, иначе компилятор не поймёт, к полям какого класса мы хотим получить доступ;
</aside>

<aside>
🧷 `explicit` — ключевое слово перед оператором приведения типа, препятствует неявному приведению типа, например просто `bool b = v1`:

- приведение к bool будет работать лишь в нужных нам контекстах (например, внутри условия `if`);
</aside>

### Ограничения перегрузки операторов

- Неперегружаемые ператоры
    - Можно перегружать практически все операторы, кроме:
        - `?:` (conditional)
        - `.` (member selection)
        - `.*` (member selection with pointer-to-member)
        - `::` (scope resolution)
        - `sizeof` (object size information)
        - `typeid` (object type information)
        - `static_cast` (casting operator)
        - `const_cast` (casting operator)
        - `reinterpret_cast` (casting operator)
        - `dynamic_cast` (casting operator)
- Мы не можем создавать свои операторы, а лишь перегружать существующие;
- Мы не можем поменять порядок действий операторов (`*` всегда будет выполняться перед `+`, какие действия бы мы им не присвоили);

### `At(size_t index)` — переопределение функции доступа по индексу для класса `Vector` из лекции 6

```cpp
// ВНУТРИ КЛАССА:
// вектор из double:
double At(size_t index) { // границы не проверяем
	return values_[index];
}
// если мы сделаем метод константным, то не сработает следующий код:
v1.At(0) = 10; // присваиваем значение временному объекту...

// ВНУТРИ КЛАССА:
// перепишем метод как константный:
// возвращает ссылку на область памяти по соответствующему индексу:
double& At(size_t index) const { // границы, допустим, не проверяем
	return values_[index];
}

// хотим реализовать вне класса функционал вида:
v1.At(0) = 10;
// сейчас мы будем в ссылку на область памяти (в область) присваивать новое значение:
```

### `[]` — перегрузка оператора

```cpp
// ВНУТРИ КЛАССА:
double operator[](size_t index) {
	return values_[index];
}
double& operator[](size_t index) {
	return values_[index];
}
```

### `==` — перегрузка оператора

```cpp
// ВНУТРИ КЛАССА:
// берём на вход константную ссылку на вектор (не хотим изменять)
bool operator==(const Vector& other) const {
	return std::tie(values_, size_) == std::tie(other.values_, other.size_); // сравниваем члены нашего класса
}
```

→ По аналогии можем перегрузить операторы `<= < > >= !=` ;

→ Не удобно, так как C++ не генерирует самостоятельно симметричные операторы;

→ Надо отдельно написать 6 операторов сравнения (популярно — ссылаться на существующие операторы, например:

### `!=` — перезагрузка оператора используя существующий

```cpp
// ВНУТРИ КЛАССА:
bool operator!=(const Vector& other) const {
	return !(*this == other); // this — указатель, *this — разыименовываем
}
```

### `<=>`  — оператор starship

- Появляется в стандарте C++20;
- Возвращает специальный тип `std::strong_ordering`:
    
    ```cpp
    std::strong_ordering::less;
    std::strong_ordering::greater;
    std::strong_ordering::equal;
    std::strong_ordering::equivalent;
    ```
    
- Сравнивает размер векторов и далее их компоненты в лексикографическом порядке (как строки в словаре — размер важен в конце, если все элементы равны);
- Производит сравнение по схеме:
    
    ```cpp
    // псевдокод сравнения:
    a <=> b {
    	if (a > b) {
    		return value > 0;
    	} else if (a == b) {
    		return 0; // либо equal, либо equivalent
    	} else if (a < b) {
    		return value < 0;
    	}
    }
    ```
    
- Можно использовать синтаксис `default` :
    - Автоматически генерируются операторы сравнения `< > <= >= == !=`;
        
        ```cpp
        // ВНУТРИ КЛАССА:
        auto operator<=>(const Vector& other) const = default;
        ```
        
    - Компилятор генерирует реализацию по умолчанию → для всех полей класса в том порядке, в котором они объявлены в классе, применяет сравнение `<=>`;
    - Либо если данный оператор не определён на этом типе данных, то с помощью `default` генерируются остальные сравнения на основе уже определённых операторов сравнения/равенства;
    - Однако, если определить оператор `<=>` (”руками”, без использования `default`), то реализация `==` автоматически сгенерирована не будет, ибо этот оператор обычно можно реализовать более эффективно → реализуем самостоятельно;
    - Если определён оператор `<=>`, то под капотом выражение `a < b` будет заменено на `a <=> b < 0` (с выводом `true / false`);

→ Если нам надо определить оператор, левым операндом (аргументом) которого будет какой-то тип, который мы сами не контролируем (не наш класс, а тип из стандартной библиотеки), то мы должны написать перегрузку оператора вне класса — как функцию — и дописать `friend`  в класс;

### `+`  — перезагрузка оператора

```cpp
// хотим реализовать вне класса функционал вида:
v2 = v1 + 10; // не меняем v1, лишь прибавляем к каждому его объекту 10 и присваиваем в v2

// ВНУТРИ КЛАССА:
Vector operator+(int number) { // возвращем копию, я не ссылку
	Vector v_temp = *this;
	for (auto& value : v_temp.values_) {
		value += number;
	}
	return v_temp; // возвращаем временный объект
}
```

- Если мы внутри класса `Vector` определяем данную операцию, то это значит, что у нас всегда слева от `+`  стоит `Vector`, а справа — `int`;
- Если мы хотим реализовать `int + vector` (обратный порядок — не будет работать из коробки):
    
    ```cpp
    // ВНУТРИ КЛАССА:
    // добавляем ключевое слово friend:
    class Vector {
    
    public:
    	Vector operator+(int number) {
    		Vector v_temp = *this;
    		for (auto& value: v_temp.values_) {
    			value += number;
    		}
    		return v_temp;
    	}
    	
    	// позволяет объявить какой-то метод/оператор "другом" класса:
    	friend Vector operator+(int number, const Vector& vector);
    
    private:
    	std::vector<double> values_;
       size_t size_ = 0;
    }; 
    
    // ВНЕ КЛАССА:
    Vector operator+(int number, const Vector& vector) {
    	Vector v_temp = vector;
    	for (auto& value: v_temp.values_) {
    	// здесь values_ — приватное поле класса Vector, поэтому 
    		value += number;
    	}
    	return v_temp;
    }
    ```
    

### `++` — инкрементный оператор

- Мы имеем префиксный `++v` и постфиксный `v++`;
    
    ```cpp
    // ВНУТРИ КЛАССА:
    // сигнатура префиксного оператора:
    Vector& operator++() { // без аргументов, возвращает ссылку на вектор
    	for (auto& value : values_) {
    		value += 1;
    	}
    	return *this;
    }
    
    // ВНУТРИ КЛАССА:
    // для постфиксного оператора мы хотим реализовать тоже ссылку на вектор,
    // однако надо остерегаться выдачи ссылки на временный объект, который перестанет существовать при выходе из функции:
    Vector operator++(int) { // надо чтобы сигнатуры отличались, int никак не используется
     	Vector v_temp = *this;
    	for (auto& value : values_) {
    		value += 1;
    	}
    	return v_temp;
    	// уже не можем возвращать ссылку, возвращаем объект
    }
    ```
    
- В общем случае префиксный оператор `++` реализован более эффективно;

### Оператор приведения к другому типу

- Например, реализуем неявное приведение к типу `bool`, определяющее, пуст ли вектор:
    
    ```cpp
    // хотим реализовать вне класса функционал вида:
    if (v1) {
    	std::cout << "Not empty." << std::endl;
    }
    
    // ВНУТРИ КЛАССА:
    // не пишем тип возвращаемого значения — указываем его в названии функции: 
    operator bool() const {
    	return (size_ != 0);
    }
    
    // в дополнение мы получили ещё и функционал неявного приведения типа:
    bool b = v1; // возможно, мы такого не хотим
    
    // чтобы запретить неявное приведение к типу используем ключевое слово explicit:
    // ВНУТРИ КЛАССА:
    explicit operator bool() const {
    	return (size_ != 0);
    }
    
    // функционал вне класса будет также работать и так:
    if (!v1) {
    	std::cout << "Empty." << std::endl;
    } // не надо отдельно определять "!"
    ```
    

### `<<` — оператор вывода в поток

- Правый операнд — наш класс, левый — не контролируемый → мы должны писать оператор вне класса:
    
    ```cpp
    // ВНУТРИ КЛАССА:
    // добавляем ключевое слово friend:
    class Vector {
    public:
    	friend std::ostream& operator<<(std::ostream& stream, const Vector& vector);
    }; 
    
    // ВНЕ КЛАССА:
    std::ostream& operator<<(std::ostream& stream, const Vector& vector) {
    	for (auto& value : vector.values_) {
    		stream << value;
    	}
    	return stream;
    }
    ```
    
- Поток ввывода имеет возвращаемый тип `std::ostream`, в нашем операторе мы тоже возвращаем `stream` типа `std::ostream`, чтобы мы за ним могли “джойнить” например `std::endl` с помощью нативного оператора `<<`.