# C++14

## Обзор

C++14 включает следующие новые языковые возможности:
- [binary literals](#binary-literals)
- [generic lambda expressions](#generic-lambda-expressions)
- [lambda capture initializers](#lambda-capture-initializers)
- [return type deduction](#return-type-deduction)
- [decltype(auto)](#decltypeauto)
- [relaxing constraints on constexpr functions](#relaxing-constraints-on-constexpr-functions)
- [variable templates](#variable-templates)
- [\[\[deprecated\]\] attribute](#deprecated-attribute)

C++14 включает следующие новые библиотечные возможности:
- [user-defined literals for standard library types](#user-defined-literals-for-standard-library-types)
- [compile-time integer sequences](#compile-time-integer-sequences)
- [std::make_unique](#stdmake_unique)

## Языковые возможности C++14

### Binary literals
Двоичные литералы предоставляют удобный способ представления чисел в двоичной системе счисления.
Можно разделять цифры с помощью символа `'`.
```c++
0b110 // == 6
0b1111'1111 // == 255
```

### Generic lambda expressions
В C++14 разрешается использовать спецификатор типа `auto` в списке параметров, что позволяет создавать полиморфные лямбды.
```c++
auto identity = [](auto x) { return x; };
int three = identity(3); // == 3
std::string foo = identity("foo"); // == "foo"
```

### Lambda capture initializers
Теперь можно создавать захватываемые значения в лямбдах, инициализируемые произвольными выражениями. Имя, данное захваченному значению, не обязано быть связанным с какими-либо переменными во внешней области видимости, и вводит новое имя внутри тела лямбды. Выражение инициализации вычисляется в момент создания лямбды (а не при её вызове).
```c++
int factory(int i) { return i * 10; }
auto f = [x = factory(2)] { return x; }; // возвращает 20

auto generator = [x = 0] () mutable {
  // этот код не скомпилируется без 'mutable', так как мы изменяем x при каждом вызове
  return x++;
};
auto a = generator(); // == 0
auto b = generator(); // == 1
auto c = generator(); // == 2
```

Поскольку теперь стало возможным перемещать (или передавать) значения в лямбду, которые ранее можно было захватывать только по значению или по ссылке, мы теперь можем захватывать move-only типы в лямбду по значению. Обратите внимание, что в приведённом ниже примере `p` в списке захвата `task2` слева от `=` — это новая переменная, приватная для тела лямбды, и не относится к исходному `p`.
```c++
auto p = std::make_unique<int>(1);

auto task1 = [=] { *p = 5; }; // ОШИБКА: std::unique_ptr нельзя копировать
// против
auto task2 = [p = std::move(p)] { *p = 5; }; // OK: p перемещается в объект замыкания
// исходный p становится пустым после создания task2
```

Таким образом, захват по ссылке может использовать имена, отличные от имён захваченных переменных.
```c++
auto x = 1;
auto f = [&r = x, x = x * 10] {
  ++r;
  return r + x;
};
f(); // устанавливает x в 2 и возвращает 12
```

### Return type deduction
Использование возвращаемого типа `auto` в C++14 позволяет компилятору самому попытаться вывести тип. В лямбдах теперь можно выводить возвращаемый тип с помощью `auto`, что делает возможным возврат выводимой ссылки или rvalue-ссылки.
```c++
// Вывод возвращаемого типа как `int`.
auto f(int i) {
 return i;
}
```
```c++
template <typename T>
auto& f(T& t) {
  return t;
}

// Возвращает ссылку на выводимый тип.
auto g = [](auto& x) -> auto& { return f(x); };
int y = 123;
int& z = g(y); // ссылка на `y`
```

### decltype(auto)
Спецификатор типа `decltype(auto)` также выводит тип, как и `auto`. Однако он сохраняет ссылки и квалификаторы cv при выводе возвращаемого типа, в то время как `auto` этого не делает.
```c++
const int x = 0;
auto x1 = x; // int
decltype(auto) x2 = x; // const int
int y = 0;
int& y1 = y;
auto y2 = y1; // int
decltype(auto) y3 = y1; // int&
int&& z = 0;
auto z1 = std::move(z); // int
decltype(auto) z2 = std::move(z); // int&&
```
```c++
// Примечание: особенно полезно для шаблонного кода

// Возвращаемый тип — `int`.
auto f(const int& i) {
 return i;
}

// Возвращаемый тип — `const int&`.
decltype(auto) g(const int& i) {
 return i;
}

int x = 123;
static_assert(std::is_same<const int&, decltype(f(x))>::value == 0);
static_assert(std::is_same<int, decltype(f(x))>::value == 1);
static_assert(std::is_same<const int&, decltype(g(x))>::value == 1);
```

См. также: `[decltype (C++11)](README.md#decltype)`.

### Relaxing constraints on constexpr functions
Ослабление ограничений на constexpr функции
В C++11 тела `constexpr` функций могли содержать только ограниченный набор синтаксиса, включая (но не ограничиваясь): `typedef`, `using` и единственное `return`-выражение. В C++14 набор разрешённого синтаксиса расширен и включает наиболее распространённые конструкции, такие как `if`, множественные `return`, циклы и т.д.
```c++
constexpr int factorial(int n) {
  if (n <= 1) {
    return 1;
  } else {
    return n * factorial(n - 1);
  }
}
factorial(5); // == 120
```

### Variable templates
C++14 позволяет делать шаблонными переменные:
```c++
template<class T>
constexpr T pi = T(3.1415926535897932385);
template<class T>
constexpr T e  = T(2.7182818284590452353);
```

### [\[\[deprecated\]\] attribute
C++14 вводит атрибут `[[deprecated]]`, позволяющий указать, что тот или иной блок кода (функция, класс и т.д.) является устаревшим и может вызвать предупреждение при компиляции. Если указана причина, она будет включена в предупреждение.
```c++
[[deprecated]]
void old_method();
[[deprecated("Use new_method instead")]]
void legacy_method();
```

## C++14 Библиотечные возможности

### User-defined literals for standard library types
Новые пользовательские литералы для типов стандартной библиотеки, включая встроенные литералы для `chrono` и `basic_string`. Они могут быть `constexpr`, то есть использоваться во время компиляции. Некоторые применения этих литералов включают разбор целых чисел во время компиляции, двоичные литералы и литералы мнимых чисел.
```c++
using namespace std::chrono_literals;
auto day = 24h;
day.count(); // == 24
std::chrono::duration_cast<std::chrono::minutes>(day).count(); // == 1440
```

### Compile-time integer sequences
Шаблон класса `std::integer_sequence` представляет собой последовательность целых чисел, известную во время компиляции. Существуют вспомогательные утилиты:
* `std::make_integer_sequence<T, N>` — создаёт последовательность `0, ..., N - 1` с типом `T`.
* `std::index_sequence_for<T...>` — преобразует пакет шаблонных параметров в целочисленную последовательность.
Преобразование массива в кортеж:
```c++
template<typename Array, std::size_t... I>
decltype(auto) a2t_impl(const Array& a, std::integer_sequence<std::size_t, I...>) {
  return std::make_tuple(a[I]...);
}

template<typename T, std::size_t N, typename Indices = std::make_index_sequence<N>>
decltype(auto) a2t(const std::array<T, N>& a) {
  return a2t_impl(a, Indices());
}
```

### std::make_unique
`std::make_unique` — рекомендуемый способ создания экземпляров `std::unique_ptr` по следующим причинам:
* Избегает необходимости использовать оператор `new`.
* Предотвращает дублирование кода при указании базового типа указателя.
* Обеспечивает безопасность при исключениях. Предположим, мы вызываем функцию `foo` следующим образом:
```c++
foo(std::unique_ptr<T>{new T{}}, function_that_throws(), std::unique_ptr<T>{new T{}});
```
Компилятор волен вызвать сначала `new T{}`, затем `function_that_throws()` и т.д. Поскольку мы выделили память на куче при первом создании `T`, здесь возникает утечка. С `std::make_unique` мы получаем безопасность при исключениях:
```c++
foo(std::make_unique<T>(), function_that_throws(), std::make_unique<T>());
```

См. раздел о [smart pointers (C++11)](README.md#smart-pointers) для получения дополнительной информации о `std::unique_ptr` и `std::shared_ptr`.
