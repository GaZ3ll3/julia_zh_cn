.. _man-calling-c-and-fortran-code:

************************
 调用 C 和 Fortran 代码
************************
尽管大多数的代码都可以用 Julia 来实现， 但是目前很多成熟的数值计算库是用 C 和 Fortran 实现的。 为了方便地使用这些已有的代码， Julia 可以简单高效地调用 C 和 Fortran 函数。 在Julia 里， 使用 :func:`ccall` 可以像对待普通的函数一样来调用 C 和 Fortran 代码。

被调用的代码应该是共享库的格式。大多数 C 和 Fortran 库都已经被编译为共享库。如果自己使用 GCC （或 Clang ）编译代码，需要添加 ``-shared`` 和 ``-fPIC`` 选项。 因为由 JIT 生成的机器指令和本地的 C 语言的一样， 所以 Julia 调用这些库的开销与本地 C 语言相同。 

调用共享库和函数时使用多元组形式： ``(:function, "library")`` 或 ``("function", "library")`` ，其中 ``function`` 是 C 的导出函数名， ``library`` 是共享库名。共享库依据名字来解析，路径由环境变量来确定，有时需要直接指明。

多元组内有时仅有函数名（仅 ``:function`` 或 ``"function"`` ）。此时，函数名由当前进程解析。这种形式可以用来调用 C 库函数， Julia 运行时函数，及链接到 Julia 的应用中的函数。

默认情况下， Fortran 编译器 `会进行名字修饰
<https://en.wikipedia.org/wiki/Name_mangling#Fortran>`_
(例如, 将函数名改为小写或大写, 经常会加一个下划线)， 所以在用 :func:`ccall` 调用 Fortran 函数时必须将修饰过的函数名传递进去。 并且， 调用 Fortran 函数时，所有的输入都必须通过引用来传递。

最后， 可以使用 ``ccall`` 来生成库函数调用。 ``ccall`` 的参数如下:

1. (:function, "library") 多元组对儿（必须为常量，详见下面）。
2. 返回类型 (参见下面的表格对应 声明的 C 类型到 Julia)
  - 这个参数会在编译时被处理。
3. 输入的类型的多元组， 与上述的返回类型的要求类似。 输入必须是多元组，而不是值为多元组的变量或表达式。
  - 这个参数会在编译是被处理。
4. 后面的参数， 如果有的话，都是被调用函数的实参。

下例调用标准 C 库中的 ``clock`` ： ::

    julia> t = ccall( (:clock, "libc"), Int32, ())
    2292761

    julia> t
    2292761

    julia> typeof(ans)
    Int32

``clock`` 函数没有参数，返回 ``Int32`` 类型。输入的类型如果只有一个，常写成一元多元组，在后面跟一逗号。例如要调用 ``getenv`` 函数取得指向一个环境变量的指针，应这样调用： ::

    julia> path = ccall( (:getenv, "libc"), Ptr{Uint8}, (Ptr{Uint8},), "SHELL")
    Ptr{Uint8} @0x00007fff5fbffc45

    julia> bytestring(path)
    "/bin/bash"

注意，类型多元组的参数必须写成 ``(Cstring,)`` ，而不是 ``(Cstring)`` 。这是因为 ``(Cstring)`` 等价于 ``Cstring`` ，它并不是一个包含 ``(Cstring)`` 的一元多元组： ::

    julia> (Cstring)
    Cstring

    julia> (Cstring,)
    (Cstring,)

实际中要提供可复用代码时，通常要使用 Julia 的函数来封装 ``ccall`` ，设置参数，然后检查 C 或 Fortran 函数中可能出现的任何错误，将其作为异常传递给 Julia 的函数调用者。下例中， ``getenv`` C 库函数被封装在 `env.jl <https://github.com/JuliaLang/julia/blob/master/base/env.jl>`_ 里的 Julia 函数中： ::

    function getenv(var::AbstractString)
      val = ccall((:getenv, "libc"),
                  Cstring, (Cstring,), var)
      if val == C_NULL
        error("getenv: undefined variable: ", var)
      end
      unsafe_string(val)
    end

上例中，如果函数调用者试图读取一个不存在的环境变量，封装将抛出异常： ::

    julia> getenv("SHELL")
    "/bin/bash"

    julia> getenv("FOOBAR")
    getenv: undefined variable: FOOBAR

下例稍复杂些，显示本地机器的主机名： ::

    function gethostname()
      hostname = Array(Uint8, 128)
      ccall( (:gethostname, "libc"), Int32,
            (Ptr{Uint8}, Uint),
            hostname, length(hostname))
      return bytestring(convert(Ptr{Uint8}, hostname))
    end

此例先分配出一个字节数组，然后调用 C 库函数 ``gethostname`` 向数组中填充主机名，取得指向主机名缓冲区的指针，在默认其为空结尾 C 字符串的前提下，将其转换为 Julia 字符串。 C 库函数一般都用这种方式从函数调用者那儿，将申请的内存传递给被调用者，然后填充。在 Julia 中分配内存，通常都需要通过构建非初始化数组，然后将指向数据的指针传递给 C 函数。

创建与 C 兼容的 Julia 函数指针
---------------------

可以把 Julia 函数传递给本地的接受函数指针为输入的 C 函数。 比如， ::

    typedef returntype (*functiontype)(argumenttype,...)
   
函数 :func:`cfunction` 会生成对应的与 C 兼容的函数指针来调用 Julia 的库函数。

:func:`cfunction` 的参数形式如下 :

1. 一个 Julia 函数
2. 返回类型
3. 输入的类型的多元组

一个经典的例子就是 C 标准库的 ``qsort``， 声明为::

    void qsort(void *base, size_t nmemb, size_t size,
               int(*compare)(const void *a, const void *b));
   
``base`` 是长度为 ``nmemb`` 数组指针， 所有的元素都是 ``size`` 字节， ``compare`` 是一个回调函数， 其输入为两个指针， 返回为一个小于/大于零的整数。 假设我们有一个 Julia 里的一维数组 ``A``， 希望调用 ``qsort`` 进行排序(而不是用 Julia 内部的 ``sort`` 函数)。 我们得先写一个关于任意类型 T 的比较函数:: 

    function mycompare{T}(a::T, b::T)
        return convert(Cint, a < b ? -1 : a > b ? +1 : 0)::Cint
    end
    
注意， 我们必须仔细处理返回类型: ``qsort`` 需要函数返回一个 C ``int``， 所以我们需要用 ``convert`` 和 ``typeassert`` 保证返回的是一个 ``Cint``。

为了将这个函数传递到 C， 我们还需用 ``cfunction`` 得到它的地址::

    const mycompare_c = cfunction(mycompare, Cint, (Ref{Cdouble}, Ref{Cdouble}))
    
:func:`cfunction` 要接受三个参数: Julia 函数 (``mycompare``),
返回类型 (``Cint``), and 参数类型的多元组, 来对元素为 ``Cdouble`` (``Float64``) 的数组进行排序。

最终调用 ``qsort`` 如下::

    A = [1.3, -2.7, 4.4, 3.1]
    ccall(:qsort, Void, (Ptr{Cdouble}, Csize_t, Csize_t, Ptr{Void}),
          A, length(A), sizeof(eltype(A)), mycompare_c)
          
运行后， ``A`` 变成有序的数组 ``[-2.7, 1.3, 3.1, 4.4]``。 注意， Julia 可以转换数组到 ``Ptr{Cdouble}``，
可以计算一种类型的大小 (和 C 里的 ``sizeof`` 一致)。 有意思的是， 将 ``println("mycompare($a,$b)")`` 写到 
``mycompare`` 里可以打印出 ``qsort`` 进行的所有比较。

把 C 类型映射到 Julia
--------------------

在 Julia 里， 正确地匹配 C 类型很重要， 不一致的类型会导致在一个系统下正确的代码在另一个系统下发生错误。

注意， 调用 C 函数时， C 头文件是不需要的: 你需要保证 Julia 类型和函数签名和头文件里的一致。

自动转换
~~~~~~~~~

Julia 自动调用 ``convert`` 函数，将参数转换为指定类型。例如： ::

    ccall( (:foo, "libfoo"), Void, (Int32, Float64), x, y)

会按如下操作： ::

    ccall((:foo, "libfoo"), Void, (Int32, Float64),
          Base.unsafe_convert(Int32, Base.cconvert(Int32, x)),
          Base.unsafe_convert(Float64, Base.cconvert(Float64, y)))
         
:func:`cconvert` 会正常调用 :func:`convert`, 但也可以被定义返回更合适的类型来传递给 C. 例如， 
将一个数组 (比如字符串数组) 转换为指针数组。

:func:`unsafe_convert` 处理转换为 ``Ptr`` 类型的情况. 它被称作不安全是因为将对象转换为指针后对垃圾回收不可见， 会导致过早地被释放。

类型的对应
~~~~~~~~
先看一看 Julia 的相关类型术语:

.. rst-class:: text-wrap

==============================  ==============================  ======================================================
Syntax / Keyword                Example                         Description
==============================  ==============================  ======================================================
``type``                        ``String``                      "Leaf Type" :: A group of related data that includes
                                                                a type-tag, is managed by the Julia GC, and
                                                                is defined by object-identity.
                                                                The type parameters of a leaf type must be fully defined
                                                                (no ``TypeVars`` are allowed)
                                                                in order for the instance to be constructed.

``abstract``                    ``Any``,                        "Super Type" :: A super-type (not a leaf-type)
                                ``AbstractArray{T,N}``,         that cannot be instantiated, but can be used to
                                ``Complex{T}``                  describe a group of types.

``{T}``                         ``Vector{Int}``                 "Type Parameter" :: A specialization of a type
                                                                (typically used for dispatch or storage optimization).

                                                                "TypeVar" :: The ``T`` in the type parameter declaration
                                                                is referred to as a TypeVar (short for type variable).

``bitstype``                    ``Int``,                        "Bits Type" :: A type with no fields, but a size. It
                                ``Float64``                     is stored and defined by-value.

``immutable``                   ``Pair{Int,Int}``               "Immutable" :: A type with all fields defined to be
                                                                constant. It is defined by-value. And may be stored
                                                                with a type-tag.

                                ``Complex128`` (``isbits``)     "Is-Bits" :: A ``bitstype``, or an ``immutable`` type
                                                                where all fields are other ``isbits`` types. It is
                                                                defined by-value, and is stored without a type-tag.

``type ...; end``               ``nothing``                     "Singleton" :: a Leaf Type or Immutable with no fields.

``(...)`` or ``tuple(...)```    ``(1,2,3)``                     "Tuple" :: an immutable data-structure similar to an
                                                                anonymous immutable type, or a constant array.
                                                                Represented as either an array or a struct.

``typealias``                   Not applicable here             Type aliases, and other similar mechanisms of
                                                                doing type indirection, are resolved to their base
                                                                type (this includes assigning a type to another name,
                                                                or getting the type out of a function call).
==============================  ==============================  ======================================================

位类型:
~~~~~~~~~~~

特殊的类型:

``Float32``
    对应 C 的 ``float`` (或 Fortran 的 ``REAL*4``).

``Float64``
    对应 C 的 ``double`` (或 Fortran 的 ``REAL*8``).

``Complex64``
    对应 C 的 ``complex float`` (或 Fortran 的 ``COMPLEX*8``).

``Complex128``
    对应 C 的 ``complex double`` (或 Fortran 的 ``COMPLEX*16``).

``Signed``
    对应 C 的 ``signed`` (或任何 Fortran 的 ``INTEGER`` 类型). Julia 里不是 ``Signed`` 的子类都被认为是无符号的.

``Ref{T}``
    行为类似 ``Ptr{T}`` 但拥有自己的内存.

``Array{T,N}``
    当数组被当作 ``Ptr{T}`` 传到 C 里的时候, 
    并不是强制转换: Julia 需要元素类型是 ``T``, 然后地一个元素的地址将被传入。

    所以当一个 ``Array`` 包含错误格式的数据, 就必须用 ``trunc(Int32,a)`` 显式地转换。

    如果要把数组 ``A`` 当作另一个类型的指针来传递而不进行预先转换, 可以声明为 ``Ptr{Void}`` 类型。

    如果元素类型为 ``Ptr{T}`` 的数组被当作 ``Ptr{Ptr{T}}`` 传递,
    :func:`Base.cconvert` 会首先拷贝该数组为一个空值结尾的数组， 其元素为原数组元素经 :func:`cconvert` 处理后的结果. 这将允许把 ``argv`` 指针数组类型 ``Vector{String}`` 传递为 ``Ptr{Ptr{Cchar}}`` 类型.

基础的 C/C++ 类型和 Julia 类型对照如下。每个 C 类型也有一个对应名称的 Julia 类型，不过冠以了前缀 C 。这有助于编写简便的代码（但 C 中的 int 与 Julia 中的 Int 不同）。

**与系统无关：**

+-----------------------------------+-----------------+----------------------+-----------------------------------+
| C name                            | Fortran name    | Standard Julia Alias | Julia Base Type                   |
+===================================+=================+======================+===================================+
| ``unsigned char``                 | ``CHARACTER``   | ``Cuchar``           | ``UInt8``                         |
|                                   |                 |                      |                                   |
| ``bool`` (C++)                    |                 |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``short``                         | ``INTEGER*2``   | ``Cshort``           | ``Int16``                         |
|                                   |                 |                      |                                   |
|                                   | ``LOGICAL*2``   |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``unsigned short``                |                 | ``Cushort``          | ``UInt16``                        |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``int``                           | ``INTEGER*4``   | ``Cint``             | ``Int32``                         |
|                                   |                 |                      |                                   |
| ``BOOL`` (C, typical)             | ``LOGICAL*4``   |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``unsigned int``                  |                 | ``Cuint``            | ``UInt32``                        |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``long long``                     | ``INTEGER*8``   | ``Clonglong``        | ``Int64``                         |
|                                   |                 |                      |                                   |
|                                   | ``LOGICAL*8``   |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``unsigned long long``            |                 | ``Culonglong``       | ``UInt64``                        |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``intmax_t``                      |                 | ``Cintmax_t``        | ``Int64``                         |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``uintmax_t``                     |                 | ``Cuintmax_t``       | ``UInt64``                        |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``float``                         | ``REAL*4i``     | ``Cfloat``           | ``Float32``                       |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``double``                        | ``REAL*8``      | ``Cdouble``          | ``Float64``                       |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``complex float``                 | ``COMPLEX*8``   | ``Complex64``        | ``Complex{Float32}``              |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``complex double``                | ``COMPLEX*16``  | ``Complex128``       | ``Complex{Float64}``              |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``ptrdiff_t``                     |                 | ``Cptrdiff_t``       | ``Int``                           |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``ssize_t``                       |                 | ``Cssize_t``         | ``Int``                           |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``size_t``                        |                 | ``Csize_t``          | ``UInt``                          |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``void``                          |                 |                      | ``Void``                          |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``void`` and                      |                 |                      | ``Union{}``                       |
| ``[[noreturn]]`` or ``_Noreturn`` |                 |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``void*``                         |                 |                      | ``Ptr{Void}``                     |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``T*`` (where T represents an     |                 |                      | ``Ref{T}``                        |
| appropriately defined type)       |                 |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``char*``                         | ``CHARACTER*N`` |                      | ``Cstring`` if NUL-terminated, or |
| (or ``char[]``, e.g. a string)    |                 |                      | ``Ptr{UInt8}`` if not             |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``char**`` (or ``*char[]``)       |                 |                      | ``Ptr{Ptr{UInt8}}``               |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``jl_value_t*``                   |                 |                      | ``Any``                           |
| (any Julia Type)                  |                 |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``jl_value_t**``                  |                 |                      | ``Ref{Any}``                      |
| (a reference to a Julia Type)     |                 |                      |                                   |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``va_arg``                        |                 |                      | Not supported                     |
+-----------------------------------+-----------------+----------------------+-----------------------------------+
| ``...``                           |                 |                      | ``T...`` (where ``T``             |
| (variadic function specification) |                 |                      | is one of the above types,        |
|                                   |                 |                      | variadic functions of different   |
|                                   |                 |                      | argument types are not supported) |
+-----------------------------------+-----------------+----------------------+-----------------------------------+

``Cstring`` 和 ``Ptr{UInt8}`` 等价 , 除了当 Julia 字符串包含任何嵌入的 NUL 时， 转换成 ``Cstring`` 时会抛出错误。  当你传递 ``char*`` 到 C 程式且字符串不以 NUL 结尾， 或者确定 Julia 字符串不包含 NUL 且希望跳过检查, 就可以把 ``Ptr{UInt8}`` 作为参数类型。
``Cstring`` 可以作为 :func:`ccall` 的返回类型, 但这样做显然不会引入额外的检查， 只是增加可读性。

**与系统有关：**

======================  ==============  =======
``char``                ``Cchar``       ``Int8`` (x86, x86_64)

                                        ``Uint8`` (powerpc, arm)
``long``                ``Clong``       ``Int`` (UNIX)

                                        ``Int32`` (Windows)
``unsigned long``       ``Culong``      ``Uint`` (UNIX)

                                        ``Uint32`` (Windows)
``wchar_t``             ``Cwchar_t``    ``Int32`` (UNIX)

                                        ``Uint16`` (Windows)
======================  ==============  =======

.. note::

    调用 Fortran 函数时, 所有输入必须按引用传递, 所以所有类型都要加上 ``Ptr{..}`` 或者
    ``Ref{..}``。

.. warning::

    字符串参数 (``char*``) 在 Julia 里类型必须是 ``Cstring`` (如果是 NUL 结尾的话) 或是 ``Ptr{Cchar}`` 与 ``Ptr{UInt8}`` 之一，
    而不是 ``String``。 类似地, 数组参数 (``T[]`` 或 ``T*``), 其 Julia 类型必须是 ``Ptr{T}``, 而不是 ``Vector{T}``。

.. warning::

    Julia 里 ``Char`` 类型是 32 位, 和宽字符类型在所有平台上不同 (``wchar_t`` or ``wint_t``) 。

.. warning::

    返回类型为 ``Union{}`` 意味着函数将不会返回，
    比如， C++11 ``[[noreturn]]`` 或 C11 ``_Noreturn`` (例如 ``jl_throw`` 或
    ``longjmp``). 不要在返回空值的函数上使用。

.. note::

    对于 ``wchar_t*`` 参数, Julia 类型为 ``Cwstring`` (如果是 NUL 结尾的话) 或 ``Ptr{Cwchar_t}``。 注意有些 UTF-8 字符串在 Julia 里是 NUL 结尾的, 所以可以直接传递给需要 NUL 结尾字符串为参数的 C 函数里(但用 ``Cwstring`` 类型会抛出错误)。

.. note::

对应于字符串参数（ ``char*`` ）的 Julia 类型为 ``Ptr{Uint8}`` ，而不是 ``ASCIIString`` 。参数中有 ``char**`` 类型的 C 函数，在 Julia 中调用时应使用 ``Ptr{Ptr{Uint8}}`` 类型。例如，C 函数： ::

    int main(int argc, char **argv);

在 Julia 中应该这样调用： ::

    argv = [ "a.out", "arg1", "arg2" ]
    ccall(:main, Int32, (Int32, Ptr{Ptr{Uint8}}), length(argv), argv)

.. note::

    声明返回 ``Void`` 的 C 函数在 Julia 里会返回 ``nothing`` 。

结构体类型的对应
~~~~~~~~~~~~~~~~~~~~~~~~~~~

复合类型， C 里的 ``struct`` 或 Fortran90 的 ``TYPE`` 
(或 F77的一些变种里的 ``STRUCTURE`` / ``RECORD``),
可以对应到 Julia 的 ``type`` 或 ``immutable``.

When used recursively, ``isbits`` types are stored inline.
All other types are stored as a pointer to the data.
When mirroring a struct used by-value inside another struct in C,
it is imperative that you do not attempt to manually copy the fields over,
as this will not preserve the correct field alignment.
Instead, declare an immutable ``isbits`` type and use that instead.
Unnamed structs are not possible in the translation to Julia.

Packed structs and union declarations are not supported by Julia.

You can get a near approximation of a ``union`` if you know, a priori,
the field that will have the greatest size (potentially including padding).
When translating your fields to Julia, declare the Julia field to be only
of that type.

Arrays of parameters must be expanded manually, currently
(either inline, or in an immutable helper type). For example::

    in C:
    struct B {
        int A[3];
    };
    b_a_2 = B.A[2];

    in Julia:
    immutable B_A
        A_1::Cint
        A_2::Cint
        A_3::Cint
    end
    type B
        A::B_A
    end
    b_a_2 = B.A.(2)

Arrays of unknown size are not supported.

In the future, some of these restrictions may be reduced or eliminated.


SIMD 类型
~~~~~~~~~~~

注意: 仅支持 64-bit x86 和 AArch64 平台.

如果 C/C++ 程式输入或输出包含本地的 SIMD 类型, 其对应的 Julia 类型为
元素为 ``VecElement`` 的多元组。 特别地:

    - 多元组和该 SIMD 类型具有相同大小.
      比如, 代表 ``__m128`` 的多元组在 x86 上必须有 16 字节。

    - 多元组的元素类型必须是 ``VecElement{T}`` 的实例，
      这里 ``T`` 是位类型， 大小可以是 1, 2, 4 or 8 字节.

举个例子, 下面的 C 程式用到了 AVX 内部函数::

    #include <immintrin.h>

    __m256 dist( __m256 a, __m256 b ) {
        return _mm256_sqrt_ps(_mm256_add_ps(_mm256_mul_ps(a, a),
                                            _mm256_mul_ps(b, b)));
    }

这一段 Julia 代码通过 ``ccall`` 调用了 ``dist`` ::

    typealias m256 NTuple{8,VecElement{Float32}}

    a = m256(ntuple(i->VecElement(sin(Float32(i))),8))
    b = m256(ntuple(i->VecElement(cos(Float32(i))),8))

    function call_dist(a::m256, b::m256)
        ccall((:dist, "libdist"), m256, (m256, m256), a, b)
    end

    println(call_dist(a,b))

主机需要必要的 SIMD 寄存器。 上面的代码仅对支持 AVX 的主机有效。

内存所有权
~~~~~~~~~~~~~~~~

**malloc/free**

内存分配与释放必须由合适的方法来处理. 在 Julia 里， 不要试图用 ``Libc.free`` 来释放由 C 分配的内存, 这会导致错误的 ``free`` 函数被调用而且会使 Julia 崩溃. 反过来也一样.

何时改用 T, Ptr{T} 和 Ref{T}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 Julia 里调用 C 程式, 在 :func:`ccall` 里普通的参数 (非指针)
应声明为 ``T`` , 因为它们按值传递。  对于接受指针的, 应当用 ``Ref{T}``, 内存可以被 Julia 或 C 通过隐式调用 :func:`cconvert` 来管理。
相反, 由 C 函数返回的指针应为 ``Ptr{T}``, 表示它们的内存由 C 管理。  C 结构体里的指针应为对应的 Julia
不可变类型里的 ``Ptr{T}`` 。

在 Julia 调用 Fortran 程式, 所有输入都应是 ``Ref{T}``, 因为 Fortran 按引用传递。 Fortran 子程式返回类型为 ``Void`` ,
或返回 ``T`` 如果 Fortran 程式返回 ``T`` 类型.

将 C 函数映射到 Julia
----------------------------

``ccall``/``cfunction`` 输入参数类型参考
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

将 C 参数列表对应到 Julia:

* ``T``, 这里 ``T`` 是:
  ``char``, ``int``, ``long``, ``short``, ``float``, ``double``, ``complex``, ``enum`` 或它们的 ``typedef`` 等价类型

  + ``T`` 为 Julia 的位类型 (如上表)
  + 若``T`` 类型为 ``enum``, 其类型等价为 ``Cint`` 或 ``Cuint``
  + 按值传递的参数

* ``struct T`` (包含 typedef 的)

  + ``T`` 为 Julia 的 leaf type
  + 按值传递的参数
 
* ``void*``

  + depends on how this parameter is used, first translate this to the intended pointer type,
    then determine the Julia equivalent using the remaining rules in this list
  + this argument may be declared as ``Ptr{Void}``, if it really is just an unknown pointer

* ``jl_value_t*``

  + ``Any``
  + argument value must be a valid Julia object
  + currently unsupported by :func:`cfunction`

* ``jl_value_t**``

  + ``Ref{Any}``
  + argument value must be a valid Julia object (or ``C_NULL``)
  + currently unsupported by :func:`cfunction`

* ``T*``

  + ``Ref{T}``, where ``T`` is the Julia type corresponding to ``T``
  + argument value will be copied if it is an ``isbits`` type
    otherwise, the value must be a valid Julia object

* ``(T*)(...)`` (e.g. a pointer to a function)

  + ``Ptr{Void}`` (you may need to use :func:`cfunction` explicitly to create this pointer)

* ``...`` (e.g. a vararg)

  + ``T...``, where ``T`` is the Julia type

* ``va_arg``

  + not supported

``ccall``/``cfunction`` 返回类型参考
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For translating a C return type to Julia:

* ``void``

  + ``Void`` (this will return the singleton instance ``nothing::Void``)

* ``T``, where ``T`` is one of the primitive types:
  ``char``, ``int``, ``long``, ``short``, ``float``, ``double``, ``complex``, ``enum``
  or any of their ``typedef`` equivalents

  + ``T``, where ``T`` is an equivalent Julia Bits Type (per the table above)
  + if ``T`` is an ``enum``, the argument type should be equivalent to ``Cint`` or ``Cuint``
  + argument value will be copied (returned by-value)

* ``struct T`` (including typedef to a struct)

  + ``T``, where ``T`` is a Julia Leaf Type
  + argument value will be copied (returned by-value)

* ``void*``

  + depends on how this parameter is used, first translate this to the intended pointer type,
    then determine the Julia equivalent using the remaining rules in this list
  + this argument may be declared as ``Ptr{Void}``, if it really is just an unknown pointer

* ``jl_value_t*``

  + ``Any``
  + argument value must be a valid Julia object

* ``jl_value_t**``

  + ``Ref{Any}``
  + argument value must be a valid Julia object (or ``C_NULL``)

* ``T*``

  + If the memory is already owned by Julia, or is an ``isbits`` type, and is known to be non-null:

    + ``Ref{T}``, where ``T`` is the Julia type corresponding to ``T``
    + a return type of ``Ref{Any}`` is invalid, it should either be ``Any``
      (corresponding to ``jl_value_t*``) or ``Ptr{Any}`` (corresponding to ``Ptr{Any}``)
    + C **MUST NOT** modify the memory returned via ``Ref{T}`` if ``T`` is an ``isbits`` type

  + If the memory is owned by C:

    + ``Ptr{T}``, where ``T`` is the Julia type corresponding to ``T``

* ``(T*)(...)`` (e.g. a pointer to a function)

  + ``Ptr{Void}`` (you may need to use :func:`cfunction` explicitly to create this pointer)

用指针传递修改值
-------------------------------------

.. Passing Pointers for Modifying Inputs
.. -------------------------------------

.. Because C doesn't support multiple return values, often C functions will take
.. pointers to data that the function will modify. To accomplish this within a
.. ``ccall`` you need to encapsulate the value inside an array of the appropriate
.. type. When you pass the array as an argument with a ``Ptr`` type, julia will
.. automatically pass a C pointer to the encapsulated data

因为 C 不支持多返回值， 所以通常 C 函数会用指针来修改值。 在 ``ccall`` 里完成这些需要把值放在适当类型的数组里。当你用 ``Ptr`` 传递整个数组时，
Julia 会自动传递一个 C 指针到被这个值::

    width = Cint[0]
    range = Cfloat[0]
    ccall(:foo, Void, (Ptr{Cint}, Ptr{Cfloat}), width, range)
    
在返回时， ``width`` 和 ``range`` 会被 ``width[]`` and ``range[]`` 接收。

Special Reference Syntax for ccall (deprecated):
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``&`` syntax is deprecated, use the ``Ref{T}`` argument type instead.

A prefix ``&`` is used on an argument to :func:`ccall` to indicate that a pointer
to a scalar argument should be passed instead of the scalar value itself
(required for all Fortran function arguments, as noted above). The following
example computes a dot product using a BLAS function.

::

    function compute_dot(DX::Vector{Float64}, DY::Vector{Float64})
      assert(length(DX) == length(DY))
      n = length(DX)
      incx = incy = 1
      product = ccall((:ddot_, "libLAPACK"),
                      Float64,
                      (Ptr{Int32}, Ptr{Float64}, Ptr{Int32}, Ptr{Float64}, Ptr{Int32}),
                      &n, DX, &incx, DY, &incy)
      return product
    end

The meaning of prefix ``&`` is not quite the same as in C. In
particular, any changes to the referenced variables will not be
visible in Julia unless the type is mutable (declared via
``type``). However, even for immutable types it will not cause any
harm for called functions to attempt such modifications (that is,
writing through the passed pointers). Moreover, ``&`` may be used with
any expression, such as ``&0`` or ``&f(x)``.

When a scalar value is passed with ``&`` as an argument of type
``Ptr{T}``, the value will first be converted to type ``T``.


Some Examples of C Wrappers
---------------------------

Here is a simple example of a C wrapper that returns a ``Ptr`` type::

    type gsl_permutation
    end

    # The corresponding C signature is
    #     gsl_permutation * gsl_permutation_alloc (size_t n);
    function permutation_alloc(n::Integer)
        output_ptr = ccall(
            (:gsl_permutation_alloc, :libgsl), #name of C function and library
            Ptr{gsl_permutation},              #output type
            (Csize_t,),                        #tuple of input types
            n                                  #name of Julia variable to pass in
        )
        if output_ptr==C_NULL # Could not allocate memory
            throw(OutOfMemoryError())
        end
        return output_ptr
    end

The `GNU Scientific Library <https://www.gnu.org/software/gsl/>`_ (here assumed
to be accessible through ``:libgsl``) defines an opaque pointer,
``gsl_permutation *``, as the return type of the C function
``gsl_permutation_alloc()``. As user code never has to look inside the
``gsl_permutation`` struct, the corresponding Julia wrapper simply needs a new
type declaration, ``gsl_permutation``, that has no internal fields and whose
sole purpose is to be placed in the type parameter of a ``Ptr`` type.  The
return type of the :func:`ccall` is declared as ``Ptr{gsl_permutation}``, since the
memory allocated and pointed to by ``output_ptr`` is controlled by C (and not
Julia).

The input ``n`` is passed by value, and so the function's input signature is
simply declared as ``(Csize_t,)`` without any ``Ref`` or ``Ptr`` necessary.
(If the wrapper was calling a Fortran function instead, the corresponding
function input signature should instead be ``(Ref{Csize_t},)``, since Fortran
variables are passed by reference.) Furthermore, ``n`` can be any type that is
convertable to a ``Csize_t`` integer; the :func:`ccall` implicitly calls
:func:`Base.cconvert(Csize_t, n) <cconvert>`.


Here is a second example wrapping the corresponding destructor::

    # The corresponding C signature is
    #     void gsl_permutation_free (gsl_permutation * p);
    function permutation_free(p::Ref{gsl_permutation})
        ccall(
            (:gsl_permutation_free, :libgsl), #name of C function and library
            Void,                             #output type
            (Ref{gsl_permutation},),          #tuple of input types
            p                                 #name of Julia variable to pass in
        )
    end

Here, the input ``p`` is declared to be of type ``Ref{gsl_permutation}``,
meaning that the memory that ``p`` points to may be managed by Julia or by C.
A pointer to memory allocated by C should be of type ``Ptr{gsl_permutation}``,
but it is convertable using :func:`cconvert` and therefore can be used in the
same (covariant) context of the input argument to a :func:`ccall`. A pointer to
memory allocated by Julia must be of type ``Ref{gsl_permutation}``, to ensure
that the memory address pointed to is valid and that Julia's garbage collector
manages the chunk of memory pointed to correctly. Therefore, the
``Ref{gsl_permutation}`` declaration allows pointers managed by C or Julia to
be used.

If the C wrapper never expects the user to pass pointers to memory managed by
Julia, then using ``p::Ptr{gsl_permutation}`` for the method signature of the
wrapper and similarly in the :func:`ccall` is also acceptable.


Here is a third example passing Julia arrays::

    # The corresponding C signature is
    #    int gsl_sf_bessel_Jn_array (int nmin, int nmax, double x,
    #                                double result_array[])
    function sf_bessel_Jn_array(nmin::Integer, nmax::Integer, x::Real)
        if nmax<nmin throw(DomainError()) end
        result_array = Array{Cdouble}(nmax-nmin+1)
        errorcode = ccall(
            (:gsl_sf_bessel_Jn_array, :libgsl), #name of C function and library
            Cint,                               #output type
            (Cint, Cint, Cdouble, Ref{Cdouble}),#tuple of input types
            nmin, nmax, x, result_array         #names of Julia variables to pass in
        )
        if errorcode!= 0 error("GSL error code $errorcode") end
        return result_array
    end

The C function wrapped returns an integer error code; the results of the actual
evaluation of the Bessel J function populate the Julia array ``result_array``.
This variable can only be used with corresponding input type declaration
``Ref{Cdouble}``, since its memory is allocated and managed by
Julia, not C. The implicit call to :func:`Base.cconvert(Ref{Cdouble},
result_array) <cconvert>` unpacks the Julia pointer to a Julia array data
structure into a form understandable by C.

Note that for this code to work correctly, ``result_array`` must be declared to
be of type ``Ref{Cdouble}`` and not ``Ptr{Cdouble}``. The memory is managed by
Julia and the ``Ref`` signature alerts Julia's garbage collector to keep
managing the memory for ``result_array`` while the :func:`ccall` executes. If
``Ptr{Cdouble}`` were used instead, the :func:`ccall` may still work, but
Julia's garbage collector would not be aware that the memory declared for
``result_array`` is being used by the external C function. As a result, the
code may produce a memory leak if ``result_array`` never gets freed by the
garbage collector, or if the garbage collector prematurely frees
``result_array``, the C function may end up throwing an invalid memory access
exception.

垃圾回收机制的安全
------------------

给 ccall 传递数据时，最好避免使用 ``pointer()`` 函数。应当定义一个转换方法，将变量直接传递给 ccall 。ccall 会自动安排，使得在调用返回前，它的所有参数都不会被垃圾回收机制处理。如果 C API 要存储一个由 Julia 分配好的内存的引用，当 ccall 返回后，需要自己设置，使对象对垃圾回收机制保持可见。推荐的方法为，在一个类型为 ``Array{Any,1}`` 的全局变量中保存这些值，直到 C 接口通知它已经处理完了。

只要构造了指向 Julia 数据的指针，就必须保证原始数据直至指针使用完之前一直存在。Julia 中的许多方法，如 ``unsafe_ref()`` 和 ``bytestring()`` ，都复制数据而不是控制缓冲区，因此可以安全释放（或修改）原始数据，不会影响到 Julia 。有一个例外需要注意，由于性能的原因， ``pointer_to_array()`` 会共享（或控制）底层缓冲区。

垃圾回收并不能保证回收的顺序。例如，当 ``a`` 包含对 ``b`` 的引用，且两者都要被垃圾回收时，不能保证 ``b`` 在 ``a`` 之后被回收。这需要用其它方式来处理。

非常量函数说明
--------------

``(name, library)`` 函数说明应为常量表达式。可以通过 ``eval`` ，将计算结果作为函数名： ::

    @eval ccall(($(string("a","b")),"lib"), ...

表达式用 ``string`` 构造名字，然后将名字代入 ``ccall`` 表达式进行计算。注意 ``eval`` 仅在顶层运行，因此在表达式之内，不能使用本地变量（除非本地变量的值使用 ``$`` 进行过内插）。 ``eval`` 通常用来作为顶层定义，例如，将包含多个相似函数的库封装在一起。

间接调用
--------

``ccall`` 的第一个参数可以是运行时求值的表达式。此时，表达式的值应为 ``Ptr`` 类型，指向要调用的原生函数的地址。这个特性用于 ``ccall``
的第一参数包含对非常量（本地变量或函数参数）的引用时。

调用方式
--------

``ccall`` 的第二个（可选）参数指定调用方式（在返回值之前）。如果没指定，将会使用操作系统的默认 C 调用方式。其它支持的调用方式为: ``stdcall`` , ``cdecl`` , ``fastcall`` 和 ``thiscall`` 。例如 (来自 base/libc.jl)： ::

    hn = Array(Uint8, 256)
    err=ccall(:gethostname, stdcall, Int32, (Ptr{Uint8}, Uint32), hn, length(hn))

更多信息请参考 `LLVM Language Reference`_.

.. _LLVM Language Reference: http://llvm.org/docs/LangRef.html#calling-conventions

Accessing Global Variables
--------------------------

Global variables exported by native libraries can be accessed by name using the
:func:`cglobal` function. The arguments to :func:`cglobal` are a symbol specification
identical to that used by :func:`ccall`, and a type describing the value stored in
the variable::

    julia> cglobal((:errno,:libc), Int32)
    Ptr{Int32} @0x00007f418d0816b8

The result is a pointer giving the address of the value. The value can be
manipulated through this pointer using :func:`unsafe_load` and :func:`unsafe_store!`.


Accessing Data through a Pointer
--------------------------------
The following methods are described as "unsafe" because a bad pointer
or type declaration can cause Julia to terminate abruptly.

Given a ``Ptr{T}``, the contents of type ``T`` can generally be copied from
the referenced memory into a Julia object using ``unsafe_load(ptr, [index])``.
The index argument is optional (default is 1),
and follows the Julia-convention of 1-based indexing.
This function is intentionally similar to the behavior of :func:`getindex` and :func:`setindex!`
(e.g. ``[]`` access syntax).

The return value will be a new object initialized
to contain a copy of the contents of the referenced memory.
The referenced memory can safely be freed or released.

If ``T`` is ``Any``, then the memory is assumed to contain a reference to
a Julia object (a ``jl_value_t*``), the result will be a reference to this object,
and the object will not be copied. You must be careful in this case to ensure
that the object was always visible to the garbage collector (pointers do not
count, but the new reference does) to ensure the memory is not prematurely freed.
Note that if the object was not originally allocated by Julia, the new object
will never be finalized by Julia's garbage collector.  If the ``Ptr`` itself
is actually a ``jl_value_t*``, it can be converted back to a Julia object
reference by :func:`unsafe_pointer_to_objref(ptr) <unsafe_pointer_to_objref>`.
(Julia values ``v`` can be converted to ``jl_value_t*`` pointers, as
``Ptr{Void}``, by calling :func:`pointer_from_objref(v)
<pointer_from_objref>`.)

The reverse operation (writing data to a ``Ptr{T}``), can be performed using
:func:`unsafe_store!(ptr, value, [index]) <unsafe_store!>`.  Currently, this is only supported
for bitstypes or other pointer-free (``isbits``) immutable types.

Any operation that throws an error is probably currently unimplemented
and should be posted as a bug so that it can be resolved.

If the pointer of interest is a plain-data array (bitstype or immutable), the
function :func:`unsafe_wrap(Array, ptr,dims,[own]) <unsafe_wrap>` may be
more useful. The final parameter should be true if Julia should "take
ownership" of the underlying buffer and call ``free(ptr)`` when the returned
``Array`` object is finalized.  If the ``own`` parameter is omitted or false,
the caller must ensure the buffer remains in existence until all access is
complete.

Arithmetic on the ``Ptr`` type in Julia (e.g. using ``+``) does not behave the
same as C's pointer arithmetic. Adding an integer to a ``Ptr`` in Julia always
moves the pointer by some number of *bytes*, not elements. This way, the
address values obtained from pointer arithmetic do not depend on the
element types of pointers.


Thread-safety
-------------

Some C libraries execute their callbacks from a different thread, and
since Julia isn't thread-safe you'll need to take some extra
precautions. In particular, you'll need to set up a two-layered
system: the C callback should only *schedule* (via Julia's event loop)
the execution of your "real" callback.
To do this, create a ``AsyncCondition`` object and wait on it::

  cond = Base.AsyncCondition()
  wait(cond)

The callback you pass to C should only execute a :func:`ccall` to
``:uv_async_send``, passing ``cb.handle`` as the argument,
taking care to avoid any allocations or other interactions with the Julia runtime.

Note that events may be coalesced, so multiple calls to uv_async_send
may result in a single wakeup notification to the condition.

More About Callbacks
--------------------

For more details on how to pass callbacks to C libraries, see this
`blog post <http://julialang.org/blog/2013/05/callback>`_.

C++
---

Limited support for C++ is provided by the `Cpp <https://github.com/timholy/Cpp.jl>`_,
`Clang <https://github.com/ihnorton/Clang.jl>`_, and `Cxx <https://github.com/Keno/Cxx.jl>`_ packages.
