# ブロッキングタスク

ブロッキングタスクは最も単純な種類のタスクであり、`GcFrame` を受け取るクロージャです。これらはランタイムスレッドに送信され、動的スコープで実行されます。その名前が示すように、ブロッキングタスクが実行されると、タスクが完了するまでランタイムスレッドはブロックされます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let (async_handle, thread_handle) = Builder::new()
        .n_threads(4)
        .async_runtime(Tokio::<3>::new(false))
        .spawn()
        .expect("cannot init Julia");

    let dispatch = async_handle
        .blocking_task(|mut frame| {
            // Safety: we're just printing a string
            unsafe { Value::eval_string(&mut frame, "println(\"Hello from the async runtime\")") }
                .expect("caught an exception");
        });

    let recv = dispatch
        .try_dispatch()
        .expect("cannot dispatch task");

    recv.blocking_recv()
        .expect("cannot receive result");

    std::mem::drop(async_handle);
    thread_handle.join().expect("runtime thread panicked")
}
```

タスクを送信するには2段階のプロセスがあります。`AsyncHandle::blocking_task` メソッドは `Dispatch` のインスタンスを返し、タスクをディスパッチするための同期および非同期メソッドを提供します。バックチャネルがいっぱいの場合、`Dispatch::try_dispatch` は失敗しますが、再試行を可能にするために `Err` として自身を返します。

タスクが正常にディスパッチされた場合、最終的にそのタスクの結果を受け取る tokio のワンショットチャネルの受信側が返されます。
