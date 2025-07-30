# C++11

## Обзор

C++11 включает следующие новые языковые возможности:
- [move semantics](#move-semantics)
- [variadic templates](#variadic-templates)
- [rvalue references](#rvalue-references)
- [forwarding references](#forwarding-references)
- [initializer lists](#initializer-lists)
- [static assertions](#static-assertions)
- [auto](#auto)
- [lambda expressions](#lambda-expressions)
- [decltype](#decltype)
- [type aliases](#type-aliases)
- [nullptr](#nullptr)
- [strongly-typed enums](#strongly-typed-enums)
- [attributes](#attributes)
- [constexpr](#constexpr)
- [delegating constructors](#delegating-constructors)
- [user-defined literals](#user-defined-literals)
- [explicit virtual overrides](#explicit-virtual-overrides)
- [final specifier](#final-specifier)
- [default functions](#default-functions)
- [deleted functions](#deleted-functions)
- [range-based for loops](#range-based-for-loops)
- [special member functions for move semantics](#special-member-functions-for-move-semantics)
- [converting constructors](#converting-constructors)
- [explicit conversion functions](#explicit-conversion-functions)
- [inline-namespaces](#inline-namespaces)
- [non-static data member initializers](#non-static-data-member-initializers)
- [right angle brackets](#right-angle-brackets)
- [ref-qualified member functions](#ref-qualified-member-functions)
- [trailing return types](#trailing-return-types)
- [noexcept specifier](#noexcept-specifier)
- [char32_t and char16_t](#char32_t-and-char16_t)
- [raw string literals](#raw-string-literals)

C++11 включает следующие новые библиотечные возможности:
- [std::move](#stdmove)
- [std::forward](#stdforward)
- [std::thread](#stdthread)
- [std::to_string](#stdto_string)
- [type traits](#type-traits)
- [smart pointers](#smart-pointers)
- [std::chrono](#stdchrono)
- [tuples](#tuples)
- [std::tie](#stdtie)
- [std::array](#stdarray)
- [unordered containers](#unordered-containers)
- [std::make_shared](#stdmake_shared)
- [std::ref](#stdref)
- [memory model](#memory-model)
- [std::async](#stdasync)
- [std::begin/end](#stdbeginend)

## Языковые возможности C++11

### Move semantics
Перемещение объекта означает передачу владения каким-либо управляемым им ресурсом другому объекту.

Первое преимущество семантики перемещения — оптимизация производительности. Когда объект достигает конца своего времени жизни (будь то временный объект или явный вызов `std::move`), перемещение часто является более дешёвым способом передачи ресурсов. Например, перемещение `std::vector` заключается лишь в копировании нескольких указателей и внутреннего состояния в новый вектор — копирование же потребовало бы копирования каждого элемента вектора, что дорого и излишне, если старый вектор скоро будет уничтожен.

Перемещение также позволяет некопируемым типам, таким как `std::unique_ptr` ([smart pointers](#smart-pointers)), гарантировать на уровне языка, что одновременно существует только один экземпляр управляемого ресурса, при этом позволяя передавать этот экземпляр между областями видимости.

См. разделы: [rvalue references](#rvalue-references), [special member functions for move semantics](#special-member-functions-for-move-semantics), [`std::move`](#stdmove), [`std::forward`](#stdforward), [`forwarding references`](#forwarding-references).

### Rvalue references
C++11 представляет новый тип ссылки — _rvalue reference_. Ссылка на rvalue типа `T` (где `T` — нешаблонный параметр типа, например `int` или пользовательский тип) создаётся с помощью синтаксиса `T&&`. Ссылки на rvalue привязываются только к rvalue.

Вывод типов с lvalue и rvalue:
```c++
int x = 0; // `x` — это lvalue типа `int`
int& xl = x; // `xl` — это lvalue типа `int&`
int&& xr = x; // ошибка компиляции -- `x` — это lvalue
int&& xr2 = 0; // `xr2` — это lvalue типа `int &&` -- привязывается к временному rvalue, `0`

void f(int& x) {}
void f(int&& x) {}

f(x);  // вызывает f(int&)
f(xl); // вызывает f(int&)
f(3);  // вызывает f(int&&)
f(std::move(x)); // вызывает f(int&&)

f(xr2);           // вызывает f(int&)
f(std::move(xr2)); // вызывает f(int&& x)
```

См. также: [`std::move`](#stdmove), [`std::forward`](#stdforward), [`forwarding references`](#forwarding-references).

### Forwarding references
Также известны как _universal references_. Передающая ссылка создаётся с помощью синтаксиса `T&&`, где `T` — шаблонный параметр типа, либо с использованием `auto&&`. Это позволяет реализовать _perfect forwarding_ (идеальную передачу): возможность передавать аргументы, сохраняя их исходный тип ссылки (например, lvalue остаются lvalue, временные объекты передаются как rvalue).

Передающие ссылки позволяют ссылке привязываться либо к lvalue, либо к rvalue в зависимости от типа. Они подчиняются правилам _reference collapsing_ (свертывания ссылок):
* `T& &` становится `T&`
* `T& &&` становится `T&`
* `T&& &` становится `T&`
* `T&& &&` становится `T&&`

Вывод типа `auto` с lvalue и rvalue:
```c++
int x = 0; // `x` — это lvalue типа `int`
auto&& al = x; // `al` — это lvalue типа `int&` -- привязывается к lvalue `x`
auto&& ar = 0; // `ar` — это lvalue типа `int&&` -- привязывается к временному rvalue `0`
```

Вывод шаблонного параметра типа с lvalue и rvalue:
```c++
// Начиная с C++14:
void f(auto&& t) {
  // ...
}

// Начиная с C++11:
template  <typename T >
void f(T&& t) {
  // ...
}

int x = 0;
f(0); // T — int, выводится как f(int &&) => f(int&&)
f(x); // T — int&, выводится как f(int& &&) => f(int&)

int& y = x;
f(y); // T — int&, выводится как f(int& &&) => f(int&)

int&& z = 0; // `z` — это lvalue с типом `int&&`
f(z); // T — int&, выводится как f(int& &&) => f(int&)
f(std::move(z)); // T — int, выводится как f(int &&) => f(int&&)
```

См. также: [`std::move`](#stdmove), [`std::forward`](#stdforward), [`rvalue references`](#rvalue-references).

### Variadic templates
Синтаксис `...` создаёт _parameter pack_ (пакет параметров) или раскрывает его. Пакет параметров шаблона — это параметр шаблона, принимающий ноль или более аргументов (не типов, типов или шаблонов). Шаблон, имеющий хотя бы один пакет параметров, называется _variadic template_ (вариадическим шаблоном).
```c++
template <typename... T>
struct arity {
  constexpr static int value = sizeof...(T);
};
static_assert(arity<>::value == 0);
static_assert(arity<char, short, int>::value == 3);
```

Интересное применение — создание _initializer list_ (списка инициализации) из _parameter pack_ (пакета параметров) для итерации по аргументам вариадической функции.
```c++
template <typename First, typename... Args>
auto sum(const First first, const Args... args) -> decltype(first) {
  const auto values = {first, args...};
  return std::accumulate(values.begin(), values.end(), First{0});
}

sum(1, 2, 3, 4, 5); // 15
sum(1, 2, 3);       // 6
sum(1.5, 2.0, 3.7); // 7.2
```

### Initializer lists
Лёгкий массивоподобный контейнер элементов, создаваемый с помощью синтаксиса «фигурных скобок». Например, `{ 1, 2, 3 }` создаёт последовательность целых чисел типа `std::initializer_list<int>`. Удобно использовать вместо передачи вектора объектов в функцию.
```c++
int sum(const std::initializer_list<int>& list) {
  int total = 0;
  for (auto& e : list) {
    total += e;
  }

  return total;
}

auto list = {1, 2, 3};
sum(list); // == 6
sum({1, 2, 3}); // == 6
sum({}); // == 0
```

### Static assertions
Утверждения, вычисляемые во время компиляции.
```c++
constexpr int x = 0;
constexpr int y = 1;
static_assert(x == y, "x != y");
```

### auto
Переменные, объявленные с типом `auto`, имеют тип, выводимый компилятором на основе инициализирующего выражения.
```c++
auto a = 3.14; // double
auto b = 1; // int
auto& c = b; // int&
auto d = { 0 }; // std::initializer_list<int>
auto&& e = 1; // int&&
auto&& f = b; // int&
auto g = new auto(123); // int*
const auto h = 1; // const int
auto i = 1, j = 2, k = 3; // int, int, int
auto l = 1, m = true, n = 1.61; // ошибка -- `l` выведено как int, `m` — bool
auto o; // ошибка -- `o` требует инициализатор
```

Полезно для повышения читаемости, особенно для сложных типов:
```c++
std::vector<int> v = ...;
std::vector<int>::const_iterator cit = v.cbegin();
// vs.
auto cit = v.cbegin();
```

Функции также могут выводить возвращаемый тип с помощью `auto`. В C++11 возвращаемый тип должен быть указан либо явно, либо с использованием `decltype`, как показано ниже:
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2); // == 3
add(1, 2.0); // == 3.0
add(1.5, 1.5); // == 3.0
```
Возвращаемый тип после параметров в примере выше — это _declared type_ (объявленный тип) (см. раздел [`decltype`](#decltype)) выражения `x + y`. Например, если `x` — целое, а `y` — double, то `decltype(x + y)` — тип double. Таким образом, функция выводит тип в зависимости от того, какой тип имеет выражение `x + y`. Отметим, что возвращаемый тип имеет доступ к своим параметрам и, при необходимости, к `this`.

### Lambda expressions
`lambda` — это безымянный объект-функция, способный захватывать переменные из области видимости. Состоит из: _capture list_ (списка захвата); необязательного набора параметров с необязательным возвращаемым типом после параметров; тела функции. Примеры списков захвата:
* `[]` — ничего не захватывает.
* `[=]` — захват локальных объектов (локальных переменных, параметров) по значению.
* `[&]` — захват локальных объектов по ссылке.
* `[this]` — захват `this` по ссылке.
* `[a, &b]` — захват объекта `a` по значению, `b` — по ссылке.

```c++
int x = 1;

auto getX = [=] { return x; };
getX(); // == 1

auto addX = [=](int y) { return x + y; };
addX(1); // == 2

auto getXRef = [&]() -> int& { return x; };
getXRef(); // int& на `x`
```

По умолчанию захваченные по значению переменные нельзя изменять внутри лямбды, так как генерируемый компилятором метод помечен как `const`. Ключевое слово `mutable` позволяет модифицировать захваченные переменные. Оно размещается после списка параметров (который должен присутствовать, даже если он пуст).
```c++
int x = 1;

auto f1 = [&x] { x = 2; }; // OK: x — ссылка, изменяет оригинал

auto f2 = [x] { x = 2; }; // ОШИБКА: лямбда может выполнять только const-операции над захваченным значением
// vs.
auto f3 = [x]() mutable { x = 2; }; // OK: лямбда может выполнять любые операции над захваченным значением
```

### decltype
`decltype` — это оператор, возвращающий _declared type_ (объявленный тип) выражения, переданного ему. Квалификаторы cv и ссылки сохраняются, если они являются частью выражения. Примеры использования `decltype`:
```c++
int a = 1; // `a` объявлен как тип `int`
decltype(a) b = a; // `decltype(a)` — `int`
const int& c = a; // `c` объявлен как тип `const int&`
decltype(c) d = a; // `decltype(c)` — `const int&`
decltype(123) e = 123; // `decltype(123)` — `int`
int&& f = 1; // `f` объявлен как тип `int&&`
decltype(f) g = 1; // `decltype(f)` — `int&&`
decltype((a)) h = g; // `decltype((a))` — int&
```
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2.0); // `decltype(x + y)` => `decltype(3.0)` => `double`
```

### Type aliases
По смыслу аналогичны `typedef`, однако псевдонимы с `using` легче читаются и совместимы с шаблонами.
```c++
template <typename T>
using Vec = std::vector<T>;
Vec<int> v; // std::vector<int>

using String = std::string;
String s {"foo"};
```

### nullptr
C++11 представляет новый тип нулевого указателя, предназначенный для замены макроса `NULL` из C. `nullptr` имеет тип `std::nullptr_t` и может неявно преобразовываться в типы указателей, но, в отличие от `NULL`, не преобразуется в целочисленные типы, кроме `bool`.
```c++
void foo(int);
void foo(char*);
foo(NULL); // ошибка -- неоднозначно
foo(nullptr); // вызывает foo(char*)
```

### Strongly-typed enums
Безопасные по типу перечисления, решающие ряд проблем с C-перечислениями, включая: неявные преобразования, невозможность указать базовый тип, засорение области видимости.
```c++
// Указание базового типа как `unsigned int`
enum class Color : unsigned int { Red = 0xff0000, Green = 0xff00, Blue = 0xff };
// `Red`/`Green` в `Alert` не конфликтуют с `Color`
enum class Alert : bool { Red, Green };
Color c = Color::Red;
```

### Attributes
Атрибуты предоставляют универсальный синтаксис вместо `__attribute__(...)`, `__declspec` и т.п.
```c++
// Атрибут `noreturn` указывает, что `f` не возвращает управления.
[[ noreturn ]] void f() {
  throw "error";
}
```

### constexpr
Константные выражения — это выражения, которые *могут* быть вычислены компилятором во время компиляции. Только несложные вычисления могут выполняться в константном выражении (эти правила постепенно смягчаются в более поздних версиях). Используется спецификатор `constexpr`, чтобы указать, что переменная, функция и т.д. является константным выражением.
```c++
constexpr int square(int x) {
  return x * x;
}

int square2(int x) {
  return x * x;
}

int a = square(2);  // mov DWORD PTR [rbp-4], 4

int b = square2(2); // mov edi, 2
                    // call square2(int)
                    // mov DWORD PTR [rbp-8], eax
```
В предыдущем фрагменте вычисление при вызове `square` происходит во время компиляции, и результат внедряется в генерируемый код, тогда как `square2` вызывается во время выполнения.
`constexpr` значения — это те, которые компилятор может(но не обязан), вычислить во время компиляции:
```c++
const int x = 123;
constexpr const int& y = x; // ошибка -- constexpr переменная `y` должна быть инициализирована константным выражением
```
Константные выражения с классами:
```c++
struct Complex {
  constexpr Complex(double r, double i) : re{r}, im{i} { }
  constexpr double real() { return re; }
  constexpr double imag() { return im; }

private:
  double re;
  double im;
};

constexpr Complex I(0, 1);
```

### Delegating constructors
Конструкторы теперь могут вызывать другие конструкторы в том же классе через список инициализации.
```c++
struct Foo {
  int foo;
  Foo(int foo) : foo{foo} {}
  Foo() : Foo(0) {}
};

Foo foo;
foo.foo; // == 0
```

### User-defined literals
Пользовательские литералы позволяют расширить язык и добавить собственный синтаксис. Для создания литерала требуется определить функцию `T operator "" X(...) { ... }`, возвращающую тип `T`, с именем `X`. Имя этой функции определяет имя литерала. Любые имена литералов, не начинающиеся с подчёркивания, зарезервированы и не будут вызываться. Существуют правила, какие параметры должна принимать функция пользовательского литерала, в зависимости от типа литерала.
Преобразование температуры из Цельсия в Фаренгейт:
```c++
// Требуется параметр `unsigned long long` для целочисленного литерала.
long long operator "" _celsius(unsigned long long tempCelsius) {
  return std::llround(tempCelsius * 1.8 + 32);
}
24_celsius; // == 75
```

Преобразование строки в число:
```c++
// Требуются параметры `const char*` и `std::size_t`.
int operator "" _int(const char* str, std::size_t) {
  return std::stoi(str);
}

"123"_int; // == 123, тип `int`
```

### Explicit virtual overrides
Указывает, что виртуальная функция переопределяет другую виртуальную функцию. Если виртуальная функция не переопределяет виртуальную функцию родителя, возникает ошибка компилятора.
```c++
struct A {
  virtual void foo();
  void bar();
};

struct B : A {
  void foo() override; // верно -- B::foo переопределяет A::foo
  void bar() override; // ошибка -- A::bar не виртуальна
  void baz() override; // ошибка -- B::baz не переопределяет A::baz
};
```

### Final specifier
Указывает, что виртуальная функция не может быть переопределена в производном классе, или что класс не может наследоваться.
```c++
struct A {
  virtual void foo();
};

struct B : A {
  virtual void foo() final;
};

struct C : B {
  virtual void foo(); // ошибка -- объявление 'foo' переопределяет 'final'-функцию
};
```

Класс не может наследоваться.
```c++
struct A final {};
struct B : A {}; // ошибка -- базовый класс 'A' помечен как 'final'
```

### Default functions
Эффективный способ предоставить реализацию функции по умолчанию, например, конструктора.
```c++
struct A {
  A() = default;
  A(int x) : x{x} {}
  int x {1};
};
A a; // a.x == 1
A a2 {123}; // a.x == 123
```

С наследованием:
```c++
struct B {
  B() : x{1} {}
  int x;
};

struct C : B {
  // Вызывает B::B
  C() = default;
};

C c; // c.x == 1
```

### Deleted functions
Эффективный способ запретить реализацию функции. Полезно для предотвращения копирования объектов.
```c++
class A {
  int x;

public:
  A(int x) : x{x} {};
  A(const A&) = delete;
  A& operator=(const A&) = delete;
};

A x {123};
A y = x; // ошибка -- вызов удалённого конструктора копирования
y = x; // ошибка -- оператор= удалён
```

### Range-based for loops
Синтаксический сахар для итерации по элементам контейнера.
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int& x : a) x *= 2;
// a == { 2, 4, 6, 8, 10 }
```

Отличие при использовании `int` вместо `int&`:
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int x : a) x *= 2;
// a == { 1, 2, 3, 4, 5 }
```

### Special member functions for move semantics
Конструктор копирования и оператор присваивания копирования вызываются при создании копий, а с появлением в C++11 семантики перемещения появились конструктор перемещения и оператор присваивания перемещения.
```c++
struct A {
  std::string s;
  A() : s{"test"} {}
  A(const A& o) : s{o.s} {}
  A(A&& o) : s{std::move(o.s)} {}
  A& operator=(A&& o) {
   s = std::move(o.s);
   return *this;
  }
};

A f(A a) {
  return a;
}

A a1 = f(A{}); // создано перемещением из временного rvalue
A a2 = std::move(a1); // создано перемещением с помощью std::move
A a3 = A{};
a2 = std::move(a3); // присвоение перемещением
a1 = f(A{}); // присвоение перемещением из временного rvalue
```

### Converting constructors
Преобразующие конструкторы преобразуют значения из синтаксиса фигурных скобок в аргументы конструктора.
```c++
struct A {
  A(int) {}
  A(int, int) {}
  A(int, int, int) {}
};

A a {0, 0}; // вызывает A::A(int, int)
A b(0, 0); // вызывает A::A(int, int)
A c = {0, 0}; // вызывает A::A(int, int)
A d {0, 0, 0}; // вызывает A::A(int, int, int)
```

Синтаксис фигурных скобок не допускает сужающих преобразований:
```c++
struct A {
  A(int) {}
};

A a(1.1); // OK
A b {1.1}; // Ошибка: сужающее преобразование double в int
```

Если конструктор принимает `std::initializer_list`, он будет вызван в первую очередь:
```c++
struct A {
  A(int) {}
  A(int, int) {}
  A(int, int, int) {}
  A(std::initializer_list<int>) {}
};

A a {0, 0}; // вызывает A::A(std::initializer_list<int>)
A b(0, 0); // вызывает A::A(int, int)
A c = {0, 0}; // вызывает A::A(std::initializer_list<int>)
A d {0, 0, 0}; // вызывает A::A(std::initializer_list<int>)
```

### Explicit conversion functions
Функции преобразования теперь могут быть явными с помощью спецификатора `explicit`.
```c++
struct A {
  operator bool() const { return true; }
};

struct B {
  explicit operator bool() const { return true; }
};

A a;
if (a); // OK, вызывает A::operator bool()
bool ba = a; // OK, копирующая инициализация выбирает A::operator bool()

B b;
if (b); // OK, вызывает B::operator bool()
bool bb = b; // ошибка, копирующая инициализация не учитывает B::operator bool()
```

### Inline namespaces
Все члены встроенного пространства имён рассматриваются так, будто они являются частью родительского пространства имён, что позволяет специализировать функции и упрощает процесс версионирования. Это транзитивное свойство: если A содержит B, которое в свою очередь содержит C, и B|C — встроенные пространства имён, то члены C можно использовать так, будто они находятся в A.
```c++
namespace Program {
  namespace Version1 {
    int getVersion() { return 1; }
    bool isFirstVersion() { return true; }
  }
  inline namespace Version2 {
    int getVersion() { return 2; }
  }
}

int version {Program::getVersion()};              // Использует getVersion() из Version2
int oldVersion {Program::Version1::getVersion()}; // Использует getVersion() из Version1
bool firstVersion {Program::isFirstVersion()};    // Не компилируется при добавлении Version2
```

### Non-static data member initializers
Позволяет инициализировать нестатические члены данных там, где они объявлены, очищая конструкторы от инициализаций по-умолчанию.
```c++
// Стандартная инициализация до C++11
class Human {
    Human() : age{0} {}
  private:
    unsigned age;
};
// Стандартная инициализация в C++11
class Human {
  private:
    unsigned age {0};
};
```

### Right angle brackets
C++11 теперь может определить, когда серия правых угловых скобок используется как оператор или как закрывающая скобка typedef, без необходимости добавлять пробелы.
```c++
typedef std::map<int, std::map <int, std::map <int, int> > > cpp98LongTypedef;
typedef std::map<int, std::map <int, std::map <int, int>>>   cpp11LongTypedef;
```

### Ref-qualified member functions
Члены-функции теперь могут иметь квалификатор в зависимости от того, является ли `*this` ссылкой на lvalue или rvalue.
```c++
struct Bar {
  // ...
};

struct Foo {
  Bar& getBar() & { return bar; }
  const Bar& getBar() const& { return bar; }
  Bar&& getBar() && { return std::move(bar); }
  const Bar&& getBar() const&& { return std::move(bar); }
private:
  Bar bar;
};

Foo foo{};
Bar bar = foo.getBar(); // вызывает `Bar& getBar() &`

const Foo foo2{};
Bar bar2 = foo2.getBar(); // вызывает `Bar& Foo::getBar() const&`

Foo{}.getBar(); // вызывает `Bar&& Foo::getBar() &&`
std::move(foo).getBar(); // вызывает `Bar&& Foo::getBar() &&`
std::move(foo2).getBar(); // вызывает `const Bar&& Foo::getBar() const&`
```

### Trailing return types
C++11 позволяет функциям и лямбдам использовать альтернативный синтаксис для указания возвращаемого типа.
```c++
int f() {
  return 123;
}
// vs.
auto f() -> int {
  return 123;
}
```
```c++
auto g = []() -> int {
  return 123;
};
```
Эта возможность особенно полезна, когда определение возвращаемого типа невозможно обычным способом:
```c++
// Следующий код не компилируется
template <typename T, typename U>
decltype(a + b) add(T a, U b) {
    return a + b;
}

// С помощью возвращаемого типа после параметров это возможно:
template <typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}
```
В C++14 вместо этого можно использовать [`decltype(auto) (C++14)`](README.md#decltypeauto).

### Noexcept specifier
Спецификатор `noexcept` указывает, может ли функция выбрасывать исключения. Это улучшенная версия `throw()`.
```c++
void func1() noexcept;        // не выбрасывает
void func2() noexcept(true);  // не выбрасывает
void func3() throw();         // не выбрасывает

void func4() noexcept(false); // может выбрасывать
```

Функции, не выбрасывающие исключения, могут вызывать потенциально выбрасывающие исключения функции. Если во время поиска обработчика исключение достигает внешнего блока функции, помеченной как `noexcept`, вызывается `std::terminate`.
```c++
extern void f();  // потенциально выбрасывающая
void g() noexcept {
    f();          // допустимо, даже если f выбрасывает
    throw 42;     // допустимо, фактически вызов std::terminate
}
```

### char32_t and char16_t
Предоставляет стандартные типы для представления UTF-8 строк.
```c++
char32_t utf8_str[] = U"\u0123";
char16_t utf8_str[] = u"\u0123";
```

### Raw string literals
C++11 представляет новый способ объявления строковых литералов — «сырые строковые литералы». Символы, входящие в escape-последовательности (табуляции, переводы строк, одиночные обратные слэши и т.д.), можно вводить «как есть», сохраняя форматирование.
Сырой строковой литерал объявляется с помощью следующего синтаксиса:
```
R"delimiter(raw_characters)delimiter"
```
где:
* `delimiter` — необязательная последовательность символов, состоящая из любых символов, кроме скобок, обратных слэшей и пробелов.
* `raw_characters` — любая последовательность символов; не должна содержать закрывающую последовательность `")delimiter"`.
Пример:
```cpp
// msg1 и msg2 эквивалентны.
const char* msg1 = "\nHello,\n\tworld!\n";
const char* msg2 = R"(
Hello,
    world!
)";
```

## Библиотечные возможности C++11

### std::move
`std::move` указывает, что объект, переданный в него, может передать свои ресурсы.
Определение `std::move` (перемещение — это приведение к ссылке на rvalue):
```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& arg) {
  return static_cast<typename remove_reference<T>::type&&>(arg);
}
```

Передача `std::unique_ptr`:
```c++
std::unique_ptr<int> p1 {new int{0}};  // на практике используй std::make_unique
std::unique_ptr<int> p2 = p1; // ошибка -- нельзя копировать unique pointers
std::unique_ptr<int> p3 = std::move(p1); // перемещаем `p1` в `p3`
                                         // теперь опасно разыменовывать объект, которым владеет `p1`
```

### std::forward
Возвращает переданные аргументы, сохраняя их категорию значения и квалификаторы cv. Полезно для шаблонного кода и фабрик. Используется вместе с [`forwarding references`](#forwarding-references).

Определение `std::forward`:
```c++
template <typename T>
T&& forward(typename remove_reference<T>::type& arg) {
  return static_cast<T&&>(arg);
}
```

Пример функции `wrapper`, которая просто перенаправляет объекты `A` в конструктор копирования или перемещения нового объекта `A`:
```c++
struct A {
  A() = default;
  A(const A& o) { std::cout  <<  "copied "  << std::endl; }
  A(A&& o) { std::cout  <<  "moved "  << std::endl; }
};

template  <typename T >
A wrapper(T&& arg) {
  return A{std::forward <T>(arg)};
}

wrapper(A{}); // moved
A a;
wrapper(a); // copied
wrapper(std::move(a)); // moved
```

См. также: [`forwarding references`](#forwarding-references), [`rvalue references`](#rvalue-references).

### std::thread
`std::thread` предоставляет стандартный способ управления потоками, например, создание и завершение. В примере ниже создаются несколько потоков для выполнения разных вычислений, после чего программа ждёт завершения всех потоков.
```c++
void foo(bool clause) { /* working */ }

std::vector<std::thread> threadsVector;
threadsVector.emplace_back([]() {
  // Лямбда-функция, которая будет вызвана
});
threadsVector.emplace_back(foo, true);  // поток запустит foo(true)
for (auto& thread : threadsVector) {
  thread.join(); // Ждать завершения потоков
}
```

### std::to_string
Преобразует числовое значение в `std::string`.
```c++
std::to_string(1.2); // == "1.2"
std::to_string(123); // == "123"
```

### Type traits
Type traits определяют шаблонный интерфейс времени компиляции для запроса или изменения свойств типов.
```c++
static_assert(std::is_integral<int>::value);
static_assert(std::is_same<int, int>::value);
static_assert(std::is_same<std::conditional<true, int, double>::type, int>::value);
```

### Smart pointers
C++11 представляет новые умные указатели: `std::unique_ptr`, `std::shared_ptr`, `std::weak_ptr`. `std::auto_ptr` становится устаревшим и будет удален в C++17.

`std::unique_ptr` — некопируемый, перемещаемый указатель, управляющий своей памятью в куче. **Примечание: рекомендуется использовать вспомогательные функции `std::make_X`, а не конструкторы.**
```c++
std::unique_ptr<Foo> p1 { new Foo{} };  // `p1` владеет `Foo`
if (p1) {
  p1->bar();
}

{
  std::unique_ptr<Foo> p2 {std::move(p1)};  // Теперь `p2` владеет `Foo`
  f(*p2);

  p1 = std::move(p2);  // Владение возвращается к `p1` -- `p2` уничтожается
}

if (p1) {
  p1->bar();
}
// Экземпляр `Foo` уничтожается, когда `p1` выходит из области видимости
```

`std::shared_ptr` — умный указатель, управляющий ресурсом, общим между несколькими владельцами. У shared pointer содержит _control block_ (блок управления) который содержит, среди прочего, управляемый объект и счётчик ссылок. Доступ ко всему блоку управления потокобезопасен, однако работа с самим управляемым объектом — нет.
```c++
void foo(std::shared_ptr<T> t) {
  // Делать что-то с `t`...
}

void bar(std::shared_ptr<T> t) {
  // Делать что-то с `t`...
}

void baz(std::shared_ptr<T> t) {
  // Делать что-то с `t`...
}

std::shared_ptr<T> p1 {new T{}};

foo(p1);
bar(p1);
baz(p1);
```

### std::chrono
Библиотека chrono содержит набор утилит и типов для работы с _clocks_ (длительностями), _clocks_ (часами) и _time points_ (моментами времени). Одно из применений — замер производительности кода:
```c++
std::chrono::time_point<std::chrono::steady_clock> start, end;
start = std::chrono::steady_clock::now();
// Какие-то вычисления...
end = std::chrono::steady_clock::now();

std::chrono::duration<double> elapsed_seconds = end - start;
double t = elapsed_seconds.count(); // t — количество секунд, представленное как `double`
```

### Tuples
Кортежи — это коллекция фиксированного размера с разнородными значениями. Доступ к элементам `std::tuple` осуществляется через распаковку с помощью [`std::tie`](#stdtie) или с помощью `std::get`.
```c++
// `playerProfile` имеет тип `std::tuple <int, const char*, const char* >`.
auto playerProfile = std::make_tuple(51,  "Frans Nielsen ",  "NYI ");
std::get<0>(playerProfile); // 51
std::get<1>(playerProfile); //  "Frans Nielsen "
std::get<2>(playerProfile); //  "NYI "
```

### std::tie
Создаёт кортеж из ссылок lvalue. Полезно для распаковки объектов `std::pair` и `std::tuple`. Используется `std::ignore` как заполнитель для игнорируемых значений.
```c++
// С кортежами
std::string playerName;
std::tie(std::ignore, playerName, std::ignore) = std::make_tuple(91, "John Tavares", "NYI");

// С парами
std::string yes, no;
std::tie(yes, no) = std::make_pair("yes", "no");
std::tie(yes, no) = std::make_pair("yes", "no");
```

### std::array
`std::array` — контейнер, построенный на основе C-подобного массива. Поддерживает общие операции контейнеров, такие как сортировка.
```c++
std::array<int, 3> a = {2, 1, 3};
std::sort(a.begin(), a.end()); // a == { 1, 2, 3 }
for (int& x : a) x *= 2; // a == { 2, 4, 6 }
```

### Unordered containers
Эти контейнеры обеспечивают среднюю константную сложность поиска, вставки и удаления. Для достижения константного времени жертвуют упорядоченностью контейнера ради скорости, помещая элементы в корзины по хешу. Есть четыре неупорядоченных контейнера:
* `unordered_set`
* `unordered_multiset`
* `unordered_map`
* `unordered_multimap`

### std::make_shared
`std::make_shared` — рекомендуемый способ создания экземпляров `std::shared_ptr` по следующим причинам:
* Позволяет избежать использования оператора `new`.
* Предотвращает повторение кода при указании базового типа указателя.
* Обеспечивает безопасность по исключениям. Предположим, мы вызываем функцию `foo` следующим образом:
```c++
foo(std::shared_ptr<T>{new T{}}, function_that_throws(), std::shared_ptr<T>{new T{}});
```
Компилятор может свободно вызвать `new T{}`, затем `function_that_throws()` и т.д. Поскольку мы уже выделили память в куче при первом создании `T`, здесь возникает утечка.`std::make_shared` обеспечивает безопасность по исключениям:
```c++
foo(std::make_shared<T>(), function_that_throws(), std::make_shared<T>());
```
* Предотвращает необходимость двух выделений памяти. При вызове `std::shared_ptr{ new T{} }` нужно выделить память под `T`, а затем в shared pointer — память под блок управления.
См. раздел [smart pointers](#smart-pointers) для получения дополнительной информации о `std::unique_ptr` и `std::shared_ptr`.

### std::ref
`std::ref(val)` используется для создания объекта типа `std::reference_wrapper`, который содержит ссылку на `val`. Используется в случаях, когда обычная передача по ссылке с помощью `&` не компилируется или `&` теряется из-за вывода типов. `std::cref` аналогичен, но созданный враппер содержит константную ссылку на `val`.
```c++
// создаём контейнер для хранения ссылок на объекты.
auto val = 99;
auto _ref = std::ref(val);
_ref++;
auto _cref = std::cref(val);
//_cref++; не компилируется
std::vector <std::reference_wrapper <int>>vec; // vector <int&>vec не компилируется
vec.push_back(_ref); // vec.push_back( &i) не компилируется
cout  << val  << endl; // печатает 100
cout  << vec[0]  << endl; // печатает 100
cout  << _cref; // печатает 100
```

### Memory model
C++11 представляет модель памяти для C++, что означает поддержку многопоточности и атомарных операций. Некоторые из этих операций включают (но не ограничиваются): атомарные чтение/запись, compare-and-swap, атомарные флаги, promises, futures, блокировки и условные переменные.
См. разделы: [std::thread](#stdthread)

### std::async
`std::async` запускает заданную функцию либо асинхронно, либо с отложенным вычислением, и возвращает `std::future`, содержащий результат вызова функции.

Первый параметр — политика, которая может быть:
1. `std::launch::async | std::launch::deferred` — выбор между асинхронным выполнением и отложенным вычислением остаётся за реализацией.
2. `std::launch::async` — запуск вызываемого объекта в новом потоке.
3. `std::launch::deferred` — отложенное вычисление в текущем потоке.
```c++
int foo() {
  /* working */
  return 1000;
}

auto handle = std::async(std::launch::async, foo);  // создать асинхронную задачу
auto result = handle.get();  // ждать результата
```

### std::begin/end
Были добавлены функции `std::begin` и `std::end` для получения итераторов начала и конца контейнера. Эти функции также работают с обычными массивами, у которых нет методов `begin` и `end`.
```c++
template <typename T>
int CountTwos(const T& container) {
  return std::count_if(std::begin(container), std::end(container), [](int item) {
    return item == 2;
  });
}

std::vector<int> vec = {2, 2, 43, 435, 4543, 534};
int arr[8] = {2, 43, 45, 435, 32, 32, 32, 32};
auto a = CountTwos(vec); // 2
auto b = CountTwos(arr);  // 1
```