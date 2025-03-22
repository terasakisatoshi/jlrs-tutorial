# 非同期タスク

非同期タスクは非同期関数を呼び出すことができ、非同期関数を待機している間にランタイムは別の非同期タスクに切り替えることができます。`CallAsync`トレイトのメソッドを使用して、任意のJulia関数を新しいJuliaタスクとして呼び出し、その完了を待機することが可能です。これを効果的に行うには、Juliaが複数のスレッドを使用するように設定されている必要があります。

非同期タスクを作成するには、`AsyncTask`トレイトを実装する必要があります。2つの数を加算する簡単なタスクを実装してみましょう。

```rust,ignore
use jlrs::prelude::*;

struct AdditionTask {
    a: f64,
    b: f64,
}

#[async_trait(?Send)]
impl AsyncTask for AdditionTask {
    type Output = f64;

    async fn run<'frame>(&mut self, mut frame: AsyncGcFrame<'frame>) -> Self::Output {
        let v1 = Value::new(&mut frame, self.a);
        let v2 = Value::new(&mut frame, self.b);
        let add_fn = Module::base(&frame)
            .global(&mut frame, "+")
            .expect("cannot find Base.+");

        // Safety: we're just adding two floating-point numbers
        unsafe { add_fn.call_async(&mut frame, [v1, v2]) }
            .await
            .expect("caught an exception")
            .unbox::<f64>()
            .expect("cannot unbox as f64")
    }
}

fn main() {
    let (async_handle, thread_handle) = Builder::new()
        .n_threads(4)
        .async_runtime(Tokio::<3>::new(false))
        .spawn()
        .expect("cannot init Julia");

    let recv = async_handle
        .task(AdditionTask { a: 1.0, b: 2.0 })
        .try_dispatch()
        .expect("runtime has shut down");

    let res = recv
        .blocking_recv()
        .expect("cannot receive result");

    assert_eq!(res, 3.0);

    std::mem::drop(async_handle);
    thread_handle.join().expect("runtime thread panicked")
}
```

トレイトの実装は`#[async_trait(?Send)]`でマークされています。これは、`AsyncTask::run`によって返される未来が他のスレッドに送信されることはできず、ランタイムスレッド上で実行されなければならないためです。このメソッドはこれまでスコープで使用してきたクロージャに非常に似ていますが、主な違いは非同期メソッドであることと、これまで使用していなかった`AsyncGcFrame`を取ることです。

`AsyncGcFrame`は、いくつかの追加機能を提供する`GcFrame`です。特に、`CallAsync`トレイトのメソッド、例えば`call_async`は任意のターゲットを取るのではなく、`AsyncGcFrame`への可変参照で呼び出されなければなりません。これらのメソッドは、関数を新しいJuliaタスクとして実行し、その完了を待機できるようにします。ランタイムスレッドは、このタスクが完了するのを待っている間に他のタスクに切り替えることができます。

非同期タスクをランタイムにディスパッチするのは、ブロッキングタスクをディスパッチするのと非常に似ています。ただし、`AsyncHandle::blocking_task`を`AsyncHandle::task`に置き換える必要があります。
