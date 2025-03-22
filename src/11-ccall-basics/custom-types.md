# カスタムタイプ

Rustのデータを、どのJuliaタイプにも対応しないタイプで公開したい場合はどうすればよいでしょうか？

最も簡単な方法は、データをボックス化し、ポインタとしてリークし、Juliaで`Ptr{Cvoid}`として扱うことです。`Ptr{Cvoid}`は`isbits`タイプなので、Rustからポインタを単純に返し、Juliaで`Ptr{Cvoid}`を期待することができます。使用が終わったら、リークしたデータを解放するためのカスタム関数が必要になります。

明確にするために、このチュートリアルの例では、ライブラリの検索パスにライブラリがあると仮定し、その内容に名前でアクセスできるようにしています。

```rust,ignore
#[no_mangle]
pub unsafe extern "C" fn new_rs_string(ptr: *const u8, len: usize) -> *mut String {
    let slice = std::slice::from_raw_parts(ptr, len);
    let s = String::from_utf8_lossy(slice).into_owned();
    let boxed = Box::new(s);
    Box::leak(boxed) as *mut _
}

#[no_mangle]
pub unsafe extern "C" fn print_rs_string(s: *mut String) {
    let s = s.as_ref().unwrap();
    println!("{s}")
}

#[no_mangle]
pub unsafe extern "C" fn free_rs_string(s: *mut String) {
    let _ = Box::from_raw(s);
}
```

```julia
julia> s = "Foo"
"Foo"

julia> rs_s = ccall((:new_rs_string, "libjulia_lib"), Ptr{Cvoid}, (Ptr{UInt8}, UInt), s, sizeof(s))
Ptr{Nothing} @0x000000000224ea40

julia> ccall((:print_rs_string, "libjulia_lib"), Cvoid, (Ptr{Cvoid},), rs_s)
Foo

julia> ccall((:free_rs_string, "libjulia_lib"), Cvoid, (Ptr{Cvoid},), rs_s)

```

データをボックス化できない場合は、Juliaで不変タイプを定義してこのタイプに適応する必要があります。

```rust,ignore
#[repr(C)]
#[derive(Copy, Clone)]
pub struct Compound {
    a: f64,
    b: f64
}

#[no_mangle]
pub unsafe extern "C" fn new_compound(a: f64, b: f64) -> Compound {
    Compound { a, b }
}

#[no_mangle]
pub unsafe extern "C" fn add_compound(compound: Compound) -> f64 {
    compound.a + compound.b
}
```

```julia
julia> struct Compound
           a::Float64
           b::Float64
       end

julia> ccall((:new_compound, "libjulia_lib"), Compound, (Float64, Float64), 1.0, 2.0)
Compound(1.0, 2.0)

julia> ccall((:add_compound, "libjulia_lib"), Float64, (Compound,), compound)
3.0
```
