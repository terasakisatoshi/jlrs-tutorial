# ガベージコレクション、ロック、およびその他のブロッキング関数

以前、管理データとルート化について議論した際、GCは管理データを割り当てることでトリガーされる可能性があると言われました。あるスレッドによってGCがトリガーされると、そのスレッドは、Juliaに呼び出しを行う可能性のある他のすべてのスレッドがセーフポイントに到達するまでブロックされます。セーフポイントは通常、管理データを割り当てることで到達されますが、最近のバージョンのJuliaでは、Julia関数が呼び出されたときにも到達します。セーフポイントに到達すると、そのスレッドは、まだ使用する必要のあるすべての管理データがルート化されているか、少なくともルートから到達可能であることを約束します。

これは1つのスレッドのみが使用される場合には問題になりませんが、複数のスレッドがJuliaに呼び出しを行うことができる場合には容易に問題になる可能性があります。問題はロックで最も顕著です。次のような状況を考えてみましょう。スレッドAとBがJuliaに呼び出しを行うことができ、これらのスレッドがロックを取得し、Juliaデータを割り当てます。もし1つのスレッドがガベージコレクタをトリガーし、他のスレッドがロックを待っている間にデッドロック状態に陥ります。他のスレッドは、決して解放されないロックを待っているため、セーフポイントに到達することはありません。

この特定の問題を解決するために、jlrsは複数のGCセーフロックタイプを提供します。GCセーフとは、明示的なセーフポイントに到達しなくてもGCを実行するのが安全であることを意味します。上記の問題でGCセーフロックを使用すると、ロックを待っている間にGCが実行できるため、デッドロックが解消されます。以下のGCセーフロックが提供されています：

- `GcSafeMutex`
- `GcSafeFairMutex`
- `GcSafeRwLock`
- `GcSafeOnceLock`

これらのGCセーフな代替品は、parking_lotやonce_cellに見られる同様の名前のタイプから適応されたもので、唯一の違いは、ブロッキング操作がGCセーフブロックで呼び出されることです。

Juliaに呼び出しを行わない任意の長時間実行コードを呼び出す場合にも同様の問題が発生します：それはセーフポイントに到達せず、GCが実行される必要がある場合、この操作が完了するまで待つ必要があります。この操作がJuliaに呼び出しを行う必要がないため、GCセーフブロックで実行するのが安全です。`gc_safe`関数を使用してこれを行うことができますが、GCセーフブロック内でJuliaとどのようにしても相互作用するのは不健全です。

```rust,ignore
use std::{thread, time::Duration};

use jlrs::{memory::gc::gc_safe, prelude::*, runtime::builder::Builder};

fn long_running_op() {
    thread::sleep(Duration::from_secs(5));
}

fn main() {
    let (mut mt_handle, thread_handle) = Builder::new().spawn_mt().expect("cannot init Julia");
    let mut mt_handle2 = mt_handle.clone();

    let t1 = thread::spawn(move || {
        mt_handle.with(|handle| {
            handle.local_scope::<_, 1>(|mut frame| {
                // Safety: long_running_op doesn't interact with Julia
                unsafe { gc_safe(long_running_op) };

                // Safety: we're just printing a string
                unsafe { Value::eval_string(&mut frame, "println(\"Hello from thread 1\")") }
                    .expect("caught exception");
            })
        })
    });

    let t2 = thread::spawn(move || {
        mt_handle2.with(|handle| {
            handle.local_scope::<_, 1>(|mut frame| {
                // Safety: we're just printing a string
                unsafe { Value::eval_string(&mut frame, "println(\"Hello from thread 2\")") }
                    .expect("caught exception");
            })
        })
    });

    t1.join().expect("thread 1 panicked");
    t2.join().expect("thread 2 panicked");
    thread_handle.join().expect("runtime thread panicked")
}
```
