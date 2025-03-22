# 非同期ランタイム

非同期ランタイムを使用すると、Juliaをバックグラウンドスレッドで実行できます。そのハンドルを使って、このスレッドにタスクを送信できます。これを使用するには、`async-rt`機能を有効にする必要があります。一部のタスクは非同期操作をサポートしているため、非同期エグゼキュータも必要です。`tokio-rt`機能を有効にすると、tokioベースのエグゼキュータが利用可能になり、この機能は自動的に`async-rt`も有効にします。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let (async_handle, thread_handle) = Builder::new()
        .n_threads(4)
        .async_runtime(Tokio::<3>::new(false))
        .spawn()
        .expect("cannot init Julia");

    std::mem::drop(async_handle);
    thread_handle.join().expect("runtime thread panicked")
}
```

Juliaを4つのスレッドで使用するように設定しました[^1]。ビルダーは必要な設定を提供することで`AsyncBuilder`にアップグレードされます。ここでは、I/Oドライバなしでtokioベースのエグゼキュータを使用し、3つの同時タスクをサポートするようにランタイムを設定しています[^2]。このドライバを使用したい場合は、`tokio-net`機能を有効にし、`Tokio::new`の引数を`true`に変更してください。

デフォルトでは、ハンドルがランタイムスレッドと通信するために無制限のチャネルが使用されます。ランタイムを生成する前に`AsyncBuilder::channel_capacity`を呼び出すことで、制限付きチャネルを使用することもできます。

ランタイムを生成した後、ランタイムスレッドと対話するために使用できる`AsyncHandle`と、そのスレッドへの`JoinHandle`を取得します。すべての`AsyncHandle`がドロップされると、ランタイムスレッドはシャットダウンします。また、`AsyncHandle::close`を呼び出すことで手動でランタイムをシャットダウンすることも可能です。

[^1]: Julia 1.9以降、これらのスレッドはJuliaのデフォルトスレッドプールに属しており、`(Async)Builder::n_interactive_threads`でインタラクティブスレッドの数を設定することもできます。

[^2]: 非同期操作をサポートする複数のタスクを同時に実行できます。ランタイムは非同期操作の完了を待っている間に別のタスクに切り替えることができます。タスクは並行して実行されるのではなく、すべてのタスクが単一のランタイムスレッドで実行されます。
