# Juliaデータのキャッシュ

モジュール内のデータにアクセスすることは高コストになることがあります。特に頻繁にアクセスする必要がある場合はなおさらです。これらのアクセスは、`StaticRef`を使用してキャッシュすることができます。`StaticRef`は、`define_static_ref!`マクロで定義し、`static_ref!`マクロでアクセスできます。

```rust,ignore
use jlrs::{define_static_ref, prelude::*, static_ref};

define_static_ref!(ADD_FUNCTION, Value, "Base.+");

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 3>(|mut frame| {
        let v1 = Value::new(&mut frame, 1.0f64);
        let v2 = Value::new(&mut frame, 2.0f64);

        let add_func = static_ref!(ADD_FUNCTION, &frame);
        let res = unsafe { add_func.call2(&mut frame, v1, v2) }
            .expect("caught an exception")
            .unbox::<f64>()
            .expect("wrong type");

        assert_eq!(res, 3.0);
    })
}
```

これら2つの操作を`inline_static_ref!`で組み合わせることも可能です。これは、データを単一の関数でのみ使用する必要がある場合や、アクセスするための別の関数を公開したい場合に便利です。

```rust,ignore
use jlrs::{inline_static_ref, prelude::*};

#[inline]
fn add_function<'target, Tgt>(target: &Tgt) -> Value<'target, 'static>
where
    Tgt: Target<'target>,
{
    inline_static_ref!(ADD_FUNCTION, Value, "Base.+", target)
}

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 3>(|mut frame| {
        let v1 = Value::new(&mut frame, 1.0f64);
        let v2 = Value::new(&mut frame, 2.0f64);

        let add_func = add_function(&frame);
        let res = unsafe { add_func.call2(&mut frame, v1, v2) }
            .expect("caught an exception")
            .unbox::<f64>()
            .expect("wrong type");

        assert_eq!(res, 3.0);
    })
}
```

`StaticRef`はスレッドセーフです。内部的には単なるアトミックポインタで、初めてアクセスされたときに初期化されます。Juliaに呼び出せるスレッドはどれでもアクセス可能で、複数のスレッドが初期化される前にこのデータにアクセスしようとした場合、すべてのスレッドが初期化を試みます。データはグローバルにルートされているので、自分でルートする必要はありません。[^1]

[^1]: 重要なこととして、変更が以前のグローバルデータを到達不能にする可能性があることを覚えておく必要があります。`StaticRef`として公開されているグローバルを変更しないことを保証するのは私たちの責任です。
