---

layout: post

title: "struct 也能里氏替换？Swift 中的 UnsafeMutablePointer 为何能替换 UnsafePointer"

---

如果有人问你为什么在 Objective-C 中，一个参数类型为 `NSArray` 的方法可以传入类型为 `NSMutableArray` 的对象，你一定会下意识觉得这个人是个初学者。原因很简单，`NSMutableArray` 继承自 `NSArray`，子类可以替换其父类对象是面向对象思想 [SOLID](https://zh.wikipedia.org/zh-cn/SOLID_(面向对象设计)) 中的[里氏替换原则](https://zh.wikipedia.org/zh-cn/里氏替换原则)。这些基础知识和思想可能是每个面向对象设计课程中都会涉及到的。

因此，当我发现给 `UnsafeBufferPointer` 的构造方法 `init(start: UnsafePointer<Element>?, count: Int)` 传入一个类型为 `UnsafeMutablePointer<Element>` 的对象一切正常时并没有太过注意，直觉上认为类似于 Objective-C 的风格， `UnsafeMutablePointer<Pointee>` 自然是继承自 `UnsafePointer<Pointee>`。直到我查看 API 时才发现，不只是 `UnsafePointer`，所有的 Pointer 都是 struct 定义的。



## 0 背景知识

虽然在平时在 Swift 的使用中较少被用到，但一旦涉及到与 C 等语言的 API 调用，各种 Pointer 的出场率还是挺高的。根据指向的内容是否可变（Mutable）/指向内容是否有明确类型（Raw）/指向的是单个内容还是多个内容连续排列（Buffer），Swift 中的 Pointer 总共有 pow(2, 3) = 8 种不同的组合：

|                               | 可变？ | 类型明确？ | 连续？ |
| :---------------------------: | :----: | :--------: | :----: |
|         UnsafePointer         |   N    |     Y      |   N    |
|     UnsafeMutablePointer      |   Y    |     Y      |   N    |
|       UnsafeRawPointer        |   N    |     N      |   N    |
|    UnsafeMutableRawPointer    |   Y    |     N      |   N    |
|      UnsafeBufferPointer      |   N    |     Y      |   Y    |
|  UnsafeMutableBufferPointer   |   Y    |     Y      |   Y    |
|    UnsafeRawBufferPointer     |   N    |     N      |   Y    |
| UnsafeMutableRawBufferPointer |   Y    |     N      |   Y    |

如果只从公开的 API 来看，其中每一个类型都是互相独立的 struct，例如 `struct UnsafePointer<Pointee>`，所以理论上下面的类型判断和类型转换都不成立：

```swift
var value = 10

withUnsafeMutablePointer(to: &value) { mutablePointer in
  mutablePointer is UnsafePointer<Int> // warning: Cast from 'UnsafeMutablePointer<Int>' to unrelated type 'UnsafePointer<Int>' always fails
                                      
  if let pointer = mutablePointer as? UnsafePointer<Int> { // warning: Cast from 'UnsafeMutablePointer<Int>' to unrelated type 'UnsafePointer<Int>' always fails
  }

  // call UnsafeBufferPointer.init(start: UnsafePointer<Element>?, count: Int)
  let bufferPointer = UnsafeBufferPointer<Int>(start: mutablePointer, count: 1) // UnsafeBufferPointer(start: 0x00000001055e4320, count: 1)
  print(bufferPointer.baseAddress?.pointee) // Optional(10)
}
```

但实际运行证明，`bufferPointer` 确实被成功构造了。



## 1. 构造方法有猫腻？

有没有可能 `UnsafeBufferPointer` 有两个名称相同，只是类型不同的构造方法？类似于：

```swift
UnsafeBufferPointer.init(start: UnsafePointer<Element>?, count: Int)
UnsafeBufferPointer.init(start: UnsafeMutablePointer<Element>?, count: Int)
```

然而从 API 看，并没有。而且同样的，即使是自定义的方法也一样有类似的机制：

```swift
func logPointer<T>(at pointer: UnsafePointer<T>) {
  print(pointer.pointee)
}

var value = 10
withUnsafeMutablePointer(to: &value) { mutablePointer in
  logPointer(at: mutablePointer) // 10
}
```



## 2. 看看编译器怎么说

既然代码能成功通过编译，说明编译的各个流程都是没问题的。靠后的 .o 文件生成、链接等流程给我们的帮助不大，或许可以通过语法树的生成过程找到些蛛丝马迹 （AST 等相关的更多信息，参考 [Swift Compiler](https://www.swift.org/swift-compiler/)）。为了对比，同时加上一个不用 Pointer 的类似逻辑。

```swift
// main.swift
var value = 10

func log<T>(value: T) {
  print(value)
}

log(value: value)

func logPointer<T>(at pointer: UnsafePointer<T>) {
  print(pointer.pointee)
}

withUnsafeMutablePointer(to: &value) { mutablePointer in
  logPointer(at: mutablePointer)
}
```



在终端执行

```shell
swiftc -dump-ast main.swift
```



得到 AST 数据：

```
(source_file "main.swift"
  (top_level_code_decl range=[main.swift:1:1 - line:1:13]
    (brace_stmt implicit range=[main.swift:1:1 - line:1:13]
      (pattern_binding_decl range=[main.swift:1:1 - line:1:13]
        (pattern_named type='Int' 'value')
        Original init:
        (integer_literal_expr type='Int' location=main.swift:1:13 range=[main.swift:1:13 - line:1:13] value=10 builtin_initializer=Swift.(file).Int.init(_builtinIntegerLiteral:) initializer=**NULL**)
        Processed init:
        (integer_literal_expr type='Int' location=main.swift:1:13 range=[main.swift:1:13 - line:1:13] value=10 builtin_initializer=Swift.(file).Int.init(_builtinIntegerLiteral:) initializer=**NULL**))
))
  (var_decl range=[main.swift:1:5 - line:1:5] "value" type='Int' interface type='Int' access=internal readImpl=stored writeImpl=stored readWriteImpl=stored)
  (func_decl range=[main.swift:3:1 - line:5:1] "log(value:)" <T> interface type='<T> (value: T) -> ()' access=internal captures=(<generic> )
    (parameter_list range=[main.swift:3:12 - line:3:21]
      (parameter "value" apiName=value type='T' interface type='T'))
    (call_expr type='()' location=main.swift:4:3 range=[main.swift:4:3 - line:4:14] nothrow
      (declref_expr type='(Any..., String, String) -> ()' location=main.swift:4:3 range=[main.swift:4:3 - line:4:3] decl=Swift.(file).print(_:separator:terminator:) function_ref=single)
      (argument_list labels=_:separator:terminator:
        (argument
          (vararg_expansion_expr implicit type='Any...' location=main.swift:4:9 range=[main.swift:4:9 - line:4:9]
            (array_expr implicit type='Any...' location=main.swift:4:9 range=[main.swift:4:9 - line:4:9] initializer=**NULL**
              (erasure_expr implicit type='Any' location=main.swift:4:9 range=[main.swift:4:9 - line:4:9]
                (declref_expr type='T' location=main.swift:4:9 range=[main.swift:4:9 - line:4:9] decl=main.(file).log(value:).value@main.swift:3:13 function_ref=unapplied)))))
        (argument label=separator
          (default_argument_expr implicit type='String' location=main.swift:4:8 range=[main.swift:4:8 - line:4:8] default_args_owner=Swift.(file).print(_:separator:terminator:) param=1))
        (argument label=terminator
          (default_argument_expr implicit type='String' location=main.swift:4:8 range=[main.swift:4:8 - line:4:8] default_args_owner=Swift.(file).print(_:separator:terminator:) param=2))
      )))
  (top_level_code_decl range=[main.swift:7:1 - line:7:17]
    (brace_stmt implicit range=[main.swift:7:1 - line:7:17]
      (call_expr type='()' location=main.swift:7:1 range=[main.swift:7:1 - line:7:17] nothrow
        (declref_expr type='(Int) -> ()' location=main.swift:7:1 range=[main.swift:7:1 - line:7:1] decl=main.(file).log(value:)@main.swift:3:6 [with (substitution_map generic_signature=<T> (substitution T -> Int))] function_ref=single)
        (argument_list labels=value:
          (argument label=value
            (load_expr implicit type='Int' location=main.swift:7:12 range=[main.swift:7:12 - line:7:12]
              (declref_expr type='@lvalue Int' location=main.swift:7:12 range=[main.swift:7:12 - line:7:12] decl=main.(file).value@main.swift:1:5 function_ref=unapplied)))
        ))))
  (func_decl range=[main.swift:9:1 - line:11:1] "logPointer(at:)" <T> interface type='<T> (at: UnsafePointer<T>) -> ()' access=internal captures=(<generic> )
    (parameter_list range=[main.swift:9:19 - line:9:48]
      (parameter "pointer" apiName=at type='UnsafePointer<T>' interface type='UnsafePointer<T>'))
    (call_expr type='()' location=main.swift:10:3 range=[main.swift:10:3 - line:10:24] nothrow
      (declref_expr type='(Any..., String, String) -> ()' location=main.swift:10:3 range=[main.swift:10:3 - line:10:3] decl=Swift.(file).print(_:separator:terminator:) function_ref=single)
      (argument_list labels=_:separator:terminator:
        (argument
          (vararg_expansion_expr implicit type='Any...' location=main.swift:10:9 range=[main.swift:10:9 - line:10:17]
            (array_expr implicit type='Any...' location=main.swift:10:9 range=[main.swift:10:9 - line:10:17] initializer=**NULL**
              (erasure_expr implicit type='Any' location=main.swift:10:17 range=[main.swift:10:9 - line:10:17]
                (member_ref_expr type='T' location=main.swift:10:17 range=[main.swift:10:9 - line:10:17] decl=Swift.(file).UnsafePointer.pointee [with (substitution_map generic_signature=<Pointee> (substitution Pointee -> T))]
                  (declref_expr type='UnsafePointer<T>' location=main.swift:10:9 range=[main.swift:10:9 - line:10:9] decl=main.(file).logPointer(at:).pointer@main.swift:9:23 function_ref=unapplied))))))
        (argument label=separator
          (default_argument_expr implicit type='String' location=main.swift:10:8 range=[main.swift:10:8 - line:10:8] default_args_owner=Swift.(file).print(_:separator:terminator:) param=1))
        (argument label=terminator
          (default_argument_expr implicit type='String' location=main.swift:10:8 range=[main.swift:10:8 - line:10:8] default_args_owner=Swift.(file).print(_:separator:terminator:) param=2))
      )))
  (top_level_code_decl range=[main.swift:13:1 - line:15:1]
    (brace_stmt implicit range=[main.swift:13:1 - line:15:1]
      (call_expr type='()' location=main.swift:13:1 range=[main.swift:13:1 - line:15:1] nothrow
        (declref_expr type='(inout Int, (UnsafeMutablePointer<Int>) throws -> ()) throws -> ()' location=main.swift:13:1 range=[main.swift:13:1 - line:13:1] decl=Swift.(file).withUnsafeMutablePointer(to:_:) [with (substitution_map generic_signature=<T, Result> (substitution T -> Int) (substitution Result -> ()))] function_ref=single)
        (argument_list labels=to:_:
          (argument label=to inout
            (inout_expr type='inout Int' location=main.swift:13:30 range=[main.swift:13:30 - line:13:31]
              (declref_expr type='@lvalue Int' location=main.swift:13:31 range=[main.swift:13:31 - line:13:31] decl=main.(file).value@main.swift:1:5 function_ref=unapplied)))
          (argument
            (function_conversion_expr implicit type='(UnsafeMutablePointer<Int>) throws -> ()' location=main.swift:13:38 range=[main.swift:13:38 - line:15:1]
              (closure_expr type='(UnsafeMutablePointer<Int>) -> ()' location=main.swift:13:38 range=[main.swift:13:38 - line:15:1] discriminator=0 single-expression
                (parameter_list range=[main.swift:13:40 - line:13:40]
                  (parameter "mutablePointer" type='UnsafeMutablePointer<Int>' interface type='UnsafeMutablePointer<Int>'))
                (brace_stmt range=[main.swift:13:38 - line:15:1]
                  (return_stmt implicit range=[main.swift:14:3 - line:14:32]
                    (call_expr type='()' location=main.swift:14:3 range=[main.swift:14:3 - line:14:32] nothrow
                      (declref_expr type='(UnsafePointer<Int>) -> ()' location=main.swift:14:3 range=[main.swift:14:3 - line:14:3] decl=main.(file).logPointer(at:)@main.swift:9:6 [with (substitution_map generic_signature=<T> (substitution T -> Int))] function_ref=single)
                      (argument_list labels=at:
                        (argument label=at
                          (pointer_to_pointer implicit type='UnsafePointer<Int>' location=main.swift:14:18 range=[main.swift:14:18 - line:14:18]
                            (declref_expr type='UnsafeMutablePointer<Int>' location=main.swift:14:18 range=[main.swift:14:18 - line:14:18] decl=main.(file).top-level code.explicit closure discriminator=0.mutablePointer@main.swift:13:40 function_ref=unapplied)))
                      )))))))
        )))))
```

乍一看确实容易被长度吓住，但两个版本的同一段大致结构是类似的，我们只看两版本 log 方法的调用部分：

```
// func log<T>(value: T)
(call_expr type='()' location=main.swift:7:1 range=[main.swift:7:1 - line:7:17] nothrow
        (declref_expr type='(Int) -> ()' location=main.swift:7:1 range=[main.swift:7:1 - line:7:1] decl=main.(file).log(value:)@main.swift:3:6 [with (substitution_map generic_signature=<T> (substitution T -> Int))] function_ref=single)
        (argument_list labels=value:
          (argument label=value
            (load_expr implicit type='Int' location=main.swift:7:12 range=[main.swift:7:12 - line:7:12]
              (declref_expr type='@lvalue Int' location=main.swift:7:12 range=[main.swift:7:12 - line:7:12] decl=main.(file).value@main.swift:1:5 function_ref=unapplied)))
        ))


// func logPointer<T>(at pointer: UnsafePointer<T>)
(call_expr type='()' location=main.swift:14:3 range=[main.swift:14:3 - line:14:32] nothrow
                      (declref_expr type='(UnsafePointer<Int>) -> ()' location=main.swift:14:3 range=[main.swift:14:3 - line:14:3] decl=main.(file).logPointer(at:)@main.swift:9:6 [with (substitution_map generic_signature=<T> (substitution T -> Int))] function_ref=single)
                      (argument_list labels=at:
                        (argument label=at
                          (pointer_to_pointer implicit type='UnsafePointer<Int>' location=main.swift:14:18 range=[main.swift:14:18 - line:14:18]
                            (declref_expr type='UnsafeMutablePointer<Int>' location=main.swift:14:18 range=[main.swift:14:18 - line:14:18] decl=main.(file).top-level code.explicit closure discriminator=0.mutablePointer@main.swift:13:40 function_ref=unapplied)))
                      ))
```



再具体点，只看参数部分：

```
// func log<T>(value: T)
(argument label=value
            (load_expr implicit type='Int' location=main.swift:7:12 range=[main.swift:7:12 - line:7:12]
              (declref_expr type='@lvalue Int' location=main.swift:7:12 range=[main.swift:7:12 - line:7:12] decl=main.(file).value@main.swift:1:5 function_ref=unapplied)))
              
              
// func logPointer<T>(at pointer: UnsafePointer<T>)
(argument label=at
                          (pointer_to_pointer implicit type='UnsafePointer<Int>' location=main.swift:14:18 range=[main.swift:14:18 - line:14:18]
                            (declref_expr type='UnsafeMutablePointer<Int>' location=main.swift:14:18 range=[main.swift:14:18 - line:14:18] decl=main.(file).top-level code.explicit closure discriminator=0.mutablePointer@main.swift:13:40 function_ref=unapplied)))
```



`pointer_to_pointer` 这个操作看上去就是问题的关键了，似乎就是这一步将原本的 `UnsafeMutablePointer<Int>` 转成了方法所需的 `UnsafePointer<Int>`。我们就以此为关键词在 Swift 的仓库中[搜索](https://github.com/apple/swift/search?q=PointerToPointer)。

从 [Expr.h](https://github.com/apple/swift/blob/a66951aa3da3c0b925fef8ae205a5e7d04c2c271/include/swift/AST/Expr.h#L2934) 的定义中我们可以看出，PointerToPointer 隶属于隐式转换操作：

```cpp
/// Convert a pointer to a different kind of pointer.
class PointerToPointerExpr : public ImplicitConversionExpr {
public:
  PointerToPointerExpr(Expr *subExpr, Type ty)
    : ImplicitConversionExpr(ExprKind::PointerToPointer, subExpr, ty) {}
  
  static bool classof(const Expr *E) {
    return E->getKind() == ExprKind::PointerToPointer;
  }
};
```

从 [KnownDecls.def](https://github.com/apple/swift/blob/7123d2614b5f222d03b3762cb110d27a9dd98e24/include/swift/AST/KnownDecls.def#L33) 中可以看出，这个操作实际上调用的是名为 `_convertPointerToPointerArgument` 的私有方法：

```c
FUNC_DECL(ConvertPointerToPointerArgument,
          "_convertPointerToPointerArgument")
```

我们继续[查找](https://github.com/apple/swift/search?q=_convertPointerToPointerArgument)这个方法，可以找到这是定义在 Pointer.swift 文件中的全局方法：

```swift
/// Derive a pointer argument from a convertible pointer type.
@_transparent
public // COMPILER_INTRINSIC
func _convertPointerToPointerArgument<
  FromPointer: _Pointer,
  ToPointer: _Pointer
>(_ from: FromPointer) -> ToPointer {
  return ToPointer(from._rawValue)
}
```



## 3. 私有协议 _Pointer

查到这里答案就呼之欲出了，看似没有任何关系的各种 Pointer，实际上都实现了私有协议 `_Pointer`。再准确点说，非 Buffer 的四种指针实现了这个协议。

```swift
public protocol _Pointer
: Hashable, Strideable, CustomDebugStringConvertible, _CustomReflectableOrNone {
  /// A type that represents the distance between two pointers.
  typealias Distance = Int
  
  associatedtype Pointee

  /// The underlying raw pointer value.
  var _rawValue: Builtin.RawPointer { get }

  /// Creates a pointer from a raw value.
  init(_ _rawValue: Builtin.RawPointer)
}
```

本质上这四种指针都只是对私有类型 `Builtin.RawPointer` 的包装，因此在 `_convertPointerToPointerArgument` 方法的实现中，也是通过 `from._rawValue` 构造了新的具体 Pointer 类型。

没有实现 `_Pointer` 协议的四种 BufferPointer 自然就没有类似的隐式转换机制了。本质上，BufferPointer 只是包装了一个 Pointer 作为 `baseAddress` 的容器，存在的价值更多在于实现了 `Collection` 相关协议，便于像操作集合一样操作连续的指针。



## 4. 特殊情况的处理

### 4.1 反向转换

既然都是对于 `Builtin.RawPointer` 的包装，理论上可以自由地互相转换。但稍微考虑一下，把可变指针当做不可变指针使用并无不妥，但反过来对于不可变指针做修改就是一个很有风险的操作了，Swift 如何避免出现这种情况下的隐式转换呢？

答案是在语义分析时，只针对特定的类型使用 pointer_to_pointer [规则](https://github.com/apple/swift/blob/983e2f37d35b52be880e0433565fe8aa0091b8eb/lib/Sema/CSSimplify.cpp#L6710)：

```cpp
PointerTypeKind type1PointerKind;
bool type1IsPointer{
    unwrappedType1->getAnyPointerElementType(type1PointerKind)};
bool optionalityMatches = !type1IsOptional || type2IsOptional;
if (type1IsPointer && optionalityMatches) {
  if (type1PointerKind == PTK_UnsafeMutablePointer) {
    // Favor an UnsafeMutablePointer-to-UnsafeMutablePointer
    // conversion.
    if (type1PointerKind != pointerKind)
      increaseScore(ScoreKind::SK_ValueToPointerConversion);
    conversionsOrFixes.push_back(
      ConversionRestrictionKind::PointerToPointer);
  }
  // UnsafeMutableRawPointer -> UnsafeRawPointer
  else if (type1PointerKind == PTK_UnsafeMutableRawPointer &&
           pointerKind == PTK_UnsafeRawPointer) {
    if (type1PointerKind != pointerKind)
      increaseScore(ScoreKind::SK_ValueToPointerConversion);
    conversionsOrFixes.push_back(
      ConversionRestrictionKind::PointerToPointer);              
  }
}
```

### 4.2 Pointee 类型不同

Pointer 除了自身的类型，对于明确指向内容类型的 Pointer 也还有着关联类型 `Pointee`，那么对于 Pointee 类型无关的情况是怎么处理的呢？

答案是 Pointee 的类型同样要参与能否转换的[判断](https://github.com/apple/swift/blob/983e2f37d35b52be880e0433565fe8aa0091b8eb/lib/Sema/CSSimplify.cpp#L11939)：

```cpp
static llvm::PointerIntPair<Type, 3, unsigned>
getBaseTypeForPointer(TypeBase *type) {
  unsigned unwrapCount = 0;
  while (auto objectTy = type->getOptionalObjectType()) {
    type = objectTy.getPointer();
    ++unwrapCount;
  }

  auto pointeeTy = type->getAnyPointerElementType();
  assert(pointeeTy);
  return {pointeeTy, unwrapCount};
}

// line 11285
// T <p U ===> UnsafeMutablePointer<T> <a UnsafeMutablePointer<U>
case ConversionRestrictionKind::PointerToPointer: {
  auto t1 = type1->getDesugaredType();
  auto t2 = type2->getDesugaredType();

  auto ptr1 = getBaseTypeForPointer(t1);
  auto ptr2 = getBaseTypeForPointer(t2);

  return matchPointerBaseTypes(ptr1, ptr2);
}
```



## 5. 隐式转换虽然不显眼，其实很常见

Swift 作为一个静态类型+强类型的语言，很多情况下如果没有隐式转换的帮助，强类型就不再是安全的保证而是编码的负担，举个例子：

```swift
func foo(i: Int?) {
  print(i ?? -1)
}

var value = 10
foo(i: value)

// without implicit conversion 
foo(i: Int?(value))
// or
foo(i: .some(value))
```

`foo(i:)` 函数需要的参数类型是 `Optional<Int>` ，但将类型为 `Int` 的 `value` 传入也并无问题，这其中就是名为自动装包（inject_into_optional）的隐式转换在发挥作用：

```
(inject_into_optional implicit type='Int?' location=main.swift:6:8 range=[main.swift:6:8 - line:6:8]
              (load_expr implicit type='Int' location=main.swift:6:8 range=[main.swift:6:8 - line:6:8]
                (declref_expr type='@lvalue Int' location=main.swift:6:8 range=[main.swift:6:8 - line:6:8] decl=main.(file).value@main.swift:5:5 function_ref=unapplied)))
```

同样的，实现基于类继承的里氏替换同样是一个隐式转换操作（derived_to_base_expr）：

```swift
class Sup {
  var value: Int = 0
}

class Sub: Sup { }

func foo(i: Sup) {
  print(sub)
}

let sub = Sub()
sub.value = 10
foo(i: sub)

// AST
(derived_to_base_expr implicit type='Sup' location=main.swift:15:8 range=[main.swift:15:8 - line:15:8]
              (declref_expr type='Sub' location=main.swift:15:8 range=[main.swift:15:8 - line:15:8] decl=main.(file).sub@main.swift:13:5 function_ref=unapplied))
```

block 与 function 之间的转换也是（function_conversion_expr）：

```swift
func foo(block: (Int) -> Int) {
  print(block(10))
}

func addOne(value: Int) -> Int {
  return value + 1
}

foo(block: addOne(value:))

// AST
(call_expr type='()' location=main.swift:9:1 range=[main.swift:9:1 - line:9:26] nothrow
        (declref_expr type='((Int) -> Int) -> ()' location=main.swift:9:1 range=[main.swift:9:1 - line:9:1] decl=main.(file).foo(block:)@main.swift:1:6 function_ref=single)
        (argument_list labels=block:
          (argument label=block
            (function_conversion_expr implicit type='(Int) -> Int' location=main.swift:9:12 range=[main.swift:9:12 - line:9:25]
              (declref_expr type='(Int) -> Int' location=main.swift:9:12 range=[main.swift:9:12 - line:9:25] decl=main.(file).addOne(value:)@main.swift:5:6 function_ref=compound)))
        ))
```



所有支持的隐式转换操作都可以在 [ExprNodes.def](https://github.com/apple/swift/blob/a66951aa3da3c0b925fef8ae205a5e7d04c2c271/include/swift/AST/ExprNodes.def#L154) 的定义中找到：

```c
EXPR(Load, ImplicitConversionExpr)
EXPR(ABISafeConversion, ImplicitConversionExpr)
EXPR(DestructureTuple, ImplicitConversionExpr)
EXPR(UnresolvedTypeConversion, ImplicitConversionExpr)
EXPR(FunctionConversion, ImplicitConversionExpr)
EXPR(CovariantFunctionConversion, ImplicitConversionExpr)
EXPR(CovariantReturnConversion, ImplicitConversionExpr)
EXPR(MetatypeConversion, ImplicitConversionExpr)
EXPR(CollectionUpcastConversion, ImplicitConversionExpr)
EXPR(Erasure, ImplicitConversionExpr)
EXPR(AnyHashableErasure, ImplicitConversionExpr)
EXPR(BridgeToObjC, ImplicitConversionExpr)
EXPR(BridgeFromObjC, ImplicitConversionExpr)
EXPR(ConditionalBridgeFromObjC, ImplicitConversionExpr)
EXPR(DerivedToBase, ImplicitConversionExpr)
EXPR(ArchetypeToSuper, ImplicitConversionExpr)
EXPR(InjectIntoOptional, ImplicitConversionExpr)
EXPR(ClassMetatypeToObject, ImplicitConversionExpr)
EXPR(ExistentialMetatypeToObject, ImplicitConversionExpr)
EXPR(ProtocolMetatypeToObject, ImplicitConversionExpr)
EXPR(InOutToPointer, ImplicitConversionExpr)
EXPR(ArrayToPointer, ImplicitConversionExpr)
EXPR(StringToPointer, ImplicitConversionExpr)
EXPR(PointerToPointer, ImplicitConversionExpr)
EXPR(ForeignObjectConversion, ImplicitConversionExpr)
EXPR(UnevaluatedInstance, ImplicitConversionExpr)
EXPR(UnderlyingToOpaque, ImplicitConversionExpr)
EXPR(DifferentiableFunction, ImplicitConversionExpr)
EXPR(LinearFunction, ImplicitConversionExpr)
EXPR(DifferentiableFunctionExtractOriginal, ImplicitConversionExpr)
EXPR(LinearFunctionExtractOriginal, ImplicitConversionExpr)
EXPR(LinearToDifferentiableFunction, ImplicitConversionExpr)
EXPR(ReifyPack, ImplicitConversionExpr)
```

