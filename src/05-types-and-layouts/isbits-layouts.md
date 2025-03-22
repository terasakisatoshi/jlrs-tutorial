# `isbits` レイアウト

Juliaのプリミティブ型のレイアウトは通常、Rustのアナログと一致します。`DataType` が `Int8` の場合、この型の `Value` は `i8` へのポインタです。`Float32` の場合、`Value` は `f32` へのポインタです。例外は2つあります。`Bool` と `Char` は、`bool` や `char` ではなく、jlrsで定義された同名の型にマップされます。

複合型はもう少し複雑です。`isbits` 型はプリミティブ型や他の `isbits` 型で構成された不変型です。これらのレイアウトは、Rustにおける明白な `repr(C)` 表現にマップされます。例えば、以下のRustとJuliaの型はメモリ内で同じレイアウトを持ちます。

```julia
struct InnerBits
    a::Int8
end

struct OuterBits
    inner::InnerBits
    b::UInt8
end
```

```rust,ignore
#[repr(C)]
struct InnerBits {
    a: i8
}

#[repr(C)]
struct OuterBits {
    inner: InnerBits,
    b: u8,
}
```

Rustの`InnerBits`と`OuterBits`は、Juliaの対応する型のレイアウトを忠実に表現しているため、`ValidLayout`を実装できます。フィールド型として使用されるときに複合型にインライン化されるため、`ValidField`を実装できます。これらのレイアウトはJuliaの単一の型に対応しているため、これらの型はRustの型をJuliaの対応する型にマッピングするために`ConstructType`を実装できます。
