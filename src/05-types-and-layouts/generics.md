# ジェネリクス

Juliaの型にはパラメータを持たせることができ、これが型のレイアウトに影響を与える場合と与えない場合があります。

型パラメータがレイアウトに影響を与えない理由は2つあります：

1. それがフィールドで参照されていない値パラメータである。
2. インライン化されていないフィールドのレイアウトに影響を与える。

レイアウトに影響を与えない型パラメータは省略可能ですが、それ以外の場合はRustとJuliaで同様に扱うことができます：

```julia
struct Generic{T}
    t::T
end

struct SetGeneric
    t::Generic{UInt32}
end

struct Elided{T}
    t::UInt32
end
```

```rust,ignore
#[repr(C)]
struct Generic<T> {
    t: T,
}

#[repr(C)]
struct SetGeneric {
    t: Generic<u32>,
}

#[repr(C)]
struct Elided {
    t: u32,
}
```

これら3つの型はすべて`ValidLayout`と`ValidField`を実装できます。なぜなら、Juliaでは不変の型だからです。`Generic`の場合、`T: ValidField`であることが必要です。`T`が可変であるか、その他の非インライン化された型である場合、レイアウト型を`Generic<Option<ValueRef>>`として表現できます。

`Generic`と`SetGeneric`は型パラメータを省略しないため、`ConstructType`を実装できますが、`Elided`は型パラメータ`T`を認識しません。
