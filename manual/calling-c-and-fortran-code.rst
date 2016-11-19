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

创建 C 兼容的 Julia 函数指针
---------------------

可以把 Julia 函数传递给本地的接受函数指针为输入的 C 函数。 比如， ::

    typedef returntype (*functiontype)(argumenttype,...)
   
函数 :func:`cfunction` 会生成对应的 C 兼容的函数指针来调用 Julia 的库函数。

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

``Cstring`` 和 ``Ptr{UInt8}`` 等价 , 除了当 Julia 字符串包含任何嵌入的 NUL 时， 转换成 ``Cstring`` 时会抛出错误。  当你传递 ``char*`` 到 C 函数且字符串不以 NUL 结尾， 或者确定 Julia 字符串不包含 NUL 且希望跳过检查, 就可以把 ``Ptr{UInt8}`` 作为参数类型。
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

Note: This feature is currently implemented on 64-bit x86
and AArch64 platforms only.

If a C/C++ routine has an argument or return value that is a native
SIMD type, the corresponding Julia type is a homogeneous tuple
of ``VecElement`` that naturally maps to the SIMD type.  Specifically:

    - The tuple must be the same size as the SIMD type.
      For example, a tuple representing an ``__m128`` on x86
      must have a size of 16 bytes.

    - The element type of the tuple must be an instance of ``VecElement{T}``
      where ``T`` is a bitstype that is 1, 2, 4 or 8 bytes.

For instance, consider this C routine that uses AVX intrinsics::

    #include <immintrin.h>

    __m256 dist( __m256 a, __m256 b ) {
        return _mm256_sqrt_ps(_mm256_add_ps(_mm256_mul_ps(a, a),
                                            _mm256_mul_ps(b, b)));
    }

The following Julia code calls ``dist`` using ``ccall``::

    typealias m256 NTuple{8,VecElement{Float32}}

    a = m256(ntuple(i->VecElement(sin(Float32(i))),8))
    b = m256(ntuple(i->VecElement(cos(Float32(i))),8))

    function call_dist(a::m256, b::m256)
        ccall((:dist, "libdist"), m256, (m256, m256), a, b)
    end

    println(call_dist(a,b))

The host machine must have the requisite SIMD registers.  For example,
the code above will not work on hosts without AVX support.

通过指针读取数据
----------------

下列方法是“不安全”的，因为坏指针或类型声明可能会导致意外终止或损坏任意进程内存。

指定 ``Ptr{T}`` ，常使用 ``unsafe_ref(ptr, [index])`` 方法，将类型为 ``T`` 的内容从所引用的内存复制到 Julia 对象中。 ``index`` 参数是可选的（默认为 1 ），它是从 1 开始的索引值。此函数类似于 ``getindex()`` 和 ``setindex!()`` 的行为（如 ``[]`` 语法）。

返回值是一个被初始化的新对象，它包含被引用内存内容的浅拷贝。被引用的内存可安全释放。

如果 ``T`` 是 ``Any`` 类型，被引用的内存会被认为包含对 Julia 对象 ``jl_value_t*`` 的引用，结果为这个对象的引用，且此对象不会被拷贝。需要谨慎确保对象始终对垃圾回收机制可见（指针不重要，重要的是新的引用），来确保内存不会过早释放。注意，如果内存原本不是由 Julia 申请的，新对象将永远不会被 Julia 的垃圾回收机制释放。如果 ``Ptr`` 本身就是 ``jl_value_t*`` ，可使用 ``unsafe_pointer_to_objref(ptr)`` 将其转换回 Julia 对象引用。（可通过调用 ``pointer_from_objref(v)`` 将Julia 值 ``v`` 转换为 ``jl_value_t*`` 指针 ``Ptr{Void}``  。）

逆操作（向 Ptr{T} 写数据）可通过 ``unsafe_store!(ptr, value, [index])`` 来实现。目前，仅支持位类型和其它无指针（ ``isbits`` ）不可变类型。

现在任何抛出异常的操作，估摸着都是还没实现完呢。来写个帖子上报 bug 吧，就会有人来解决啦。

如果所关注的指针是（位类型或不可变）的目标数据数组， ``pointer_to_array(ptr,dims,[own])`` 函数就非常有用啦。如果想要 Julia “控制”底层缓冲区并在返回的 ``Array`` 被释放时调用 ``free(ptr)`` ，最后一个参数应该为真。如果省略 ``own`` 参数或它为假，则调用者需确保缓冲区一直存在，直至所有的读取都结束。

``Ptr`` 的算术(比如 ``+``) 和 C 的指针算术不同， 对 ``Ptr`` 加一个整数会将指针移动一段距离的 *字节* ， 而不是元素。这样从指针运算上得到的地址不会依赖指针类型。

.. Arithmetic on the ``Ptr`` type in Julia (e.g. using ``+``) does not behave the
.. same as C's pointer arithmetic. Adding an integer to a ``Ptr`` in Julia always
.. moves the pointer by some number of *bytes*, not elements. This way, the
.. address values obtained from pointer arithmetic do not depend on the
.. element types of pointers.

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

这被广泛用在了 Julia 的 LAPACK 接口上， 其中整数类型的 ``info`` 被以引用的方式传到 LAPACK， 再返回是否成功。

.. readproof
.. This is used extensively in Julia's LAPACK interface, where an integer ``info``
.. is passed to LAPACK by reference, and on return, includes the success code.

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
``cglobal`` function. The arguments to ``cglobal`` are a symbol specification
identical to that used by ``ccall``, and a type describing the value stored in
the variable::

    julia> cglobal((:errno,:libc), Int32)
    Ptr{Int32} @0x00007f418d0816b8

The result is a pointer giving the address of the value. The value can be
manipulated through this pointer using ``unsafe_load`` and ``unsafe_store``.

Passing Julia Callback Functions to C
-------------------------------------

It is possible to pass Julia functions to native functions that accept function
pointer arguments. A classic example is the standard C library ``qsort`` function,
declared as::

    void qsort(void *base, size_t nmemb, size_t size,
               int(*compare)(const void *a, const void *b));

The ``base`` argument is a pointer to an array of length ``nmemb``, with elements of
``size`` bytes each. ``compare`` is a callback function which takes pointers to two
elements ``a`` and ``b`` and returns an integer less/greater than zero if ``a`` should
appear before/after ``b`` (or zero if any order is permitted). Now, suppose that we
have a 1d array ``A`` of values in Julia that we want to sort using the ``qsort``
function (rather than Julia’s built-in sort function). Before we worry about calling
``qsort`` and passing arguments, we need to write a comparison function that works for
some arbitrary type T::

    function mycompare{T}(a_::Ptr{T}, b_::Ptr{T})
        a = unsafe_load(a_)
        b = unsafe_load(b_)
        return convert(Cint, a < b ? -1 : a > b ? +1 : 0)
    end

Notice that we have to be careful about the return type: ``qsort`` expects a function
returning a C ``int``, so we must be sure to return ``Cint`` via a call to ``convert``.

In order to pass this function to C, we obtain its address using the function
``cfunction``::

    const mycompare_c = cfunction(mycompare, Cint, (Ptr{Cdouble}, Ptr{Cdouble}))

``cfunction`` accepts three arguments: the Julia function (``mycompare``), the return
type (``Cint``), and a tuple of the argument types, in this case to sort an array of
``Cdouble`` (Float64) elements.

The final call to ``qsort`` looks like this::

    A = [1.3, -2.7, 4.4, 3.1]
    ccall(:qsort, Void, (Ptr{Cdouble}, Csize_t, Csize_t, Ptr{Void}),
          A, length(A), sizeof(eltype(A)), mycompare_c)

After this executes, ``A`` is changed to the sorted array ``[ -2.7, 1.3, 3.1, 4.4]``.
Note that Julia knows how to convert an array into a ``Ptr{Cdouble}``, how to compute
the size of a type in bytes (identical to C’s ``sizeof`` operator), and so on.
For fun, try inserting a ``println("mycompare($a,$b)")`` line into ``mycompare``, which
will allow you to see the comparisons that ``qsort`` is performing (and to verify that
it is really calling the Julia function that you passed to it).

Thread-safety
~~~~~~~~~~~~~

Some C libraries execute their callbacks from a different thread, and
since Julia isn't thread-safe you'll need to take some extra
precautions. In particular, you'll need to set up a two-layered
system: the C callback should only *schedule* (via Julia's event loop)
the execution of your "real" callback. Your callback
needs to be written to take two inputs (which you'll most likely just
discard) and then wrapped by ``SingleAsyncWork``::

  cb = Base.SingleAsyncWork(data -> my_real_callback(args))

The callback you pass to C should only execute a ``ccall`` to
``:uv_async_send``, passing ``cb.handle`` as the argument.

More About Callbacks
~~~~~~~~~~~~~~~~~~~~

For more details on how to pass callbacks to C libraries, see this
`blog post <http://julialang.org/blog/2013/05/callback/>`_.

C++
---

`Cpp <https://github.com/timholy/Cpp.jl>`_ 和 `Clang <https://github.com/ihnorton/Clang.jl>`_ 扩展包提供了有限的 C++ 支持。

处理不同平台
------------

当处理不同的平台库的时候，经常要针对特殊平台提供特殊函数。这时常用到变量 ``OS_NAME`` 。此外，还有一些常用的宏： ``@windows``, ``@unix``, ``@linux``, 及 ``@osx`` 。注意， linux 和 osx 是 unix 的不相交的子集。宏的用法类似于三元条件运算符。

简单的调用： ::

    ccall( (@windows? :_fopen : :fopen), ...)

复杂的调用： ::

    @linux? (
             begin
                 some_complicated_thing(a)
             end
           : begin
                 some_different_thing(a)
             end
           )

链式调用（圆括号可以省略，但为了可读性，最好加上）： ::

    @windows? :a : (@osx? :b : :c)
