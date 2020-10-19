[原文地址](https://github.com/apple/swift/blob/master/docs/HowSwiftImportsCAPIs.md)

当 Swift 导入一个模块，或者从 C 系语言（C，Objective-C）解析一个 bridging header(桥接头文件)，这些 API 会被映射到 Swift API 中，并且可以直接在 Swift 代码中使用。这为 Swift 的外部功能接口（FFI）提供了基础，而 FFI 提供了与 C 系语言编写的库的交互性。

这篇文档描述了 C 系语言的 API 如何被映射为 Swift API。本文目标受众广泛，不只包括 Swift 和 C 的语言专家。因此，在必要的时候解释了一些高级的概念（advanced concepts）。

# Names, identifiers and keywords

## Unicode

C 和 C++ 允许在标识符中使用非 ASCII 码的 Unicode 代码点（code point）。尽管 Swift 不允许在标识符中使用任何 Unicode 代码点（因此兼容性可能并不完美），但它尝试允许使用合理的 Unicode 代码点。 实际上 C、Objective-C 和 C++ 代码都不会在标识符中使用非 ASCII 代码点，因此，将它们映射到 Swift 并没有什么问题。

## Names that are keywords in Swift
有些 C 和 C++ 的标识符在 Swift 中作为了关键字。尽管如此，由于 Swift 允许转义的关键字作为标识符，因此此类名称仍按原样导入 Swift 。
```C
// C header.

// The name of this function is a keyword in Swift.
void func();
```
```Swift
// C header imported in Swift.

// The name of the function is still `func`, but it is escaped to make the
// keyword into an identifier.
func `func`()
```

```Swift
// Swift user.

func test() {
    // Call the C function declared above.
    `func`()
}
```
## Name translation
一些 C 声明的名称在 Swift 中显示有所不同。 特别是，枚举（枚举常量）的名称经过转译过程会被硬编码到编译器中。查看 [Name Translation from C to Swift](https://github.com/apple/swift/blob/main/docs/CToSwiftNameTranslation.md) 获取更多细节。

## Name customization
作为 Swift/C 相互操作的一般原则，C API 提供者可以较好的控制其 API 在 Swift 中的显示方式。 特别是，API 提供者可以使用 Clang 的swift_name 属性自定义其 C API 的名称，以使其更符合在 Swift 中的使用习惯。查看 [Name Translation from C to Swift](https://github.com/apple/swift/blob/main/docs/CToSwiftNameTranslation.md) 获取更多细节。

# Size, stride, and alignment of types
在 C 语言中，每种类型都有一个 size（使用 `sizeof` 操作符计算）和一个 alignment（使用 `alignof` 计算）。

在 Swift 中，类型具有 size，stride 和 alignment。

在 C 和 Swift 中 alignment 的概念是完全相同的。

Size 和 stride 会更复杂一些。在 Swift 中，stride 是数组中两个元素之间的距离。Swift 中类型的 size 是 stride 减去尾部填充（tail padding）。例如:

```Swift
struct SwiftStructWithPadding {
  var x: Int16
  var y: Int8
}
print(MemoryLayout<SwiftStructWithPadding>.size) // 3
print(MemoryLayout<SwiftStructWithPadding>.stride) // 4
```

C 中 size 的概念对应 Swift 中的 stride（不是 size ！）。C 中没有与 Swift 的 size 对等的概念。

Swift 能够获得类型存储数据的精确大小从而可以把另外的数据打包塞入尾部填充里，不然这部分空间也会被浪费。Swift 还会跟踪每种类型可能的和不可能的位模式（bit patterns），并重复使用不可能的位模式来编码更多信息，类似于 `llvm::PointerIntPair` 和 `llvm::PointerUnion`。Swift 会自动完成这一任务，而且对用户是透明。例如：

```Swift
// This enum takes 1 byte in memory, which has 256 possible bit patterns.
// However, only 2 bit patterns are used.
enum Foo {
  case A
  case B
}
print(MemoryLayout<Foo>.size) // 1
print(MemoryLayout<Foo?>.size) // also 1: `nil` is represented as one of the 254 bit patterns that are not used by `Foo.A` or `Foo.B`.
```

然而，对于从 C 导入的类型，它的 size 和 stride 是相等的。

```C
// C header.
struct CStructWithPadding {
  int16_t x;
  int8_t y;
};
```

```Swift
// C header imported in Swift.
struct CStructWithPadding {
  var x: Int16
  var y: Int8
}
```

```
print(MemoryLayout<CStructWithPadding>.size) // 4
print(MemoryLayout<CStructWithPadding>.stride) // 4
```

# Fundamental types
在 C 语言中，某些类型（char，int，float等）内置于语言和编译器中。这些内置类型具有的某些行为是用户定义的类型无法模仿的（例如，常用的算术转换）。

C 和 C++ 标准库提供了定义字符和整数类型的头文件，这些类型与底层内置类型一一对应，例如：`int16_t`、`size_t` 和 `ptrdiff_t`。

Swift 没有这类内置类型。Swift 和 C 基本类型的等效项在 Swift 标准库中定义为普通的结构体。它们是使用编译器内部函数实现的，但是 API 是通过普通的 Swift 代码定义的：

```Swift
// From the Swift standard library:
struct Int32 {
  internal var _value: Builtin.Int32

  // Note: `Builtin.Xyz` types are only accessible to the standard library.
}

func +(lhs: Int32, rhs: Int32) -> Int32 {
  return Int32(_value: Builtin.add_Int32(lhs._value, rhs._value))
}
```

这些“基本”的 Swift 类型的内存布局是意料之中的，没有隐藏的 vtable 指针、元数据或引用计数；正如您希望从相应的 C 类型中得到的结果那样，仅是内联存储的数据。 例如，Swift 的 `Int32` 是四个字节都存储数字的连续块。

除了少数例外，C 中的基本类型都有实现定义的大小、对齐和 stride (两个数组元素之间的距离)。

Swift 的整数和浮点类型具有固定的大小、对齐和 stride，但有两个例外：`Int` 和 `UInt`。`Int` 和 `UInt` 的大小与平台上指针的大小匹配，这与在大多数 C 实现中 `size_t`、`ptrdiff_t`、`intptr_t` 和 `uintptr_t` 具有与指针相同的大小类似。`Int` 和 `UInt` 是不同的类型，它们不是明确大小类型的类型别名（typealias）。即使在当前平台上碰巧大小匹配，也需要在 `Int`、`UInt` 和任何其他整数类型之间进行显式转换。

下表总结了 C 和 C++ 的基本类型与 Swift 类型之间的映射关系。 该表基于[swift.git/include/swift/ClangImporter/BuiltinMappedTypes.def](https://github.com/apple/swift/blob/main/include/swift/ClangImporter/BuiltinMappedTypes.def)。

| C and C++ types                      | Swift types |
| ------------------------------------ | ----------- |
| C `_Bool`, C++ `bool`                | `typealias CBool = Bool` |
| `char`, regardless if the target defines it as signed or unsigned | `typealias CChar = Int8` |
| `signed char`, explicitly signed     | `typealias CSignedChar = Int8` |
| `unsigned char`, explicitly unsigned | `typealias CUnsignedChar = UInt8` |
| `short`, `signed short`              | `typealias CShort = Int16` |
| `unsigned short`                     | `typealias CUnsignedShort = UInt16` |
| `int`, `signed int`                  | `typealias CInt = Int32` |
| `unsigned int`                       | `typealias CUnsignedInt = UInt32` |
| `long`, `signed long`                | Windows x86\_64: `typealias CLong = Int32`<br>Everywhere else: `typealias CLong = Int` |
| `unsigned long`                      | Windows x86\_64: `typealias CUnsignedLong = UInt32`<br>Everywhere else: `typealias CUnsignedLong = UInt` |
| `long long`, `signed long long`      | `typealias CLongLong = Int64` |
| `unsigned long long`                 | `typealias CUnsignedLongLong = UInt64` |
| `wchar_t`, regardless if the target defines it as signed or unsigned | `typealias CWideChar = Unicode.Scalar`<br>`Unicode.Scalar` is a wrapper around `UInt32` |
| `char8_t` (proposed for C++20) | Not mapped |
| `char16_t`                           | `typealias CChar16 = UInt16` |
| `char32_t`                           | `typealias CChar32 = Unicode.Scalar` |
| `float`                              | `typealias CFloat = Float` |
| `double`                             | `typealias CDouble = Double` |
| `long double`                        | `CLongDouble`, which is a typealias to `Float80` or `Double`, depending on the platform.<br>There is no support for 128-bit floating point. |

首先，请注意，C 类型映射到 Swift 类型别名，而不是直接映射到 Swift 基础类型。类型别名的名称基于 C 的原始类型。这些类型别名使开发人员可以轻松编写 Swift 代码和 API ，这些代码和 API 可以与从 C 导入的 API 和数据一起使用，而无需使用很多 `#if` 条件语句。

该表通常并没有什么问题：C 类型被映射到相应的大小明确的 Swift 类型，除了C 的 `long` 类型之外，C 的 `long` 类型在 32 位和 64 位的 LP64 数据模型平台上被映射为 `Int`。这样做是为了增强 32 位和 64 位代码的可移植性。

C 的 `long` 类型难处理的地方在于根据平台的不同它可以是 32 位的或者 64 位的。如果将 C 的 `long` 类型映射为确定大小的 Swift 类型，那么它在不同的平台将映射为不同的 Swift 类型，这就使编写可移植性的 Swift 代码变得更加困难（用户必须为两个平台编译 Swift 代码才能查看所有错误； 修复其中一个平台上的编译错误可能会导致另一个平台编译报错）。通过将 long 映射为不同的 Int 类型，该语言迫使用户在为任何一个平台编译时都考虑这两种情况。

然而，将 C `long` 映射到 `Int` 并不是普遍有效的。具体来说，它不适用于 LLP64 数据模型平台 （例如Windows x86_64），在这些平台上 C 的 `long` 类型为 32 位而 Swift 的 `Int` 为 64 位。这样做是为了增强 32 位和 64 位代码之间的可移植性。

在 C 标准库中整数类型的 typedef 是这样映射的（来源 [swift.git/swift/lib/ClangImporter/MappedTypes.def](https://github.com/apple/swift/blob/main/lib/ClangImporter/MappedTypes.def)):

| C and C++ types  | Swift types |
| ---------------- | ----------- |
| `uint8_t`        | `UInt8`     |
| `uint16_t`       | `UInt16`    |
| `uint32_t`       | `UInt32`    |
| `uint64_t`       | `UInt64`    |
| `int8_t`         | `Int8`      |
| `int16_t`        | `Int16`     |
| `int32_t`        | `Int32`     |
| `int64_t`        | `Int64`     |
| `intptr_t`       | `Int`       |
| `uintptr_t`      | `UInt`      |
| `ptrdiff_t`      | `Int`       |
| `size_t`         | `Int`       |
| `rsize_t`        | `Int`       |
| `ssize_t`        | `Int`       |

```C
// C header.
double Add(int x, long y);
```
```Swift
// C header imported in Swift.
func Add(_ x: CInt, _ y: CLong) -> CDouble
```

# Free functions
C 函数作为自由函数导入到 Swift 中。 C 函数签名中的每种类型都映射为相应的 Swift 类型。

## Argument labels
默认情况下，导入的 C 函数在 Swift 中没有参数标签。API 所有者可以通过在 C 头文件中添加注解来添加参数标签。

```C
// C header.

#define SWIFT_NAME(X) __attribute__((swift_name(#X)))

// No argument labels by default.
void drawString(const char *, int xPos, int yPos);

// The attribute specifies the argument labels.
void drawStringRenamed(const char *, int xPos, int yPos)
    SWIFT_NAME(drawStringRenamed(_:x:y:));
```

```Swift
// C header imported in Swift.

func drawString(_: UnsafePointer<CChar>!, _ xPos: CInt, _ yPos: CInt)
func drawStringRenamed(_: UnsafePointer<CChar>!, x: CInt, y: CInt)

drawString("hello", 10, 20)
drawStringRenamed("hello", x: 10, y: 20)
```

## Variadic arguments
带有可变参数的 C 函数不会被导入到 Swift，但是技术上是支持导入的。

请注意，带有 `va_list` 参数的函数将被导入到 Swift 中。`va_list` 对应于 Swift 中的 `CVaListPointer`。

C 语言 API 并没有定义很多可变参数函数，因此到目前为止，此限制尚未引起大问题。

通常，对于每个可变参数函数，都有一个对应的带有一个 `va_list` 的函数，可以从 Swift 调用这个函数。一个有积极性的开发人员可以编写一个叠加层（overlay），以暴露一个看起来像 C 可变参数函数的 Swift 可变参数函数，并根据基于 `va_list` 的 C 的 API 对其进行实现。您可以通过在 [swift.git/stdlib](https://github.com/apple/swift/tree/main/stdlib) 中搜索 `withVaList` 函数的用法来找到此类覆盖和包装的示例。

另请参阅有关此主题的 Apple 文档：[使用 CVaListPointer 调用可变参数函数](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/using_imported_c_functions_in_swift#2992073)

## Inline functions
在头文件中定义的 C 的内联函数被导入为常规的 Swift 函数。但是，与自由函数不同，因为不能保证其他转换单元会提供定义，内联函数要求调用者在模块内给出函数的声明。

因此，Swift 编译器使用 Clang 的 CodeGen 库为 C 内联函数发出 LLVM IR。C 内联函数的 LLVM IR 和 Swift 代码的 LLVM IR 被放在一个 LLVM 模块中，从而使所有 LLVM 优化（如内联）都可以跨语言透明地工作。

# Global variables
C 中没有 const 修饰到全局变量当作变量导入到 Swift，const 修饰的全局变量当作常量导入到 Swift。

```C
// C header.
extern int NumAlpacas;
extern const int NumLlamas;
```

```Swift
// C header imported in Swift.
var NumAlpacas: CInt
let NumLlamas: CInt
```

# Pointers to data
C 指向 `T` 类型的指针为 `T*`。

Swift 语言没有提供内置的指针类型。标准库定义了多种指针类型：

* `UnsafePointer<T>`：等效于 C 中的 `const T *`。
* `UnsafeMutablePointer<T>`：等效于 C 中的 `T*`，其中 `T` 不是常量。
* `UnsafeRawPointer`：用于访问非静态类型数据的指针，类似于 C 中的 `const void*`。与 C 指针类型不同，`UnsafeRawPointer` 允许类型双关（type punning），并提供特殊的 API 来正确、安全地进行操作。
* `UnsafeMutableRawPointer`：与 `UnsafeRawPointer` 类似，但可以改变它指向的数据。
* `OpaquePointer`：指向类型化数据的指针，但是指针的类型未知，例如，指针的类型由其他变量的值确定，或者指针的类型在 Swift 中无法表示。
* `AutoreleasingUnsafeMutablePointer<T>`：仅用于与 Objective-C 交互操作；对应于 Objective-C 指针 `T  __autoreleasing *`，其中 T 是 Objective-C 指针类型。

只要在 C 和 Swift 中指针的内存布局相同，就可以在 Swift 中简单地导入 C 指针类型。到目前为止，该文档仅描述了基本类型，它们在 C 和  Swift 中的内存布局是相同的，例如 C 中的 `char` 和 Swift 中的 Int8。指向这些类型的指针很容易导入到 Swift 中，例如，C 中的 `char *` 对应于 Swift 中 `UnsafeMutablePointer <Int8>!`。

```C
// C header.
void AddSecondToFirst(int *x, const long *y);
```

```Swift
// C header imported in Swift.
func AddSecondToFirst(_ x: UnsafeMutablePointer<CInt>!, _ y: UnsafePointer<CLong>!)
```

# Nullable and non-nullable pointers
任何 C 指针都可以为空。但是，实际上，许多指针永远不会为空。因此，代码通常不预期某些指针为空，并且没有优雅地处理空值。

C 没有提供区分可空指针和不可空指针的方法。但是，Swift进行了区分：所有指针类型（例如， `UnsafePointer <T>` 和 `UnsafeMutablePointer <T>` ）都是不可空的。Swift 使用“可选类型”来表示缺失值的可能性，可选类型可以简写为 `T?`，完整写法为 `Optional<T>`。缺省值用“nil”表示。例如，`UnsafePointer<T>?` （`Optional<UnsafePointer<T>>`的简写）可以存储 nil 值。

Swift 为可选类型的声明提供了另外一种语法，`T!`，这种语法创建了一个所谓的“隐式展开可选”的可选类型。如果需要对表达式进行编译，这种可选类型会自动进行 nil 检查并解包。解包 T? 或者包含 nil 值的  `T!` 是严重的错误（程序会终止并显示错误消息）。

形式上，由于任何 C 指针都可以为 null，因此必须在 Swift 中将 C 指针作为可选的不安全指针导入。但是，这在 Swift 中并不是常用的方式：当值确实缺失，并且对 API 有意义时，应该使用可选类型。C 的 API 不以机器可读的形式提供这些信息。关于哪些指针可以为空的信息通常在 C API 的自由格式文档中提供（如果有的话）。

Clang 实现了对 [C 语言的扩展，允许 C API  vendor 将指针注释为可空或非空](https://clang.llvm.org/docs/AttributeReference.html#nullability-attributes)。

引用Clang手册：

>`_Nonnull` 修饰符说明对于 `_Nonnull` 的指针类型 null 是没有意义的。例如，给出如下声明

```C
int fetch(int * _Nonnull ptr);
```

>`fetch` 的调用者不应提供 null 值，如果提供 null 值，编译器将产生警告。请注意，与声明属性 `nonnull` 不同，`_Nonnull` 的存在并不意味着传递 null 是未定义的行为：`fetch` 可以自由考虑 null 的未定义行为或（可能出于向后兼容的原因）防御性地处理 null 值。

根据是否为常量指针，`_Nonnull` 修饰的 C 指针作为非可选类型的 `UnsafePointer<T>` 或 `UnsafeMutablePointer<T>` 导入到 Swift 中。

```Swift
// C declaration above imported in Swift.
func fetch(_ ptr: UnsafeMutablePointer<CInt>) -> CInt
```

引用 Clang 手册：

>`_Nullable` 修饰符表示 `_Nullable` 指针类型的值可以为空。 例如，给定：

```C
int fetch_or_zero(int * _Nullable ptr);
```

>`fetch_or_zero` 的调用者可以提供 null 。

根据是否为常量指针，`_Nullable` 修饰的指针作为 `UnsafePointer<T>?` 或 `UnsafeMutablePointer<T>?` 类型导入到 Swift 中。

```Swift
// C declaration above imported in Swift.
func fetch_or_zero(_ ptr: UnsafeMutablePointer<CInt>?) -> CInt
```

引用 Clang 手册：

>`_Null_unspecified` 修饰符用于修饰 `_Nonnull` 和 `_Nullable` 修饰符均不适用的指针类型。它主要用于表示 nullability-annotated 头文件中带有特定指针的 null 的角色是不明确的，例如，由于过于复杂的实现或由于历史原因遗留的 API。

`_Null_unspecified` 和没有注解的 C 指针作为隐式解包的可选指针类型 `UnsafePointer<T>!` 或 `UnsafeMutablePointer <T>!` 导入到 Swift 中。该策略提供了与 C API 中的相同的功效（无需显式展开），以及 Swift 代码期望的安全性（在隐式展开过程中对 null 进行动态检查）。

这些修饰符不会影响 C 和 C++ 中的程序语义，从而使 C API 提供者可以安全地将它们添加到头文件中来满足 Swift 的用户，而且不会影响现有的 C 和 C++ 用户。

在 C API 中，大多数指针都是非空的。为了减轻注解的负担，Clang 提供了一种将指针大规模注解为不可空的方法，然后使用 `_Nullable` 标记可空的指针。

```C
// C header.

void Func1(int * _Nonnull x, int * _Nonnull y, int * _Nullable z);

#pragma clang assume_nonnull begin
void Func2(int *x, int *y, int * _Nullable z);

#pragma clang assume_nonnull end
```

```Swift
// C header imported in Swift.

// Note that `Func1` and `Func2` are imported identically, but `Func2` required
// fewer annotations in the C header.
func Func1(
  _ x: UnsafeMutablePointer<CInt>,
  _ y: UnsafeMutablePointer<CInt>,
  _ z: UnsafeMutablePointer<CInt>?
)
func Func2(
  _ x: UnsafeMutablePointer<CInt>,
  _ y: UnsafeMutablePointer<CInt>,
  _ z: UnsafeMutablePointer<CInt>?
)
```

采用可空性修饰符的 API 提供者通常将所有声明包装在标头中，并带有一个 `assume_nonnull begin/end` 语法对，然后注释可空指针。

另请参阅有关主题的Apple文档：[在 Objective-C API 中指定可空性](https://developer.apple.com/documentation/swift/objective-c_and_c_code_customization/designating_nullability_in_objective-c_apis)

# Incomplete types and pointers to them
C 和 C++ 有不完整类型的概念；Swift 没有类似的概念。不完整的 C 类型不会以任何形式导入 Swift 中。

有时类型不完整只是偶然发生的，例如，当你刚好向前声明一个类型而不是包含具有完整定义的头文件时，不完整类型就产生了。在这种情况下，为了使 Swift 能够导入 C API，建议在 C 的头文件中使用 `#include` 导入定义类型的头文件来替换向前声明。

不完整类型经常被用来定义不透明类型。由于这个用法太常见了，Swift 不能忽略这个用例。Swift 便将指向不完整类型的指针作为 `OpaquePointer` 导入。例如：

```C
// C header.

struct Foo;
void Print(const Foo* foo);
```

```Swift
// C header imported in Swift.

// Swift can't import the incomplete type `Foo` and has to drop some type
// information when importing a pointer to `Foo`.
func Print(_ foo: OpaquePointer)
```

# Function pointers
C 仅支持一种形式的函数指针：`Result (*)(Arg1, Arg2, Arg3)`。

Swift 与函数指针最相似的等效项是闭包：`(Arg1, Arg2, Arg3) -> Result`。Swift 闭包与 C 函数指针的内存布局不同：Swift 中的闭包包含两个指针，一个指向代码的指针和一个指向捕获的数据（上下文）的指针。C 函数指针可以转换为 Swift 闭包； 但是需要桥接来调整内存布局。

如上所述，在某些情况下用桥接来调整内存布局是不可能，例如，当导入函数指针的指针时。例如，虽然 C 的 `int (*)(char)` 可以导入为 `(CChar) -> CInt`（需要调整内存布局），但 C 的 `int (**)(char)` 不能导入为 `UnsafePointer<(CChar) -> CInt> `，因为在 C 和 Swift 中，指针对象必须具有相同的内存布局。

因此，我们需要一个与 C 函数指针有相同内存布局的 Swift 类型，以备不时之需。这个类型是 `@convention(c) (Arg1, Arg2, Arg3) -> Result` 。

尽管在某些情况下可以将 C 函数指针作为带有上下文指针的 Swift 闭包导入，C 函数指针也总是作为 `@convention(c)` 的“ 闭包”导入（用引号引起来，因为它们没有上下文指针，因此它们不是真正的闭包）。 Swift 提供了从 `@convention(c)` 闭包到具有上下文的 Swift 闭包的隐式转换。

导入 C 函数指针时还需要考虑指针的 nullability：可空 C 函数指针作为可选类型导入。

```C
// C header.

void qsort(
  void *base,
  size_t nmemb,
  size_t size,
  int (*compar)(const void *, const void *));

void qsort_annotated(
  void * _Nonnull base,
  size_t nmemb,
  size_t size,
  int (* _Nonnull compar)(const void * _Nonnull, const void * _Nonnull));
```

```Swift
// C header imported in Swift.

func qsort(
  _ base: UnsafeMutableRawPointer!,
  _ nmemb: Int,
  _ size: Int,
  _ compar: (@convention(c) (UnsafeRawPointer?, UnsafeRawPointer?) -> CInt)!
)

func qsort_annotated(
  _ base: UnsafeMutableRawPointer,
  _ nmemb: Int,
  _ size: Int,
  _ compar: @convention(c) (UnsafeRawPointer, UnsafeRawPointer) -> CInt
)
```

另请参阅有关主题的Apple文档：[在Swift中使用导入的C函数](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/using_imported_c_functions_in_swift)

# Fixed-size arrays
C 的固定大小数组作为 Swift 元组导入。

```C
// C header.

extern int x[4];
```

```Swift
// C header imported in Swift.

var x: (CInt, CInt, CInt, CInt) { get set }
```

由于 Swift 元组的结构（ergonomics）与 C 的固定大小的数组不匹配，因此这种映射策略远非最佳。例如，不能通过仅在运行时才知道的索引来访问 Swift 元组。如果需要按索引访问元组元素，则必须使用 `withUnsafeMutablePointer(&myTuple) { ... }` 获取指向同质（homogeneous）元组中值的不安全指针，然后执行指针计算。

固定大小的数组是 Swift 中常见的特性，这个好的提议很可能被接受。一旦 Swift 在语言中使用固定大小的数组，我们就可以使用它们来提高与 C 的互通性。

# Structs
C 结构体作为 Swift 结构体导入，C结构体的字段映射为 Swfit 结构体中的存储属性。C 结构体的位域映射为 Swift 结构体中的计算属性。Swift 结构体还有一个默认初始化器（将所有属性设置为零）和一个元素初始化器（将所有属性设置为提供的值）。

```C
// C header.

struct Point {
  int x;
  int y;
};
struct Line {
  struct Point start;
  struct Point end;
  unsigned int brush : 4;
  unsigned int stroke : 3;
};
```

```Swift
// C header imported in Swift.

struct Point {
  var x: CInt { get set }
  var y: CInt { get set }
  init()
  init(x: CInt, y: CInt)
}

struct Line {
  var start: Point { get set }
  var end: Point { get set }
  var brush: CUnsignedInt { get set }
  var stroke: CUnsignedInt { get set }

  // Default initializer that sets all properties to zero.
  init()

  // Elementwise initializer.
  init(start: Point, end: Point, brush: CUnsignedInt, stroke: CUnsignedInt)
}
```

Swift 还可以导入未命名的和匿名的结构体。

```C
// C header.

struct StructWithAnonymousStructs {
  struct {
    int x;
  };
  struct {
    int y;
  } containerForY;
};
```

```Swift
// C header imported in Swift.
struct StructWithAnonymousStructs {
  struct __Unnamed_struct___Anonymous_field0 {
    var x: CInt
    init()
    init(x: CInt)
  }
  struct __Unnamed_struct_containerForY {
    var y: CInt
    init()
    init(y: CInt)
  }
  var __Anonymous_field0: StructWithAnonymousStructs.__Unnamed_struct___Anonymous_field0
  var x: CInt
  var containerForY: StructWithAnonymousStructs.__Unnamed_struct_containerForY

  // Default initializer that sets all properties to zero.
  init()

  // Elementwise initializer.
  init(
    _ __Anonymous_field0: StructWithAnonymousStructs.__Unnamed_struct___Anonymous_field0,
    containerForY: StructWithAnonymousStructs.__Unnamed_struct_containerForY
  )
}
```

另请参阅有关主题的Apple文档：[在 Swift 中使用导入的 C 结构体和联合体](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/using_imported_c_structs_and_unions_in_swift)

# Unions
Swift 没有直接等同于 C union 的类型。C 的 union 被映射为带有计算属性的 Swift 结构体，这些属性读取/写入相同的后端存储（underlying storage）。

```C
// C header.

union IntOrFloat {
  int i;
  float f;
};
```

```Swift
// C header imported in Swift.

struct IntOrFloat {
  var i: CInt { get set } // Computed property.
  var f: CFloat { get set } // Computed property.
  init(i: CInt)
  init(f: CFloat)
  init()
}
```

另请参阅有关主题的Apple文档：[在 Swift 中使用导入的 C 结构体和联合体](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/using_imported_c_structs_and_unions_in_swift)

# Enums
我们希望将 C 枚举映射为 Swift 枚举，如下所示：

```C
// C header.

// Enum that is not explicitly marked as either open or closed.
enum HomeworkExcuse {
  EatenByPet,
  ForgotAtHome,
  ThoughtItWasDueNextWeek,
};
```

```Swift
// C header imported in Swift: aspiration, not an actual mapping!

enum HomeworkExcuse: CUnsignedInt {
  case EatenByPet
  case ForgotAtHome
  case ThoughtItWasDueNextWeek
}
```

实际上，普通 C 枚举被映射为 Swift 结构体，如下所示：

```Swift
// C header imported in Swift: actual mapping.

struct HomeworkExcuse: Equatable, RawRepresentable {
  init(_ rawValue: CUnsignedInt)
  init(rawValue: CUnsignedInt)
  var rawValue: CUnsignedInt { get }
  typealias RawValue = CUnsignedInt
}
var EatenByPet: HomeworkExcuse { get }
var ForgotAtHome: HomeworkExcuse { get }
var ThoughtItWasDueNextWeek: HomeworkExcuse { get }
```

为了解释为什么选择此映射，我们需要讨论 C 枚举的某些特征：

* 在 C 语言中，添加新的枚举数不会破坏源代码。 在 Itanium C ABI 中，只要枚举大小不变，ABI 也不会破坏。因此，C 库在添加枚举值时不会破坏向下兼容。
* 在 C 语言中，使用位域来实现枚举的功能：枚举器是按2的次数幂分配的值，枚举值是以枚举器做或操作组合而来的。
* 一些 C API 定义了两组枚举器：头文件中列出的“public”枚举器和仅在库的实现中使用的“private”枚举器。


基于这些编码模式，在运行时，C 枚举可以携带编译时枚举声明中没有列出的值（这些值要么是在代码编译后添加的，要么是有意将整数转换为枚举的结果）。

Swift 编译器会对 switch 语句执行穷举检查，这在 C 的枚举上执行时就会出现问题，因为对穷举的期望是不同的。

从 Swift 的角度来看，C 枚举有两种形式：closed 对应 frozen, open 对应 non-frozen。这种区别旨在支持库的演化和 ABI 的稳定性，同时允许用户按照人性化（ergonomically）的方式处理其代码。Swift 的解决方案还支持非常规的 C 枚举编码模式。

Frozen enums 有一组固定的用例（用 C 表示的枚举）。库的提供者可以在 frozen enum 中更改（添加或删除）用例，但是，这会破坏 ABI 和源代码。换句话说，可以保证在编译时看到的用例与运行时枚举变量可以携带的值集完全匹配。Swift 对 frozen enum 的 switch 语句执行穷举：如果 switch 不能处理所有枚举情况，那么用户就会收到警告。此外，优化器可以假定具有 frozen enum 类型的变量将仅存储与在编译时可见的枚举实例相对应的值；未使用的位模式可以重新用于其他的目的。

Non-frozen enums 具有一组可扩展的情况。库提供者可以在不破坏 ABI 或源代码兼容性的情况下添加用例。Swift 对 non-frozen enums 的 switch 语句执行不同类型的穷举性检查：它始终需要 `@unknown default` 子句，但仅在代码未处理编译时可用的所有情况时才产生警告。

```C
// C header.

// An open enum: we expect to add more kinds of input devices in future.
enum InputDevice {
  Keyboard,
  Mouse,
  Touchscreen,
} __attribute__((enum_extensibility(open)));
// A closed enum: we think we know enough about the geometry of Earth to
// confidently say that these are all cardinal directions we will ever need.
enum CardinalDirection {
  East,
  West,
  North,
  South,
} __attribute__((enum_extensibility(closed)));
```

```Swift
// C header imported in Swift.

enum InputDevice: CUnsignedInt, Hashable, RawRepresentable {
  init?(rawValue: CUnsignedInt)
  var rawValue: CUnsignedInt { get }
  typealias RawValue = CUnsignedInt
  case Keyboard
  case Mouse
  case Touchscreen
}

@frozen
enum CardinalDirection: CUnsignedInt, Hashable, RawRepresentable {
  init?(rawValue: CUnsignedInt)
  var rawValue: CUnsignedInt { get }
  typealias RawValue = CUnsignedInt
  case East
  case West
  case North
  case South
}
```

自 Swift 1.0 起，未标记为 open 或 closed 的 C 枚举会被映射为结构体。那时，我们意识到，如果 Swift 编译器对此类导入的枚举做出类似 Swift 的假设，那么将 C 枚举映射到 Swift 枚举是不正确的。具体地说，Swift 编译器假设枚举值只允许在枚举的情况下声明位模式。该假设对 frozen Swift 枚举有效（这是 Swift 1.0 中唯一的枚举形式）。然而，这个假设并不适用于 C 枚举，因为任何位模式都是一个有效值；这种假设产生了一个未定义的危险行为。为了解决这个问题，在 Swift 1.0 中，默认情况下将 C 枚举作为 Swift 结构体导入，并将其枚举器做为全局计算变量。用 NS_ENUM 声明的 Objective-C 枚举假设为有“枚举特性（enum nature）”，并可以被导入为 Swift 枚举。

在 Swift 5 中增加了开放枚举（open enums）的概念（[SE-0192 Handling Future Enum Cases](https://github.com/apple/swift-evolution/blob/master/proposals/0192-non-exhaustive-enums.md)），但该提议并没有改变 non-annotated C 枚举的导入策略，部分原因是出于源代码兼容性的考虑。可能仍然可以将 C 枚举更改为开放的 Swift 枚举，但是随着时间的流逝，更改将更加困难。

C 枚举的另一个特性是它们向用户公开整数值；此外，枚举值可以隐式转换为整数。Swift 枚举默认情况下是不透明的。在导入到 Swift 时，C 枚举遵循 `RawRepresentable` 协议，从而允许用户显式地在数值和类型值之间转换。

```Swift
// Converting enum values to integers and back.

var south: CardinalDirection = .South
// var southAsInteger: CUnsignedInt = south // error: type mismatch
var southAsInteger: CUnsignedInt = south.rawValue // = 3
var southAsEnum = CardinalDirection(rawValue: 3) // = .South
```

# Typedefs
除了一些用特殊方式处理的常见 C 编码模式外，C 的 typedef 通常映射为 Swift 的类型别名（typealias）。

```C
// C header.

// An ordinary typedef.
typedef int Money;

// A special case pattern that is mapped to a named struct.
typedef struct {
  int x;
  int y;
} Point;
```

```Swift
// C header imported in Swift.

typealias Money = Int

struct Point {
  var x: CInt { get set }
  var y: CInt { get set }
  init()
  init(x: CInt, y: CInt)
}
```

# Macros
C 宏通常不会在 Swift 中导入。定义常量的宏将作为只读变量导入。

```C
// C header.

#define BUFFER_SIZE 4096
#define SERVER_VERSION "3.14"
```

```Swift
// C header imported in Swift.

var BUFFER_SIZE: CInt { get }
var SERVER_VERSION: String { get }
```

另请参阅有关主题的Apple文档：[在 Swift 中使用导入的 C 宏](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/using_imported_c_macros_in_swift)

# 补充阅读
以下是翻译过程中参考的一些文章

[Swift5 Frozen enums](https://useyourloaf.com/blog/swift-5-frozen-enums/)

[Swift imports fixed-size C arrays as tuples](https://oleb.net/blog/2017/12/swift-imports-fixed-size-c-arrays-as-tuples/)

[Size, Stride, Alignment](https://swiftunboxed.com/internals/size-stride-alignment/)
