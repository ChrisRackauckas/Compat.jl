# Compat Package for Julia

[![Build Status](https://travis-ci.org/JuliaLang/Compat.jl.svg?branch=master)](https://travis-ci.org/JuliaLang/Compat.jl)
[![Build status](https://ci.appveyor.com/api/projects/status/github/JuliaLang/Compat.jl?branch=master)](https://ci.appveyor.com/project/quinnj/compat-jl/branch/master)

[![Compat](http://pkg.julialang.org/badges/Compat_0.3.svg)](http://pkg.julialang.org/?pkg=Compat&ver=0.3)
[![Compat](http://pkg.julialang.org/badges/Compat_0.4.svg)](http://pkg.julialang.org/?pkg=Compat&ver=0.4)
[![Compat](http://pkg.julialang.org/badges/Compat_0.5.svg)](http://pkg.julialang.org/?pkg=Compat&ver=0.5)
[![Compat](http://pkg.julialang.org/badges/Compat_0.6.svg)](http://pkg.julialang.org/?pkg=Compat&ver=0.6)

The **Compat** package is designed to ease interoperability between
older and newer versions of the [Julia
language](http://julialang.org/).  In particular, in cases where it is
impossible to write code that works with both the latest Julia
`master` branch and older Julia versions, or impossible to write code
that doesn't generate a deprecation warning in some Julia version, the
Compat package provides a macro that lets you use the *latest syntax*
in a backwards-compatible way.

This is primarily intended for use by other [Julia
packages](http://docs.julialang.org/en/latest/manual/packages/), where
it is important to maintain cross-version compatibility.

## Usage

To use Compat in your Julia package, add a line `Compat` to the
`REQUIRE` file in your package directory.  Then, in your package,
shortly after the `module` statement include lines like these:

```julia
using Compat
import Compat.String
```

and then as needed add

```julia
@compat ...compat syntax...
```

wherever you want to use syntax that differs in the latest Julia
`master` (the development version of Julia). The `compat syntax` is usually
the syntax on Julia `master`. However, in a few cases where this is not possible,
a slightly different syntax might be used.
Please check the list below for the specific syntax you need.

## Supported syntax

Currently, the `@compat` macro supports the following syntaxes:

* `@compat foo.:bar` — `foo.(:bar)` in 0.4 ([#15032])

* `@compat f.(args...)` — `broadcast(f, args...)` in 0.4 ([#15032])

* `@compat (a::B{T}){T}(c) = d` — the Julia 0.5-style call overload

* `@compat(get(io, s, false))`, with `s` equal to `:limit`, `:compact` or `:multiline`, to detect the corresponding print settings (performs useful work only on Julia 0.5, defaults to `false` otherwise)

* `@compat import Base.show` and `@compat function show(args...)` for handling the deprecation of `writemime` in Julia 0.5 ([#16563]). See https://github.com/JuliaLang/Compat.jl/pull/219.

* `@compat @boundscheck checkbounds(...)` rewrites to unconditionally call `checkbounds(...)` in 0.4.  The 0.4-style two-argument form of `@boundscheck` is left unchanged.

* `@compat Nullable(value, hasvalue)` to handle the switch from the `Nullable` `:isnull` field to `:hasvalue` field ([#18510])

* `@compat x .= y` converts to an in-place assignment to `x` (via `broadcast!`) ([#17510]).
  However, beware that `.=` in Julia 0.4 has the precedence of `==`, not of assignment `=`, so if the right-hand-side `y`
  includes expressions with lower precedence than `==` you should enclose it in parentheses `x .= (y)` to ensure the
  correct order of evaluation.   Also, `x .+= y` converts to `x .= (x .+ y)`, and similarly for the other updating
  assignment operators (`.*=` and so on).

* `@compat Array{<:Real}`, `@compat Array{>:Int}`, and similar uses of `<:T` (resp. `>:T`) to define a set of "covariant" (resp. "contravariant") parameterized types ([#20414]).
  In 0.4 and 0.5, this only works for non-nested usages (e.g. you can't define `Array{<:Array{<:Real}}`).

* `@compat abstract type T end` and `@compat primitive type T 8 end`
  to declare abstract and primitive types. [#20418]
  This only works when `@compat` is applied directly on the declaration.

* `@compat A{T} = B{T}` or `@compat const A{T} = B{T}` to declare type alias with free
  parameters. [#20500]. Use `const A = B` for type alias without free parameters.

* `@compat Base.IndexStyle(::Type{<:MyArray}) = IndexLinear()` and `@compat Base.IndexStyle(::Type{<:MyArray}) = IndexCartesian()` to define traits for abstract arrays, replacing the former `Base.linearindexing{T<:MyArray}(::Type{T}) = Base.LinearFast()` and `Base.linearindexing{T<:MyArray}(::Type{T}) = Base.LinearSlow()`, respectively.

* `Compat.collect(A)` returns an `Array`, no matter what indices the array `A` has (Julia 0.5 and higher). [#21257]

## Module Aliases

* In 0.6, some 0.5 iterator functions have been moved to the `Base.Iterators`
  module. Code can be written to work on both 0.5 and 0.6 by `import`ing or
  `using` the `Compat.Iterators` module instead. ([#18839])

* The `Compat.Iterators` module is also available on 0.4, including the
  iterator functions `partition`, `product`, and `flatten`, which were
  introduced in Julia 0.5. However, because of a variety of other changes to
  the iterator system between 0.4 and 0.5, these functions behave slightly
  differently. For example, the `Iterators.product` function on 0.4 does not
  return objects with shapes. ([#14596], [#14805], [#15409])

## Type Aliases

* In 0.5, `ASCIIString` and `ByteString` were deprecated, and `UTF8String` was renamed to the (now concrete) type `String`.

    Compat provides unexported `Compat.UTF8String` and `Compat.ASCIIString` type aliases which are equivalent to the same types from Base on Julia 0.4, but to `String` on Julia 0.5. In most cases, using these types by calling `import Compat: UTF8String, ASCIIString` should be enough. Note that `Compat.ASCIIString` does **not** guarantee that the string only contains ASCII characters on Julia 0.5: call `isascii` to check if the string is pure ASCII if needed.

    Compat also provides an unexported `Compat.String` type which is equivalent to `ByteString` on Julia 0.4, and to `String` on Julia 0.5. This type should be used only in places where `ByteString` was used on Julia 0.4, i.e. where either `ASCIIString` or `UTF8String` should be accepted. It should **not** be used as the default type for variables or fields holding strings, as it introduces type-instability in Julia 0.4: use `Compat.UTF8String` or `Compat.ASCIIString` instead.

* `bytestring` has been replaced in most cases with additional `String` construction methods; for 0.4 compatibility, the usage involves replacing `bytestring(args...)` with `Compat.String(args...)`. However, for converting a `Ptr{UInt8}` to a string, use the new `unsafe_string(...)` method to make a copy or `unsafe_wrap(String, ...)` to avoid a copy.

## New functions, macros, and methods

* `@views` takes an expression and converts all slices to views ([#20164]), while
  `@view` ([#16564]) converts a single array reference to a view ([#20164]).

* `@__dot__` takes an expression and converts all assignments, function calls,
  and operators to their broadcasting "dot-call" equivalents ([#20321]).   In Julia 0.6, this
  can be abbreviated `@.`, but that macro name does not parse in earlier Julia versions.
  For this to work in older versions of Julia (prior to 0.5) that don't have dot calls,
  you should instead use `@dotcompat`, which combines the `@__dot__` and `@compat` macros.

* `foreach`, similar to `map` but when the return value is not needed ([#13744])

* `walkdir`, returns an iterator that walks the directory tree of a directory ([#13707])

* `allunique`, checks whether all elements in an iterable appear only once ([#15914])

* `Base.promote_eltype_op` is available as `Compat.promote_eltype_op`

* [`normalize`](http://docs.julialang.org/en/latest/stdlib/linalg/?highlight=normalize#Base.normalize) and [`normalize!`](http://docs.julialang.org/en/latest/stdlib/linalg/?highlight=normalize#Base.normalize!), normalizes a vector with respect to the p-norm ([#13681])

* `redirect_stdout`, `redirect_stderr`, and `redirect_stdin` take an optional function as a first argument, `redirect_std*(f, stream)`, so that one may use `do` block syntax (as first available for Julia 0.6)

* `unsafe_get` returns the `:value` field of a `Nullable` object without any null-check and has a generic fallback for non-`Nullable` argument ([#18484])

* `isnull` has a generic fallback for non-`Nullable` argument

* `transcode` converts between UTF-xx string encodings in Julia 0.5 (as a lightweight
   alternative to the LegacyStrings package) ([#17323])

* `∘` (typically used infix as `f ∘ g`) for function composition can be used in 0.5 and earlier ([#17155])

* `>:`, a supertype operator for symmetry with `issubtype` (`A >: B` is equivalent to `B <: A`), can be used in 0.5 and earlier ([#20407]).

* The method of `!` to negate functions (typically used as a unary operator, as in `!isinteger`) can be used in 0.5 and earlier ([#17155]).

* `iszero(x)` efficiently checks whether `x == zero(x)` (including arrays) can be used in 0.5 and earlier ([#19950]).

* `.&` and `.|` are short syntax for `broadcast(&, xs...)` and `broadcast(|, xs...)` (respectively) in Julia 0.6 (only supported on Julia 0.5 and above) ([#17623])

* `Compat.isapprox` with `nans` keyword argument ([#20022])

* `Compat.readline` with `chomp` keyword argument ([#20203])

* `take!` method for `Task`s since some functions now return `Channel`s instead of `Task`s ([#19841])

* The `isabstract`, `parameter_upper_bound`, `typename` reflection methods were added in Julia 0.6. This package re-exports these from the `Compat.TypeUtils` submodule. On earlier versions of julia, that module contains the same functions, but operating on the pre-0.6 type system representation.

* `broadcast` is supported on tuples of the same lengths on 0.5. ([#16986])

* `zeros` and `ones` support an interface the same as `similar` ([#19635])

* `convert` can convert between different `Set` types on 0.5 and below. ([#18727])

* `isassigned(::RefValue)` is supported on 0.5 and below. ([#18082])

* `unsafe_trunc(::Type{<:Integer}, ::Integer)` is supported on 0.5. ([#18629])

* `bswap` is supported for `Complex` arguments on 0.5 and below. ([#21346])

* `Compat.invokelatest` is equivalent to `Base.invokelatest` in Julia 0.6,
  but works in Julia 0.4+, and allows you to guarantee that a function call
  invokes the latest version of a function ([#19784]).

* `Compat.StringVector` is supported on 0.5 and below. On 0.6 and later, it aliases `Base.StringVector`. This function allocates a `Vector{UInt8}` whose data can be made into a `String` in constant time; that is, without copying. On 0.5 and later, use `String(...)` with the vector allocated by `StringVector` as an argument to create a string without copying. Note that if 0.4 support is needed, `Compat.UTF8String(...)` should be used instead. ([#19449])

## Renamed functions

* `pointer_to_array` and `pointer_to_string` have been replaced with `unsafe_wrap(Array, ...)` and `unsafe_wrap(String, ...)` respectively

* `bytestring(::Ptr, ...)` has been replaced with `unsafe_string`

* `super` is now `supertype` ([#14338])

* `qr(A, pivot=b)` is now `qr(A, Val{b})`, likewise for `qrfact` and `qrfact!`

* `readall` and `readbytes` are now `readstring` and `read` ([#14660])

* `get_bigfloat_precision` is now `precision(BigFloat)`, `set_precision` is `setprecision` and `with_bigfloat_precision` is now also `setprecision`
([#13232])

* `get_rounding` is now `rounding`. `set_rounding` and `with_rounding` are now `setrounding` ([#13232])

*  `Base.tty_size` (which was not exported) is now `displaysize` in Julia 0.5

* `Compat.LinAlg.checksquare` ([#14601])

* `issym` is now `issymmetric` ([#15192])

* `istext` is now `istextmime` ([#15708])

* `symbol` is now `Symbol` ([#16154])

* `write(::IO, ::Ptr, len)` is now `unsafe_write` ([#14766])

* `slice` is now `view` ([#16972]); do `import Compat.view` and then use `view` normally without the `@compat` macro

* `fieldoffsets` has been deprecated in favor of `fieldoffset` ([#14777])

* `print_escaped` is now another method of `escape_string`, `print_unescaped` a method of `unescape_string`, and `print_joined` a method of `join` ([#16603])

* `writemime` has been merged into `show` ([#16563]). Note that to extend this function requires `@compat`; see the [Supported Syntax](#supported-syntax) section for more information

* `$` is now `xor` or `⊻` ([#18977])

* `num` and `den` are now `numerator` and `denominator` ([#19246])

* `takebuf_array` is now a method of `take!`. `takebuf_string(io)` becomes `String(take!(io))` ([#19088])

## New macros

* `@static` has been added ([#16219])

* `@functorize` (not present in any Julia version) takes a function (or operator) and turns it into a functor object if one is available in the used Julia version. E.g. something like `mapreduce(Base.AbsFun(), Base.MulFun(), x)` can now be written as `mapreduce(@functorize(abs), @functorize(*), x)`, and `f(::Base.AbsFun())` as `f(::typeof(@functorize(abs)))`, to work across different Julia versions. `Func{1}` can be written as `supertype(typeof(@functorize(abs)))` (and so on for `Func{2}`), which will fall back to `Function` on Julia 0.5.

* `Compat.@blasfunc` makes functionality of `Base.LinAlg.BLAS.@blasfunc` available on older Julia versions

* `@__DIR__` has been added ([#18380])

* `@vectorize_1arg` and `@vectorize_2arg` are deprecated on Julia 0.6 in favor
  of the broadcast syntax ([#17302]). `Compat.@dep_vectorize_1arg` and
  `Compat.@dep_vectorize_2arg` are provided so that packages can still provide
  the deprecated definitions without causing a depwarn in the package itself
  before all the users are upgraded.

  Packages are expected to use this until all users of the deprecated
  vectorized function have migrated. These macros will be dropped when the
  support for `0.6` is dropped from `Compat`.

## Other changes

* `remotecall`, `remotecall_fetch`, `remotecall_wait`, and `remote_do` have the function to be executed remotely as the first argument in Julia 0.5. Loading `Compat` defines the same methods in older versions of Julia ([#13338])

* `Base.FS` is now `Base.Filesystem` ([#12819])
  Compat provides an unexported `Compat.Filesystem` module that is aliased to
  `Base.FS` on Julia 0.4 and `Base.Filesystem` on Julia 0.5.

* `cov` and `cor` don't allow keyword arguments anymore. Loading Compat defines compatibility methods for the new API ([#13465])

* On versions of Julia that do not contain a Base.Threads module, Compat defines a Threads module containing a no-op `@threads` macro.

* `Base.SingleAsyncWork` is now `Base.AsyncCondition`
  Compat provides an unexported `Compat.AsyncCondition` type that is aliased to
  `Base.SingleAsyncWork` on Julia 0.4 and `Base.AsyncCondition` on Julia 0.5.

* `repeat` now accepts any `AbstractArray` ([#14082]): `Compat.repeat` supports this new API on Julia 0.4, and calls `Base.repeat` on 0.5.

* `OS_NAME` is now `Sys.KERNEL`. OS information available as `is_apple`, `is_bsd`, `is_linux`, `is_unix`, and `is_windows` ([#16219])

* `cholfact`, `cholfact!`, and `chol` require that input is either `Hermitian`, `Symmetric`
or that the elements are perfectly symmetric or Hermitian on 0.5. Compat now defines methods
for `HermOrSym` such that using the new methods are backward compatible.

* `Diagonal` and `*` methods support `SubArray`s even on 0.4.

* Single-argument `min`, `max` and `minmax` are defined on 0.4.

## New types

Currently, no new exported types are introduced by Compat.

## Developer tips

One of the most important rules for `Compat.jl` is to avoid breaking user code
whenever possible, especially on a released version.

Although the syntax used in the most recent Julia version
is the preferred compat syntax, there are cases where this shouldn't be used.
Examples include when the new syntax already has a different meaning
on previous versions of Julia, or when functions are removed from `Base`
Julia and the alternative cannot be easily implemented on previous versions.
In such cases, possible solutions are forcing the new feature to be used with
qualified name in `Compat.jl` (e.g. use `Compat.<name>`) or
reimplementing the old features on a later Julia version.

If you're adding additional compatibility code to this package, the [`contrib/commit-name.sh`](https://github.com/JuliaLang/julia/blob/master/contrib/commit-name.sh) script in the base Julia repository is useful for extracting the version number from a git commit SHA. For example, from the git repository of `julia`, run something like this:

```sh
bash $ contrib/commit-name.sh a378b60fe483130d0d30206deb8ba662e93944da
0.5.0-dev+2023
```

This prints a version number corresponding to the specified commit of the form
`X.Y.Z-aaa+NNNN`, and you can then test whether Julia
is at least this version by `VERSION >= v"X.Y.Z-aaa+NNNN"`.

### Tagging the correct minimum version of Compat

One of the most frequent problems package developers encounter is finding the right
version of `Compat` to add to their REQUIRE. This is meant to be a guide on how to
specify the right lower bound.

* Find the appropriate fix needed for your package from the `Compat` README. Every
function or feature added to `Compat` is documented in its README, so you are
guaranteed to find it.

* Navigate to the [blame page of the README](https://github.com/JuliaLang/Compat.jl/blame/master/README.md)
by clicking on the README file on GitHub, and then clicking on the `blame` button
which can be found in the top-right corner.

* Now find your fix, and then find the corresponding commit ID of that fix on the
left-hand side. Click on the commit ID. This navigates to a page which recorded
that particular commit.

* On the top pane, you should find the list of the tagged versions of Compat that
includes this fix. Find the minimum version from there.

* Now specify the correct minimum version for Compat in your REQUIRE file by
`Compat <version>`

[#12819]: https://github.com/JuliaLang/julia/issues/12819
[#13232]: https://github.com/JuliaLang/julia/issues/13232
[#13338]: https://github.com/JuliaLang/julia/issues/13338
[#13465]: https://github.com/JuliaLang/julia/issues/13465
[#13681]: https://github.com/JuliaLang/julia/issues/13681
[#13707]: https://github.com/JuliaLang/julia/issues/13707
[#13744]: https://github.com/JuliaLang/julia/issues/13744
[#14082]: https://github.com/JuliaLang/julia/issues/14082
[#14338]: https://github.com/JuliaLang/julia/issues/14338
[#14596]: https://github.com/JuliaLang/julia/issues/14596
[#14601]: https://github.com/JuliaLang/julia/issues/14601
[#14660]: https://github.com/JuliaLang/julia/issues/14660
[#14766]: https://github.com/JuliaLang/julia/issues/14766
[#14777]: https://github.com/JuliaLang/julia/issues/14777
[#14805]: https://github.com/JuliaLang/julia/issues/14805
[#15032]: https://github.com/JuliaLang/julia/issues/15032
[#15192]: https://github.com/JuliaLang/julia/issues/15192
[#15409]: https://github.com/JuliaLang/julia/issues/15409
[#15708]: https://github.com/JuliaLang/julia/issues/15708
[#15914]: https://github.com/JuliaLang/julia/issues/15914
[#16154]: https://github.com/JuliaLang/julia/issues/16154
[#16219]: https://github.com/JuliaLang/julia/issues/16219
[#16563]: https://github.com/JuliaLang/julia/issues/16563
[#16564]: https://github.com/JuliaLang/julia/issues/16564
[#16603]: https://github.com/JuliaLang/julia/issues/16603
[#16972]: https://github.com/JuliaLang/julia/issues/16972
[#16986]: https://github.com/JuliaLang/julia/issues/16986
[#17155]: https://github.com/JuliaLang/julia/issues/17155
[#17302]: https://github.com/JuliaLang/julia/issues/17302
[#17323]: https://github.com/JuliaLang/julia/issues/17323
[#17510]: https://github.com/JuliaLang/julia/issues/17510
[#17623]: https://github.com/JuliaLang/julia/issues/17623
[#18082]: https://github.com/JuliaLang/julia/issues/18082
[#18380]: https://github.com/JuliaLang/julia/issues/18380
[#18484]: https://github.com/JuliaLang/julia/issues/18484
[#18510]: https://github.com/JuliaLang/julia/issues/18510
[#18629]: https://github.com/JuliaLang/julia/issues/18629
[#18727]: https://github.com/JuliaLang/julia/issues/18727
[#18839]: https://github.com/JuliaLang/julia/issues/18839
[#18977]: https://github.com/JuliaLang/julia/issues/18977
[#19088]: https://github.com/JuliaLang/julia/issues/19088
[#19246]: https://github.com/JuliaLang/julia/issues/19246
[#19449]: https://github.com/JuliaLang/julia/issues/19449
[#19635]: https://github.com/JuliaLang/julia/issues/19635
[#19784]: https://github.com/JuliaLang/julia/issues/19784
[#19841]: https://github.com/JuliaLang/julia/issues/19841
[#19950]: https://github.com/JuliaLang/julia/issues/19950
[#20022]: https://github.com/JuliaLang/julia/issues/20022
[#20164]: https://github.com/JuliaLang/julia/issues/20164
[#20203]: https://github.com/JuliaLang/julia/issues/20203
[#20321]: https://github.com/JuliaLang/julia/issues/20321
[#20407]: https://github.com/JuliaLang/julia/issues/20407
[#20414]: https://github.com/JuliaLang/julia/issues/20414
[#20418]: https://github.com/JuliaLang/julia/issues/20418
[#20500]: https://github.com/JuliaLang/julia/issues/20500
[#21257]: https://github.com/JuliaLang/julia/issues/21257
[#21346]: https://github.com/JuliaLang/julia/issues/21346
