# 配列へのアクセス

異なる配列型はデータへの直接アクセスを提供しないため、まずアクセサを作成する必要があります。アクセサには複数の種類があり、使用するべきものは要素のレイアウトに依存します。

安全性に関する注意点として、RustやJuliaのコードで既にミュータブルにアクセスされている配列には決してアクセスしないでください。

要素のレイアウトを完全に無視するには、`IndeterminateAccessor`を使用することができます。これは`ArrayBase::indeterminate_data`メソッドで作成できます。`Accessor`トレイトを実装しており、`get_value`メソッドを提供し、要素を`Value`として返します。Juliaとは異なり、配列のインデックスは0から始まります。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 2>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let f64_ty = DataType::float64_type(&frame).as_value();
        let arr = RankedArray::<2>::from_slice_copied_for(&mut frame, f64_ty, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        // Safety: we never mutably access this data.
        let accessor = unsafe { arr.indeterminate_data() };

        let a21 = accessor
            .get_value(&mut frame, [1, 0])
            .expect("out of bounds")
            .expect("undefined reference")
            .unbox::<f64>()
            .expect("wrong type");

        assert_eq!(a21, 2.);
    });
}
```

前の章で見たように、複合型のフィールドは3つの方法で格納される可能性があります：インラインで格納される、管理されたデータへの参照として格納される、またはインライン化されたユニオンとして格納される。配列要素は、複合型のフィールドとして格納されるかのように格納されますが、1つの小さな例外として、インライン化されたユニオンが複合型と配列で格納される方法に違いがあります[^1]。

これらのレイアウトそれぞれに対して別々のアクセサがあります：`InlineAccessor`、`ValueAccessor`、`BitsUnionAccessor`。さらに2つのアクセサ、`BitsAccessor`と`ManagedAccessor`があります。最初のものは`IsBits`トレイトを実装するレイアウトで使用でき、後者は`Module`や`DataType`のような任意の管理された型で使用できます。

`Typed(Ranked)Array`が使用される場合、その型の`T`パラメータから正しいアクセサが推測されるかもしれません。その場合、以下のメソッドが利用可能です：`inline_data`、`value_data`、`union_data`、`bits_data`、および`managed_data`。`try_`で始まるメソッド、例えば`try_bits_data`は、一般的にすべての配列型で利用可能です。これらのメソッドは、実行時に正しいアクセサが要求されたかどうかを確認します。`ArrayBase::has_bits_layout`のようなメソッドを使用して、アクセサが要素のレイアウトと互換性があるかどうかを確認できます。

これらすべてのアクセサ型は`Accessor`を実装しており、さらに特定のインデックスで要素にアクセスするための`get`関数を提供します。`BitsUnionAccessor`を除いて、`Index`も実装しています。これらの実装は、新しい配列を作成する関数と同じ多次元インデックスを受け入れます。インデックス可能な型が提供する`as_slice`および`into_slice`メソッドを使用すると、多次元性を無視して、データを列優先順でスライスとしてアクセスできます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    // BitsAccessor
    handle.local_scope::<_, 1>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_slice_copied(&mut frame, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        // Safety: we never mutably access this data.
        let accessor = unsafe { arr.bits_data() };

        let a11 = accessor[[0, 0]];
        assert_eq!(a11, 1.);

        let a21 = accessor[[1, 0]];
        assert_eq!(a21, 2.);

        let a12 = accessor[[0, 1]];
        assert_eq!(a12, 3.);

        let a22 = accessor[[1, 1]];
        assert_eq!(a22, 4.);
    });

    // InlineAccessor
    handle.local_scope::<_, 1>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_slice_copied(&mut frame, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        // Safety: we never mutably access this data.
        let accessor = unsafe { arr.inline_data() };

        let elem = accessor[[1, 0]];
        assert_eq!(elem, 2.);
    });

    // ValueAccessor
    handle.local_scope::<_, 2>(|mut frame| {
        // Safety: this code only allocates and returns an array
        let arr = unsafe { Value::eval_string(&mut frame, "Any[:foo, :bar]") }
            .expect("caught an exception")
            .cast::<VectorAny>()
            .expect("not a VectorAny");

        // Safety: we never mutably access this data.
        let accessor = unsafe { arr.value_data() };

        let elem = accessor.get_value(&mut frame, 0)
            .expect("out of bounds")
            .expect("undefined reference");
        let sym = Symbol::new(&frame, "foo");
        assert_eq!(elem, sym);
    });

    // ManagedAccessor
    handle.local_scope::<_, 2>(|mut frame| {
        // Safety: this code only allocates and returns an array
        let arr = unsafe { Value::eval_string(&mut frame, "Symbol[:foo, :bar]") }
            .expect("caught an exception")
            .cast::<TypedVector<Symbol>>()
            .expect("not a TypedVector<Symbol>");

        // Safety: we never mutably access this data.
        let accessor = unsafe { arr.managed_data() };

        let elem = accessor.get(&mut frame, 0).expect("undefined reference or out of bounds");
        let sym = Symbol::new(&frame, "foo");
        assert_eq!(elem, sym);
    });

    // BitsUnionAccessor
    handle.local_scope::<_, 1>(|mut frame| {
        // Safety: this code only allocates and returns an array
        let arr = unsafe { Value::eval_string(&mut frame, "Union{Int, Float64}[1.0 2; 3 4.0]") }
            .expect("caught an exception")
            .cast::<Matrix>()
            .expect("not a Matrix");

        // Safety: we never mutably access this data.
        let accessor = unsafe { arr.try_union_data().expect("wrong accessor") };

        let elem = accessor.get::<isize, _>([1, 0]).expect("wrong layout").expect("out of bounds");
        assert_eq!(elem, 3);
    });
}
```

[^1]: 複合型では、データとその型を識別するタグが隣接して保存されます。配列では、フラグはデータの後にまとめて保存されます。
