# パラシュート

データを削除しなければならない状況を避けられない場合、例外が発生しても安全に削除されるように、このデータにパラシュートを取り付けることが可能かもしれません。データにパラシュートを取り付けると、それを管理されたデータに変換してRustからJuliaに移動し、GCが削除を担当するようにします。

パラシュートは `AttachParachute::attach_parachute` を呼び出すことで取り付けることができます。このトレイトは `Sized + Send + Sync + 'static` である任意の型に実装されています。結果として得られる `WithParachute` は元の型を参照解除し、`WithParachute::remove_parachute` を呼び出すことでパラシュートを取り外すことができます。

```rust,ignore
use jlrs::{catch::catch_exceptions, data::managed::parachute::AttachParachute, prelude::*};

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 2>(|mut frame| {
        // Safety: this is a POF. We attach a parachute to vec
        // to make the GC responsible for dropping it.
        unsafe {
            catch_exceptions(
                || {
                    let dims = (usize::MAX, usize::MAX);
                    let vec = vec![1usize];
                    let mut with_parachute = vec.attach_parachute(&mut frame);
                    let arr = TypedArray::<u8>::new_unchecked(&mut frame, dims);
                    with_parachute.push(2);
                    arr
                },
                |e| println!("caught exception: {e:?}"),
            )
        }
        .expect_err("allocated ridiculously-sized array successfully");
    });
}
```

`vec` にパラシュートを取り付けたので、次の行で例外が発生しても問題ありません。GCが最終的にそれを削除してくれます。
