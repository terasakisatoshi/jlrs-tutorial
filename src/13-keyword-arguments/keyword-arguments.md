# キーワード引数

カスタムキーワード引数を使用して関数を呼び出すには、いくつかの小さなステップが必要です：

1. `named_tuple!`を使用してカスタム引数を持つ`NamedTuple`を作成します。
2. `ProvideKeyword::provide_keywords`を使用して、呼び出したい関数にその引数を提供します。
3. 結果として得られる`WithKeywords`インスタンスを位置引数で呼び出します。`WithKeywords`は`Call`を実装しています。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 8>(|mut frame| {
        unsafe {
            let func = Value::eval_string(&mut frame, "add(a, b; c=3.0, d=4.0, e=5.0) = a + b + c + d + e")
                .expect("an exception occurred");

            let a = Value::new(&mut frame, 1.0);
            let b = Value::new(&mut frame, 2.0);
            let c = Value::new(&mut frame, 5.0);
            let d = Value::new(&mut frame, 1.0);
            let kwargs = named_tuple!(&mut frame, "c" => c, "d" => d);

            let res = func.call2(&mut frame, a, b)
                .expect("caught exception")
                .unbox::<f64>()
                .expect("not an f64");

            assert_eq!(res, 15.0);

            let func_with_kwargs = func
                .provide_keywords(kwargs)
                .expect("invalid keyword arguments");

            let res = func_with_kwargs.call2(&mut frame, a, b)
                .expect("caught exception")
                .unbox::<f64>()
                .expect("not an f64");

            assert_eq!(res, 14.0);
        };
    });
}
```
