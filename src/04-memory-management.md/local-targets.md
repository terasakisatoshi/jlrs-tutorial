# ローカルターゲット

これまで使用してきたフレームは `LocalGcFrame` です。これはローカルと呼ばれ、すべてのルートがスタック上にローカルに保存されるため、コンパイル時にそのサイズを知る必要があります。

`LocalGcFrame` への可変参照を使用してデータをルート化するたびに、そのスロットの1つを消費します。また、`LocalOutput` や `LocalReusableSlot` としてスロットを予約することも可能で、これらは `LocalGcFrame::local_output` および `LocalGcFrame::local_reusable_slot` を呼び出すことで作成できます。これらのメソッドはスロットを消費します。両者の主な違いは、`LocalReusableSlot` が結果のライフタイムに関して少し寛容である代わりに、ルート化されていない参照を返すことです。スコープから管理されたデータの複数のインスタンスを返す必要がある場合や、1つのスロットを再利用したい場合に便利です。

```rust,ignore
use jlrs::prelude::*;

fn add<'target, Tgt>(target: Tgt, a: u8, b: u8) -> ValueResult<'target, 'static, Tgt>
where
    Tgt: Target<'target>,
{
    target.with_local_scope::<_, _, 3>(|target, mut frame| {
        let a = Value::new(&mut frame, a);
        let b = Value::new(&mut frame, b);
        let func = Module::base(&frame)
            .global(&mut frame, "+")
            .expect("+ not found in Base");

        // Safety: calling + is safe
        unsafe { func.call2(target, a, b) }
    })
}

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 2>(|mut frame| {
        let mut output = frame.local_output();
        let mut reusable_slot = frame.local_reusable_slot();

        {
            // This result can be used until the next time `(&mut) output` is used
            let result = add(&mut output, 1, 2).expect("could not add numbers");
            let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }

        {
            // This result can be used until the scope ends
            let result = add(output, 1, 2).expect("could not add numbers");
            let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }

        {
            // This result can be used until the scope ends, but must not be used after
            // `reusable_slot` has been used again. Because the result can live longer
            // than it might be rooted, it's returned as a `ValueRef`.
            let result = add(&mut reusable_slot, 1, 2).expect("could not add numbers");

            // Safety: result is rooted until we use `reusable_slot` again
            let unboxed = unsafe { result.as_value() }.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }

        {
            // This result can be used until the scope ends
            let result = add(reusable_slot, 1, 2).expect("could not add numbers");
            let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }
    });
}
```

`UnsizedLocalGcFrame` は `LocalGcFrame` に似ていますが、主な違いはそのサイズが実行時に知られている必要がないことです。フレームのサイズが静的にわかっている場合は、`LocalGcFrame` を使用してください。

```rust,ignore
use jlrs::prelude::*;

fn add<'target, Tgt>(target: Tgt, a: u8, b: u8) -> ValueResult<'target, 'static, Tgt>
where
    Tgt: Target<'target>,
{
    target.with_local_scope::<_, _, 3>(|target, mut frame| {
        let a = Value::new(&mut frame, a);
        let b = Value::new(&mut frame, b);
        let func = Module::base(&frame)
            .global(&mut frame, "+")
            .expect("+ not found in Base");

        // Safety: calling + is safe
        unsafe { func.call2(target, a, b) }
    })
}

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.unsized_local_scope(2, |mut frame| {
        let mut output = frame.local_output();
        let mut reusable_slot = frame.local_reusable_slot();

        {
            let result = add(&mut output, 1, 2).expect("could not add numbers");
            let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }

        {
            let result = add(output, 1, 2).expect("could not add numbers");
            let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }

        {
            let result = add(&mut reusable_slot, 1, 2).expect("could not add numbers");

            // Safety: result is rooted until we use `reusable_slot` again
            let unboxed = unsafe { result.as_value() }.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }

        {
            let result = add(reusable_slot, 1, 2).expect("could not add numbers");
            let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
            assert_eq!(unboxed, 3);
        }
    })
}
```
