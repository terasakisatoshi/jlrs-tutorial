# マルチスレッドランタイム

これまでの例では、単一スレッドに制限されたローカルランタイムを使用してきました。マルチスレッドランタイムは任意のスレッドから使用でき、この機能を利用するには少なくともJulia 1.9を使用し、`multi-rt`機能を有効にする必要があります。

ローカルランタイムの代わりにマルチスレッドランタイムを使用するには、主にランタイムの開始方法を変更するだけです。

```rust,ignore
use std::thread;

use jlrs::{prelude::*, runtime::builder::Builder};

fn main() {
    let (mut mt_handle, thread_handle) = Builder::new().spawn_mt().expect("cannot init Julia");
    let mut mt_handle2 = mt_handle.clone();

    let t1 = thread::spawn(move || {
        mt_handle.with(|handle| {
            handle.local_scope::<_, 1>(|mut frame| {
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

`spawn_mt`が呼び出されると、Juliaはバックグラウンドスレッドで初期化されます。このメソッドは、Juliaを呼び出すために使用できる`MtHandle`とランタイムスレッドへのハンドルを返します。`MtHandle`はクローン化して他のスレッドに送信することができ、`MtHandle::with`を呼び出すことで、そのスレッドは一時的にスコープを作成し、Juliaを呼び出すことができる状態になります。すべての`MtHandle`がドロップされると、ランタイムスレッドはシャットダウンします。

ランタイムスレッドを生成する代わりに、現在のスレッドでJuliaを初期化し、`MtHandle`を使用できる新しいスレッドを生成することもできます。

```rust,ignore
use std::thread;

use jlrs::{
    prelude::*,
    runtime::{builder::Builder, handle::mt_handle::MtHandle},
};

fn main_inner(mut mt_handle: MtHandle) {
    let mut mt_handle2 = mt_handle.clone();

    let t1 = thread::spawn(move || {
        mt_handle.with(|handle| {
            handle.local_scope::<_, 1>(|mut frame| {
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
}

fn main() {
    Builder::new().start_mt(main_inner).expect("cannot init Julia");
}
```

これは、Qtを含むコードのように、メインアプリケーションスレッドから呼び出されることにこだわるJuliaのコードとやり取りする場合に便利です。
