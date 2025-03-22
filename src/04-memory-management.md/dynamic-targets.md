# 動的ターゲット

`GcFrame` は `LocalGcFrame` の動的サイズ版です。`GcFrame` を使用することで、必要なスロットの数を数える必要がなくなります。

まず、動的スタックを設定する必要があります。これは `WithStack::with_stack` を呼び出すことで行います。`WithStack` トレイトは `LocalHandle` に実装されています。`LocalGcFrame` と同様に、スロットを予約する二次的なターゲットとして `Output` と `ReusableSlot` があります。これらはローカルの対応物と全く同じように動作します。

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
    let mut handle = Builder::new().start_local().expect("cannot init Julia");

    handle.with_stack(|mut stack| {
        stack.scope(|mut frame| {
            let mut output = frame.output();
            let mut reusable_slot = frame.reusable_slot();

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

                // Safety: result is rooted until we use reusable_slot again
                let unboxed = unsafe { result.as_value() }.unbox::<u8>().expect("cannot unbox as u8");
                assert_eq!(unboxed, 3);
            }

            {
                // This result can be used until the scope ends
                let result = add(reusable_slot, 1, 2).expect("could not add numbers");
                let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
                assert_eq!(unboxed, 3);
            }
        })
    })
}
```

動的スコープはローカルスコープのようにネストすることができますが、これは `GcFrame::scope` を呼び出すことでのみ可能です。スタックを必要とするため、任意のターゲットが新しい動的スコープを作成することはできません[^1]。このスタックの割り当てとサイズ変更は比較的高価であり、アプリケーション全体に渡すのは複雑になる可能性があるため、ローカルスコープを使用するのが最善です。

もう一つの動的ターゲットとして `AsyncGcFrame` があります。これは追加の非同期機能を持つ `GcFrame` で、非同期ランタイムが導入される際に詳しく見ていきます。

[^1]: 技術的には弱いハンドルを作成することで可能ですが、動的スタックの設定が比較的高価であるため、これは推奨されません。
