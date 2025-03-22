# いつルート化を避けるべきか

このチュートリアルでは、主に作成した管理データが関連付けられたスコープを離れない限り有効であることを保証するために、ルート化ターゲットを使用してきました。しかし、多くの場合、管理データをルート化する必要はなく、問題なく非ルート化ターゲットを使用できます。

一部のデータはグローバルにルート化されています。特に重要なのはモジュールで定義された定数です。このような定数データに `Module::get_global` でアクセスする場合、ルート化をスキップしても安全で、使用したい場合は `Ref` 型を管理型に変換できます。データがグローバルであっても定数でない場合、Rustから使用する間は値が変わらないことを保証できれば、ルート化せずに使用しても安全です。

`nothing` のようなゼロサイズ型のインスタンスを返す関数の場合、結果をルート化する必要はありません。ゼロサイズ型のインスタンスは1つだけで、グローバルにルート化されています。シンボルやブール値、8ビット整数のインスタンスもグローバルにルート化されています。

最後に、データを使用し終わるまでGCが実行されないことを保証できる場合、データをルート化せずに残しても安全です。新しい管理データが割り当てられると、GCはいつでもトリガーされる可能性があります[^1]。GCが実行する必要があると判断した場合、各スレッドはセーフポイントに達したときに中断されます。すべてのスレッドが中断されたときにGCが実行されます。データにアクセスする間にJuliaを呼び出さなければ、セーフポイントに達しないため、ルート化せずに残すことができます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 1>(|mut frame| {
        let func = Module::base(&frame)
            .global(&frame, "println")
            .expect("cannot find println in Base");

        // Safety: println is globally rooted
        let func = unsafe { func.as_value() };

        let a = Value::new(&mut frame, 1.0);

        // Safety: We're just calling println with a Float64 argument
        let res = unsafe { func.call1(&frame, a).expect("caught exception") };

        // Safety: println returns nothing, which is globally rooted
        let res = unsafe { res.as_value() };
        assert!(res.is::<Nothing>());
    });
}
```

[^1]: GCは `Gc::gc_collect` を使用して手動でトリガーすることもでき、すべてのターゲットがこのトレイトを実装しています。
