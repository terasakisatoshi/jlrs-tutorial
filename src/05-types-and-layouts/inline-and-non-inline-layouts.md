# インラインと非インラインのレイアウト

前のセクションでは、`isbits` 型のフィールドがコンポジット型にインライン化されることを見ました。型が可変のフィールドはインライン化されません。

```julia
mutable struct Inner
    a::Int8
end

struct Outer
    inner::Inner
    b::UInt8
end
```

```rust,ignore
#[repr(C)]
struct Inner {
    a: i8
}

#[repr(C)]
struct Outer<'scope, 'data> {
    inner: Option<ValueRef<'scope, 'data>>,
    b: u8,
}
```

非インラインフィールドを表現するために、管理された型の代わりに非ルート参照が使用されます。これは、フィールドの古い値が到達不能になる可能性があるためです。管理された型と非ルート参照は、`Option<NonNull>` のニッチ最適化を利用して、`Option<ValueRef>` がポインタと同じサイズであることを保証します。`Outer` のインスタンスが管理されたデータを参照することを言います。

可変型はインライン化されないため、`Inner` は `ValidLayout` のみを実装でき、`ValidField` は実装できません。通常、イミュータブル型はインライン化されるため、`Outer` は両方のトレイトを実装できます。レイアウトは単一の型を識別するため、両方の型が `ConstructType` を実装できます。

[^1]: Julia ではヌルデータは稀であり、一般的に使用することは無効です。
