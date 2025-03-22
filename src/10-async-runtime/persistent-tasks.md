# 永続タスク

永続タスクを使用すると、非同期ランタイムとは独立してメッセージを送信できるタスクを設定できます。永続タスクが実行されると、その内部状態を設定し、入力データを使ってタスクを呼び出すためのハンドルを返します。内部状態は管理されたデータを参照することが許可されています。タスクはすべてのハンドルが破棄されるまで存在します。

永続タスクを作成するには、`PersistentTask` トレイトを実装する必要があります。浮動小数点数の合計を蓄積するタスクを実装してみましょう。

```rust,ignore
use jlrs::prelude::*;

struct AccumulatorTask {
    init_value: f64,
}

#[async_trait(?Send)]
impl PersistentTask for AccumulatorTask {
    type State<'state> = Value<'state, 'static>;
    type Input = f64;
    type Output = JlrsResult<f64>;

    async fn init<'frame>(
        &mut self,
        frame: AsyncGcFrame<'frame>,
    ) -> JlrsResult<Self::State<'frame>> {
        frame.with_local_scope::<_, _, 2>(|mut async_frame, mut local_frame| {
            let ref_ctor = Module::base(&local_frame).global(&mut local_frame, "Ref")?;
            let init_v = Value::new(&mut local_frame, self.init_value);

            // Safety: we're just calling the constructor of `Ref`, which is safe.
            let state = unsafe { ref_ctor.call1(&mut async_frame, init_v) }.into_jlrs_result()?;
            Ok(state)
        })
    }

    async fn run<'frame, 'state: 'frame>(
        &mut self,
        mut frame: AsyncGcFrame<'frame>,
        state: &mut Self::State<'state>,
        input: Self::Input,
    ) -> Self::Output {
        let getindex_func = Module::base(&frame).global(&mut frame, "getindex")?;
        let setindex_func = Module::base(&frame).global(&mut frame, "setindex!")?;

        // Safety: Calling getindex with state is equivalent to calling `state[]`.
        let current_sum = unsafe { getindex_func.call1(&mut frame, *state) }
            .into_jlrs_result()?
            .unbox::<f64>()?;

        let new_sum = current_sum + input;
        let new_value = Value::new(&mut frame, new_sum);

        // Safety: Calling setindex! with state and new_value is equivalent to calling
        // `state[] = new_value`.
        unsafe { setindex_func.call2(&mut frame, *state, new_value) }.into_jlrs_result()?;

        Ok(new_sum)
    }
}

fn main() {
    let (async_handle, thread_handle) = Builder::new()
        .n_threads(4)
        .async_runtime(Tokio::<3>::new(false))
        .spawn()
        .expect("cannot init Julia");

    let acc_task_handle = async_handle
        .persistent(AccumulatorTask { init_value: 1.0 })
        .try_dispatch()
        .expect("runtime has shut down")
        .blocking_recv()
        .expect("cannot receive result")
        .expect("AccumulatorTask::init failed");

    let recv = acc_task_handle
        .call(2.0)
        .try_dispatch()
        .expect("runtime has shut down");

    let res = recv
        .blocking_recv()
        .expect("cannot receive result")
        .expect("AccumulatorTask::run failed");

    assert_eq!(res, 3.0);

    std::mem::drop(acc_task_handle);
    std::mem::drop(async_handle);
    thread_handle.join().expect("runtime thread panicked")
}
```

非同期ランタイムによって永続タスクが開始されると、タスクの状態を初期化するために `init` メソッドが呼び出されます。この場合、状態は `Ref{Float64}` のインスタンスです。`Float64` は可変型ではないため、直接 `Float64` を使用することはできません。`init` 関数に提供される非同期フレームに根付いたデータは、タスクがシャットダウンするまで根付いたままです。一時データを根付かせるためにローカルスコープが使用されるので、非同期フレームに状態を根付かせるだけで済みます。

タスクが正常に初期化されると、`PersistentHandle` が返されます。このハンドルは `call` メソッドを提供し、入力データを使ってタスクの `run` メソッドを呼び出すことができます。これは非同期ランタイムにタスクをディスパッチするのと同じように機能します。

`AsyncHandle` と同様に、`PersistentHandle` はクローンしてスレッド間で共有することができます。すべてのハンドルが破棄されると、タスクはシャットダウンします。
