# マルチスレッドと非同期ランタイムの組み合わせ

マルチスレッドと非同期ランタイムの機能を組み合わせることが可能です。

`AsyncBuilder` は `start_mt` と `spawn_mt` メソッドを提供しており、`async-rt` と `multi-rt` の両方の機能が有効になっている場合に `MtHandle` と `AsyncHandle` の両方を使用できます。`AsyncHandle` は、メインランタイムスレッドから呼び出す必要があるコードがある場合に、そのスレッドにタスクを送信するのに役立ちます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let (mt_handle, async_handle, thread_handle) = Builder::new()
        .n_threads(4)
        .async_runtime(Tokio::<3>::new(false))
        .spawn_mt()
        .expect("cannot init Julia");

    std::mem::drop(async_handle);
    std::mem::drop(mt_handle);
    thread_handle.join().expect("runtime thread panicked")
}
```

各ワーカースレッドがJuliaに呼び出し、非同期ランタイムを実行できるスレッドプールを作成することもできます[^1]。`MtHandle::pool_builder` を使用して新しいプールを設定および作成できます。プールが生成されると、そのプールへの `AsyncHandle` が返されます。タスクは特定のスレッドではなくこのプールに送信され、ワーカーのいずれかによって処理されます。ワーカーがパニックで終了した場合、新しいワーカーが自動的に生成されます[^2]。

ワーカーは `AsyncHandle::try_add_worker` と `AsyncHandle::try_remove_worker` を使用して動的に追加および削除できます。すべてのワーカーが削除され、すべてのハンドルが破棄されるか、明示的に閉じられるとプールはシャットダウンします。非同期ランタイム自体にワーカーを追加することはできず、プールにのみ追加できます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let (mt_handle, thread_handle) = Builder::new()
        .n_threads(4)
        .spawn_mt()
        .expect("cannot init Julia");

    let pool_handle = mt_handle
        .pool_builder(Tokio::<3>::new(false))
        .n_workers(3.try_into().unwrap())
        .spawn();

    assert!(pool_handle.try_add_worker());
    assert!(pool_handle.try_remove_worker());

    std::mem::drop(pool_handle);
    std::mem::drop(mt_handle);
    thread_handle.join().expect("runtime thread panicked")
}
```

プールが非同期ランタイムスレッドよりも持つ追加の利点の一つは、通常レイテンシーがはるかに低いことです。特にメインスレッドでコードを実行する必要がない場合は、マルチスレッドランタイムを使用してプールを作成する方が効果的です。

[^1]: より正確には、非同期ランタイムではなく、エグゼキュータのインスタンスです。

[^2]: パニック時には中止することが推奨されますが、`ccall` で呼び出された関数内でパニックすることでJuliaコードに巻き戻さない限り、ワーカースレッドでパニックしても安全です。
