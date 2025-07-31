# C++17

## Обзор

C++17 включает следующие новые языковые возможности:
- [template argument deduction for class templates](#template-argument-deduction-for-class-templates)
- [declaring non-type template parameters with auto](#declaring-non-type-template-parameters-with-auto)
- [folding expressions](#folding-expressions)
- [new rules for auto deduction from braced-init-list](#new-rules-for-auto-deduction-from-braced-init-list)
- [constexpr lambda](#constexpr-lambda)
- [lambda capture this by value](#lambda-capture-this-by-value)
- [inline variables](#inline-variables)
- [nested namespaces](#nested-namespaces)
- [structured bindings](#structured-bindings)
- [selection statements with initializer](#selection-statements-with-initializer)
- [constexpr if](#constexpr-if)
- [utf-8 character literals](#utf-8-character-literals)
- [direct-list-initialization of enums](#direct-list-initialization-of-enums)
- [\[\[fallthrough\]\], \[\[nodiscard\]\], \[\[maybe_unused\]\] attributes](#fallthrough-nodiscard-maybe_unused-attributes)
- [\_\_has\_include](#\_\_has\_include)
- [class template argument deduction](#class-template-argument-deduction)

C++17 включает следующие новые библиотечные возможности:
- [std::variant](#stdvariant)
- [std::optional](#stdoptional)
- [std::any](#stdany)
- [std::string_view](#stdstring_view)
- [std::invoke](#stdinvoke)
- [std::apply](#stdapply)
- [std::filesystem](#stdfilesystem)
- [std::byte](#stdbyte)
- [splicing for maps and sets](#splicing-for-maps-and-sets)
- [parallel algorithms](#parallel-algorithms)
- [std::sample](#stdsample)
- [std::clamp](#stdclamp)
- [std::reduce](#stdreduce)
- [prefix sum algorithms](#prefix-sum-algorithms)
- [gcd and lcm](#gcd-and-lcm)
- [std::not_fn](#stdnot_fn)
- [string conversion to/from numbers](#string-conversion-tofrom-numbers)
- [rounding functions for chrono durations and timepoints](#rounding-functions-for-chrono-durations-and-timepoints)

## Языковые возможности C++17

### Template argument deduction for class templates
Автоматический вывод аргументов шаблона, аналогичный тому, как это делается для функций, но теперь включает и конструкторы классов.
```c++
template <typename T = float>
struct MyContainer {
  T val;
  MyContainer() : val{} {}
  MyContainer(T val) : val{val} {}
  // ...
};
MyContainer c1 {1}; // OK MyContainer<int>
MyContainer c2; // OK MyContainer<float>
```

### Declaring non-type template parameters with auto
Следуя правилам вывода `auto`, с учётом допустимых типов для неконстантных параметров шаблона[*], аргументы шаблона могут выводиться из типов переданных аргументов:
```c++
template <auto... seq>
struct my_integer_sequence {
  // Реализация
};

// Явно указываем тип `int` как аргумент шаблона.
auto seq = std::integer_sequence<int, 0, 1, 2>();
// Тип выводится как `int`.
auto seq2 = my_integer_sequence<0, 1, 2>();
```

\* - Например, нельзя использовать `double` в качестве типа параметра шаблона, что также делает такой вывод с помощью `auto` недопустимым.

### Folding expressions
Выражение свёртки выполняет свёртку пакета параметров шаблона с помощью бинарного оператора.
* Выражение вида `(... op e)` или `(e op ...)`, где `op` — оператор свёртки, а `e` — нераскрытый пакет параметров, называется унарной свёрткой.
* Выражение вида `(e1 op ... op e2)`, где `op` — операторы свёртки, называется бинарной свёрткой. Либо `e1`, либо `e2` является нераскрытым пакетом параметров, но не оба одновременно.
```c++
template <typename... Args>
bool logicalAnd(Args... args) {
    // Бинарная свёртка.
    return (true && ... && args);
}
bool b = true;
bool& b2 = b;
logicalAnd(b, b2, true); // == true
```
```c++
template <typename... Args>
auto sum(Args... args) {
    // Унарная свёртка.
    return (... + args);
}
sum(1.0, 2.0f, 3); // == 6.0
```

### New rules for auto deduction from braced-init-list
Изменения в выводе типа `auto` при использовании синтаксиса унифицированной инициализации. Ранее `auto x {3};` выводил `std::initializer_list<int>`, теперь же тип выводится как `int`.
```c++
auto x1 {1, 2, 3}; // ошибка: не один элемент
auto x2 = {1, 2, 3}; // x2 имеет тип std::initializer_list<int>
auto x3 {3}; // x3 имеет тип int
auto x4 {3.0}; // x4 имеет тип double
```

### constexpr lambda
Лямбды, вычисляемые во время компиляции, с использованием `constexpr`.
```c++
auto identity = [](int n) constexpr { return n; };
static_assert(identity(123) == 123);
```
```c++
constexpr auto add = [](int x, int y) {
  auto L = [=] { return x; };
  auto R = [=] { return y; };
  return [=] { return L() + R(); };
};

static_assert(add(1, 2)() == 3);
```
```c++
constexpr int addOne(int n) {
  return [n] { return n + 1; }();
}

static_assert(addOne(1) == 2);
```

### Lambda capture `this` by value
Ранее захват `this` в лямбде был возможен только по ссылке. Примером проблемы является асинхронный код с обратными вызовами, требующий, чтобы объект оставался доступным, возможно, за пределами своего времени жизни. Теперь `*this` (C++17) создаёт копию текущего объекта, в то время как `this` (C++11) продолжает захватывать по ссылке.
```c++
struct MyObj {
  int value {123};
  auto getValueCopy() {
    return [*this] { return value; };
  }
  auto getValueRef() {
    return [this] { return value; };
  }
};
MyObj mo;
auto valueCopy = mo.getValueCopy();
auto valueRef = mo.getValueRef();
mo.value = 321;
valueCopy(); // 123
valueRef(); // 321
```

### Inline variables
Спецификатор inline может применяться как к функциям, так и к переменным. Переменная, объявленная как inline, имеет ту же семантику, что и функция, объявленная как inline.
```c++
// compiler explorer
struct S { int x; };
inline S x1 = S{321}; // mov esi, dword ptr [x1]
                      // x1: .long 321

S x2 = S{123};        // mov eax, dword ptr [.L_ZZ4mainE2x2]
                      // mov dword ptr [rbp - 8], eax
                      // .L_ZZ4mainE2x2: .long 123
```

Также может использоваться для объявления и определения статической переменной-члена, таким образом, она не требует инициализации в исходном файле.
```c++
struct S {
  S() : id{count++} {}
  ~S() { count--; }
  int id;
  static inline int count{0}; // объявляем и инициализируем count внутри класса
};
```

### Nested namespaces
Использование оператора разрешения имён для создания вложенных определений пространств имён.
```c++
namespace A {
  namespace B {
    namespace C {
      int i;
    }
  }
}
```

Приведённый выше код можно записать следующим образом:
```c++
namespace A::B::C {
  int i;
}
```

### Structured bindings
Механизм декомпозиции кортежеподобных объектов, позволяющее писать `auto [ x, y, z ] = expr;`, где тип `expr` — tuple-подобный объект, элементы которого будут привязаны к объявленным переменным `x`, `y` и `z`. Tuple-подобные объекты включают `[std::tuple](README.md#tuples)`, `std::pair`, `[std::array](README.md#stdarray)` и агрегатные структуры.
```c++
using Coordinate = std::pair<int, int>;
Coordinate origin() {
  return Coordinate{0, 0};
}

const auto [ x, y ] = origin();
x; // == 0
y; // == 0
```
```c++
std::unordered_map<std::string, int> mapping {
  {"a", 1},
  {"b", 2},
  {"c", 3}
};

// Декомпозиция
for (const auto& [key, value] : mapping) {
  // key, value
}
```

### Selection statements with initializer
Новые версии операторов `if` и `switch`, упрощающие распространённые паттерны и помогающие поддерживать узкие области видимости.
```c++
{
  std::lock_guard<std::mutex> lk(mx);
  if (v.empty()) v.push_back(val);
}
// против
if (std::lock_guard<std::mutex> lk(mx); v.empty()) {
  v.push_back(val);
}
```
```c++
Foo gadget(args);
switch (auto s = gadget.status()) {
  case OK: gadget.zip(); break;
  case Bad: throw BadFoo(s.message());
}
// против
switch (Foo gadget(args); auto s = gadget.status()) {
  case OK: gadget.zip(); break;
  case Bad: throw BadFoo(s.message());
}
```

### constexpr if
Позволяет писать код, который инстанцируется в зависимости от условия времени компиляции.
```c++
template <typename T>
constexpr bool isIntegral() {
  if constexpr (std::is_integral<T>::value) {
    return true;
  } else {
    return false;
  }
}
static_assert(isIntegral<int>() == true);
static_assert(isIntegral<char>() == true);
static_assert(isIntegral<double>() == false);
struct S {};
static_assert(isIntegral<S>() == false);
```

### UTF-8 character literals
Символьный литерал, начинающийся с `u8`, является литералом типа `char`. Значение UTF-8 символьного литерала равно его кодовой точке ISO 10646.
```c++
char x = u8'x';
```

### Direct list initialization of enums
Перечисления теперь можно инициализировать с помощью фигурных скобок.
```c++
enum byte : unsigned char {};
byte b {0}; // OK
byte c {-1}; // ОШИБКА
byte d = byte{1}; // OK
byte e = byte{256}; // ОШИБКА
```

### \[\[fallthrough\]\], \[\[nodiscard\]\], \[\[maybe_unused\]\] attributes
C++17 вводит три новых атрибута: `[[fallthrough]]`, `[[nodiscard]]` и `[[maybe_unused]]`.
* `[[fallthrough]]` указывает компилятору, что переход (fallthrough) в операторе switch является преднамеренным. Этот атрибут можно использовать только в операторе switch и должен размещаться перед следующей меткой case/default.
```c++
switch (n) {
  case 1: 
    // ...
    [[fallthrough]];
  case 2:
    // ...
    break;
  case 3:
    // ...
    [[fallthrough]];
  default:
    // ...
}
```

* `[[nodiscard]]` вызывает предупреждение, если функция или класс имеют этот атрибут, а возвращаемое значение игнорируется.
```c++
[[nodiscard]] bool do_something() {
  return is_success; // 
}

do_something(); // предупреждение: игнорируется возвращаемое значение 'bool do_something()',
                // объявленной с атрибутом 'nodiscard'
```
```c++
// Выдаёт предупреждение только если `error_info` возвращается по значению.
struct [[nodiscard]] error_info {
  // ...
};

error_info do_something() {
  error_info ei;
  // ...
  return ei;
}

do_something(); // предупреждение: игнорируется возвращаемое значение типа 'error_info',
                // объявленного с атрибутом 'nodiscard'
```

* `[[maybe_unused]]` указывает компилятору, что переменная или параметр могут быть ожидаемо неиспользованными.
```c++
void my_callback(std::string msg, [[maybe_unused]] bool error) {
  // Неважно, является ли `msg` сообщением об ошибке, просто логируем его.
  log(msg);
}
```

### \_\_has\_include
Оператор `__has_include (operand)` может использоваться в выражениях `#if` и `#elif` для проверки доступности заголовочного или исходного файла (`operand`) для включения.
Один из случаев использования — использование двух библиотек с одинаковой функциональностью, при этом подключается резервная/экспериментальная библиотека, если основная отсутствует в системе.
```c++
#ifdef __has_include
#  if __has_include(<optional>)
#    include <optional>
#    define have_optional 1
#  elif __has_include(<experimental/optional>)
#    include <experimental/optional>
#    define have_optional 1
#    define experimental_optional
#  else
#    define have_optional 0
#  endif
#endif
```

Также может использоваться для включения заголовков, существующих под разными именами или в разных местах на различных платформах, без знания, на какой платформе выполняется программа. Хороший пример — заголовки OpenGL, которые находятся в каталоге `OpenGL\` на macOS и в `GL\` на других платформах.
```c++
#ifdef __has_include
#  if __has_include(<OpenGL/gl.h>)
#    include <OpenGL/gl.h>
#    include <OpenGL/glu.h>
#  elif __has_include(<GL/gl.h>)
#    include <GL/gl.h>
#    include <GL/glu.h>
#  else
#    error No suitable OpenGL headers found.
# endif
#endif
```

### Class template argument deduction
CTAD позволяет компилятору выводить аргументы шаблона из аргументов конструктора.
```c++
std::vector v{ 1, 2, 3 }; // выводит std::vector<int>

std::mutex mtx;
auto lck = std::lock_guard{ mtx }; // выводит std::lock_guard<std::mutex>

auto p = new std::pair{ 1.0, 2.0 }; // выводит std::pair<double, double>*
```

Для пользовательских типов можно использовать *deduction guides* (руководство по выводу), чтобы указать компилятору, как выводить аргументы шаблона:
```c++
template  <typename T>
struct container {
  container(T t) {}

  template  <typename Iter>
  container(Iter beg, Iter end);
};

// руководство по выводу
template  <typename Iter>
container(Iter b, Iter e) -> container <typename std::iterator_traits <Iter>::value_type>;

container a{ 7 }; // OK: выводит container <int >

std::vector <double> v{ 1.0, 2.0, 3.0 };
auto b = container{ v.begin(), v.end() }; // OK: выводит container <double >

container c{ 5, 6 }; // ОШИБКА: std::iterator_traits <int >::value_type не является типом
```

## C++17 Library Features

### std::variant
Шаблон класса `std::variant` представляет типобезопасный `union`. Экземпляр `std::variant` в любой момент времени содержит значение одного из своих типов (также возможно, что он не содержит значения).
```c++
std::variant<int, double> v{ 12 };
std::get<int>(v); // == 12
std::get<0>(v); // == 12
v = 12.0;
std::get<double>(v); // == 12.0
std::get<1>(v); // == 12.0
```

### std::optional
Шаблон класса `std::optional` управляет необязательным значением, т.е. значением, которое может присутствовать или отсутствовать. Распространённый случай использования — возвращаемое значение функции, которая может завершиться неудачей.
```c++
std::optional<std::string> create(bool b) {
  if (b) {
    return "Godzilla";
  } else {
    return {};
  }
}

create(false).value_or("empty"); // == "empty"
create(true).value(); // == "Godzilla"
// фабричные функции, возвращающие optional, могут использоваться как условия в while и if
if (auto str = create(true)) {
  // ...
}
```

### std::any
Типобезопасный контейнер для одиночного значения любого типа.
```c++
std::any x {5};
x.has_value() // == true
std::any_cast<int>(x) // == 5
std::any_cast<int&>(x) = 10;
std::any_cast<int>(x) // == 10
```

### std::string_view
Неуправляющая ссылка на строку. Полезно для создания абстракции над строками (например, для парсинга).
```c++
// Обычные строки.
std::string_view cppstr {"foo"};
// Строки с расширенными символами.
std::wstring_view wcstr_v {L"baz"};
// Массивы символов.
char array[3] = {'b', 'a', 'r'};
std::string_view array_v(array, std::size(array));
```
```c++
std::string str {"   trim me"};
std::string_view v {str};
v.remove_prefix(std::min(v.find_first_not_of(" "), v.size()));
str; //  == "   trim me"
v; // == "trim me"
```

### std::invoke
Вызов объекта `Callable` с параметрами. Примеры вызываемых объектов: `std::function` или лямбды; объекты, которые можно вызывать аналогично обычной функции.
```c++
template <typename Callable>
class Proxy {
  Callable c_;

public:
  Proxy(Callable c) : c_{ std::move(c) } {}

  template <typename... Args>
  decltype(auto) operator()(Args&&... args) {
    // ...
    return std::invoke(c_, std::forward<Args>(args)...);
  }
};

const auto add = [](int x, int y) { return x + y; };
Proxy p{ add };
p(1, 2); // == 3
```

### std::apply
Вызов объекта `Callable` с кортежем аргументов.
```c++
auto add = [](int x, int y) {
  return x + y;
};
std::apply(add, std::make_tuple(1, 2)); // == 3
```

### std::filesystem
Новая библиотека `std::filesystem` предоставляет стандартный способ манипуляции файлами, каталогами и путями в файловой системе.

Здесь большой файл копируется во временную директорию, если есть свободное место:
```c++
const auto bigFilePath {"bigFileToCopy"};
if (std::filesystem::exists(bigFilePath)) {
  const auto bigFileSize {std::filesystem::file_size(bigFilePath)};
  std::filesystem::path tmpPath {"/tmp"};
  if (std::filesystem::space(tmpPath).available > bigFileSize) {
    std::filesystem::create_directory(tmpPath.append("example"));
    std::filesystem::copy_file(bigFilePath, tmpPath.append("newFile"));
  }
}
```

### std::byte
Новый тип `std::byte` предоставляет стандартный способ представления данных как байта. Преимущества `std::byte` по сравнению с `char` или `unsigned char` в том, что он не является символьным типом и не является арифметическим типом; единственные доступные перегрузки операторов — побитовые операции.
```c++
std::byte a {0};
std::byte b {0xFF};
int i = std::to_integer<int>(b); // 0xFF
std::byte c = a & b;
int j = std::to_integer<int>(c); // 0
```
`std::byte` — это является перечислением, а фигурная инициализация перечислений стала возможной благодаря [direct-list-initialization of enums](#direct-list-initialization-of-enums).

### Splicing for maps and sets
Перемещение элементов и объединение контейнеров без накладных расходов на копирование, перемещение или выделение/освобождение памяти.

Перемещение элементов из одной map в другую:
```c++
std::map <int, string > src {{1,  "one "}, {2,  "two "}, {3,  "buckle my shoe "}};
std::map <int, string > dst {{3,  "three "}};
dst.insert(src.extract(src.find(1))); // Дешёвое удаление и вставка { 1,  "one " } из `src` в `dst`.
dst.insert(src.extract(2)); // Дешёвое удаление и вставка { 2,  "two " } из `src` в `dst`.
// dst == { { 1,  "one " }, { 2,  "two " }, { 3,  "three " } };
```

Вставка целого множества:
```c++
std::set<int> src {1, 3, 5};
std::set<int> dst {2, 4, 5};
dst.merge(src);
// src == { 5 }
// dst == { 1, 2, 3, 4, 5 }
```

Вставка элементов, которые живут дольше контейнера:
```c++
auto elementFactory() {
  std::set<...> s;
  s.emplace(...);
  return s.extract(s.begin());
}
s2.insert(elementFactory());
```

Изменение ключа элемента map:
```c++
std::map<int, string> m {{1, "one"}, {2, "two"}, {3, "three"}};
auto e = m.extract(2);
e.key() = 4;
m.insert(std::move(e));
// m == { { 1, "one" }, { 3, "three" }, { 4, "two" } }
```

### Parallel algorithms
Многие алгоритмы STL, такие как `copy`, `find` и `sort`, начали поддерживать политики параллельного выполнения: `seq`, `par` и `par_unseq`, которые означают «последовательно», «параллельно» и «параллельно без упорядочения».
```c++
std::vector<int> longVector;
// Поиск элемента с использованием параллельной политики выполнения
auto result1 = std::find(std::execution::par, std::begin(longVector), std::end(longVector), 2);
// Сортировка элементов с использованием последовательной политики выполнения
auto result2 = std::sort(std::execution::seq, std::begin(longVector), std::end(longVector));
```

### std::sample
Выбирает n элементов из заданной последовательности (без повторений), при этом каждый элемент имеет равные шансы быть выбранным.
```c++
const std::string ALLOWED_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
std::string guid;
// Выбираем 5 символов из ALLOWED_CHARS.
std::sample(ALLOWED_CHARS.begin(), ALLOWED_CHARS.end(), std::back_inserter(guid),
  5, std::mt19937{ std::random_device{}() });

std::cout << guid; // например, G1fW2
```

### std::clamp
Ограничивает заданное значение между нижней и верхней границами.
```c++
std::clamp(42, -1, 1); // == 1
std::clamp(-42, -1, 1); // == -1
std::clamp(0, -1, 1); // == 0

// `std::clamp` также принимает пользовательский компаратор:
std::clamp(0, -1, 1, std::less<>{}); // == 0
```

### std::reduce
Свёртка заданного диапазона элементов. Концептуально похоже на `std::accumulate`, но `std::reduce` выполняет свёртку параллельно. Из-за параллельного выполнения, если вы указываете бинарную операцию, она должна быть ассоциативной и коммутативной. Также операция не должна изменять элементы или делать невалидными итераторы в заданном диапазоне.

Операция по умолчанию — std::plus с начальным значением 0.
```c++
const std::array<int, 3> a{ 1, 2, 3 };
std::reduce(std::cbegin(a), std::cend(a)); // == 6
// Использование пользовательской бинарной операции:
std::reduce(std::cbegin(a), std::cend(a), 1, std::multiplies<>{}); // == 6
```

Кроме того, можно указывать преобразования:
```c++
const std::array<int, 3> b{ 1, 2, 3 };
const auto product_times_ten = [](const auto a, const auto b) { return a * b * 10; };

std::transform_reduce(std::cbegin(a), std::cend(a), 0, std::plus<>{}, times_ten); // == 60

std::transform_reduce(std::cbegin(a), std::cend(a), std::cbegin(b), 0, std::plus<>{}, product_times_ten); // == 140
```

### Prefix sum algorithms
Префиксные суммы (также называемые сканированием) позволяют вычислять кумулятивные (накопленные) значения в последовательности. Они бывают двух основных типов: 
* инклюзивное сканирование (текущий элемент включается)
* эксклюзивное сканирование (текущий элемент не включается)
```c++
const std::array <int, 3 > a{ 1, 2, 3 };

// Вычисляет сумму с включением текущего элемента
std::inclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator <int >{ std::cout,  "  " }, std::plus < >{}); // 1 3 6


// Вычисляет сумму без включения текущего элемента
std::exclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator <int >{ std::cout,  "  " }, 0, std::plus < >{}); // 0 1 3


const auto times_ten = [](const auto n) { return n * 10; };

// Применяет функцию перед инклюзивным суммированием
std::transform_inclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator <int >{ std::cout,  "  " }, std::plus < >{}, times_ten); // 10 30 60

// Применяет функцию перед эксклюзивным суммированием
std::transform_exclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator <int >{ std::cout,  "  " }, 0, std::plus < >{}, times_ten); // 0 10 30
```

### GCD and LCM
Наибольший общий делитель (GCD) и наименьшее общее кратное (LCM).
```c++
const int p = 9;
const int q = 3;
std::gcd(p, q); // == 3
std::lcm(p, q); // == 9
```

### std::not_fn
Функция, возвращающая отрицание результата функции.
```c++
const std::ostream_iterator<int> ostream_it{ std::cout, " " };
const auto is_even = [](const auto n) { return n % 2 == 0; };
std::vector<int> v{ 0, 1, 2, 3, 4 };

// Печатаем все чётные числа.
std::copy_if(std::cbegin(v), std::cend(v), ostream_it, is_even); // 0 2 4
// Печатаем все нечётные (не чётные) числа.
std::copy_if(std::cbegin(v), std::cend(v), ostream_it, std::not_fn(is_even)); // 1 3
```

### String conversion to/from numbers
Преобразование целых и вещественных чисел в строку и обратно. Преобразования не выбрасывают исключений, не выполняют выделение памяти и безопаснее эквивалентов из стандартной библиотеки C.
Пользователь отвечает за выделение достаточного объёма памяти для `std::to_chars`, иначе функция завершится неудачей, установив объект кода ошибки в своём возвращаемом значении.

Эти функции позволяют опционально передавать основание (по умолчанию — 10) или спецификатор формата для вещественных чисел.

* `std::to_chars` возвращает (неконстантный) указатель char, указывающий на символ сразу после последнего записанного символа в буфере, и объект кода ошибки.
* `std::from_chars` возвращает константный указатель char, который при успехе равен конечному указателю, переданному в функцию, и объект кода ошибки.

Оба объекта кода ошибки, возвращённые этими функциями, равны объекту кода ошибки по умолчанию при успехе.

Преобразование числа `123` в `std::string`:
```c++
const int n = 123;

// Можно использовать любой контейнер: строку, массив и т.д.
std::string str;
str.resize(3); // выделить достаточно памяти для каждой цифры `n`

const auto [ ptr, ec ] = std::to_chars(str.data(), str.data() + str.size(), n);

if (ec == std::errc{}) { std::cout << str << std::endl; } // 123
else { /* обработка ошибки */ }
```

Преобразование из `std::string` со значением `"123"` в целое число:
```c++
const std::string str{ "123" };
int n;

const auto [ ptr, ec ] = std::from_chars(str.data(), str.data() + str.size(), n);

if (ec == std::errc{}) { std::cout << n << std::endl; } // 123
else { /* обработка ошибки */ }
```

### Rounding functions for chrono durations and timepoints
Предоставляются вспомогательные функции abs, round, ceil и floor для `std::chrono::duration` и `std::chrono::time_point`.
```c++
using seconds = std::chrono::seconds;
std::chrono::milliseconds d{ 5500 };
std::chrono::abs(d); // == 5s
std::chrono::round<seconds>(d); // == 6s
std::chrono::ceil<seconds>(d); // == 6s
std::chrono::floor<seconds>(d); // == 5s
```