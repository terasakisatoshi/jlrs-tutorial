# 配列の変更

前のセクションで見た不変のアクセサに加えて、jlrsは可変のアクセサも提供します。

`(try_)bits_data_mut`のようなメソッドや`BitsAccessorMut`のような型は、不変のバリアントに類似して存在します。可変アクセサによって提供されるメソッドは、不変の対応メソッドと似た名前を持っています。一部のアクセサは要素への直接的な可変アクセスを提供しません。それらは`IndexMut`を実装していないか、`set_mut`メソッドを提供していません。この場合、`AccessorMut::set_value`を使用する必要があります。

Rustから管理されたデータを変更することは、jlrsでは一般的に安全ではありません。その理由は本質的に、「可変な値にアクセスできるからといって、それを変更することが許可されているわけではない」ということに帰着します。文字列は良い例で、可変ですがそれを変更することは未定義動作（UB）です。配列アクセサはこの点で少し特別です：可変アクセサを作成することは安全ではありませんが、それらの多くの変更メソッドは安全です。安全性の要件は、そもそも可変アクセサを作成する際の要件によって暗示されています。

配列型は`Copy`を実装しているため、同じ配列に対して2つの可変アクセサや、一般的に複数の可変および不変アクセサを作成することは簡単です。これが起こらないようにするのはあなたの責任です。この問題をある程度回避するために、配列を追跡することが可能であり、この章の後半でそれについて説明します。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    // IndeterminateAccessorMut
    handle.local_scope::<_, 4>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let f64_ty = DataType::float64_type(&frame).as_value();
        let mut arr = RankedArray::<2>::from_slice_copied_for(&mut frame, f64_ty, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        // Safety: this is the only accessor to this data.
        let mut accessor = unsafe { arr.indeterminate_data_mut() };

        let v = Value::new(&mut frame, 5.0f64);
        accessor.set_value(&mut frame, [1, 0], v).expect("index out of bounds").expect("caught exception");

        let elem = accessor
            .get_value(&mut frame, [1, 0])
            .expect("out of bounds")
            .expect("undefined reference")
            .unbox::<f64>()
            .expect("wrong type");
        assert_eq!(elem, 5.);
    });

    // BitsAccessorMut
    handle.local_scope::<_, 1>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let mut arr = TypedArray::<f64>::from_slice_copied(&mut frame, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        // Safety: this is the only accessor to this data.
        let mut accessor = unsafe { arr.bits_data_mut() };

        let elem = accessor[[1, 0]];
        assert_eq!(elem, 2.);

        accessor[[1, 0]] = 4.;
        let elem = accessor[[1, 0]];
        assert_eq!(elem, 4.);
    });

    // InlineAccessorMut
    handle.local_scope::<_, 2>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let mut arr = TypedArray::<f64>::from_slice_copied(&mut frame, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        // Safety: this is the only accessor to this data.
        let mut accessor = unsafe { arr.inline_data_mut() };

        let elem = accessor[[1, 0]];
        assert_eq!(elem, 2.);

        let v = Value::new(&mut frame, 4.0f64);
        accessor
            .set_value(&mut frame, [1, 0], v)
            .expect("index out of bounds")
            .expect("caught an exception");

        let elem = accessor[[1, 0]];
        assert_eq!(elem, 4.);
    });

    // ValueAccessorMut
    handle.local_scope::<_, 3>(|mut frame| {
        // Safety: this code only allocates and returns an array
        let mut arr = unsafe { Value::eval_string(&mut frame, "Any[:foo, :bar]") }
            .expect("caught an exception")
            .cast::<VectorAny>()
            .expect("not a VectorAny");

        // Safety: this is the only accessor to this data.
        let mut accessor = unsafe { arr.value_data_mut() };

        let v = Value::new(&mut frame, 1usize);
        accessor
            .set_value(&mut frame, 0, v)
            .expect("out of bounds")
            .expect("caught an exception");

        let elem = accessor.get(&mut frame, 0).expect("out of bounds");
        assert_eq!(v, elem);
    });

    // ManagedAccessorMut
    handle.local_scope::<_, 4>(|mut frame| {
        // Safety: this code only allocates and returns an array
        let mut arr = unsafe { Value::eval_string(&mut frame, "Symbol[:foo, :bar]") }
            .expect("caught an exception")
            .cast::<TypedVector<Symbol>>()
            .expect("not a TypedVector<Symbol>");

        // Safety: this is the only accessor to this data.
        let mut accessor = unsafe { arr.managed_data_mut() };

        let sym = Symbol::new(&mut frame, "baz");
        accessor
            .set_value(&mut frame, 0, sym.as_value())
            .expect("out of bounds")
            .expect("caught an exception");

        let elem = accessor.get(&mut frame, 0).expect("out of bounds");
        assert_eq!(sym, elem);
    });

    // BitsUnionAccessorMut
    handle.local_scope::<_, 1>(|mut frame| {
        // Safety: this code only allocates and returns an array
        let mut arr =
            unsafe { Value::eval_string(&mut frame, "Union{Int, Float64}[1.0 2; 3 4.0]") }
                .expect("caught an exception")
                .cast::<Matrix>()
                .expect("not a Matrix");

        // Safety: this is the only accessor to this data.
        let mut accessor = unsafe { arr.try_union_data_mut().expect("wrong accessor") };

        accessor
            .set([1, 0], DataType::float64_type(&frame), 4.0f64)
            .expect("out of bounds or incompatible type")
            .expect("caught an exception");

        let elem = accessor
            .get::<f64, _>([1, 0])
            .expect("wrong layout")
            .expect("out of bounds");
        assert_eq!(elem, 4.0);
    });
}
```
