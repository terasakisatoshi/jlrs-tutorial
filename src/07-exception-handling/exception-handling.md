# 例外処理

Juliaの多くの関数は例外をスローする可能性があります。これは、jlrsによって公開されている低レベルの関数も含まれます。例としては、不正な引数でJulia関数を呼び出したり、非常に大きな配列を割り当てようとしたりすることが挙げられます。これらの関数は通常、例外をキャッチする関数としない関数の2つの形で公開されます。

例外をキャッチしない関数は常にunsafeです。Juliaの例外は`longjmp`で実装されており、例外がスローされると制御フローは最も近いキャッチブロックにジャンプします。そのため、保留中のドロップを飛び越えないことを保証する必要があります。`catch_exceptions`関数を使用してカスタム例外ハンドラを持つtryブロック内で任意の関数を呼び出すことができますが、依然としてドロップを飛び越えないことを保証する必要があるため、これはunsafeです。飛び越えられるフレームが["Plain Old Frame"]である限り、深くネストされたスコープからジャンプすることは問題ありません。

例外がスローされ、利用可能なハンドラがない場合、Juliaはプロセスを中止します。

```rust,ignore
use jlrs::{catch::catch_exceptions, prelude::*};

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    // Safety: we don't jump over any pending drops if an exception is thrown.
    handle.local_scope::<_, 1>(|mut frame| unsafe {
        catch_exceptions(
            || {
                TypedArray::<u8>::new_unchecked(&mut frame, (usize::MAX, usize::MAX));
            },
            |e| {
                println!("caught exception: {e:?}")
            },
        ).expect_err("allocated ridiculously-sized array successfully");
    });
}
```

この例は`caught exception: ArgumentError("invalid Array dimensions")`を出力するはずです。

["Plain Old Frame"]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md#plain-old-frames
