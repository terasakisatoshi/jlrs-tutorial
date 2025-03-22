# ターゲットとネストされたスコープの使用

ターゲットを取る関数は値で受け取ります。つまり、ターゲットは一度しか使用できません[^1]。`&mut frame`を使ってそのような関数を呼び出すと、そのフレームのスロットは1つだけ使用され、結果をルートします。これにより、必要なスロットの数を数えるのができるだけ簡単になります。なぜなら、クロージャ内でフレームがターゲットとして使用される回数だけを数えればよいからです。

ここで明らかな疑問が生じます。ターゲットを取る関数が複数の値をルートする必要がある場合はどうすればよいのでしょうか？答えは、ターゲットを使ってネストされたスコープを作成することです。

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

        // Safety: calling + is safe.
        unsafe { func.call2(target, a, b) }
    })
}

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 1>(|mut frame| {
        let result = add(&mut frame, 1, 2).expect("could not add numbers");
        let unboxed = result.unbox::<u8>().expect("cannot unbox as u8");
        assert_eq!(unboxed, 3);
    });
}
```

このアプローチは、管理されたデータを必要以上に長くルートすることを避けるのに役立ちます。`add`を呼び出した後は、その結果だけがルートされます。その関数で作成した一時的な値は、スコープを離れたため、もはやルートされません。

特定のターゲット型を取る関数を書くことは強く避け、常にターゲットをジェネリックに取ることをお勧めします。

[^1]: 一部の関数はターゲットを不変参照で受け取り、ルートされたデータを返します。このデータはグローバルにルートされることが保証されており、操作はターゲットを消費しません。
