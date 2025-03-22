# キャスト、アンボクシング、管理データへのアクセス

これまでのところ、呼び出した唯一のJulia関数は`println`であり、これは特に面白くありません。なぜなら、`nothing`を返すからです。実際には、Julia関数を副作用のためだけでなく、その結果をRustで使用したいことがよくあります。

`Value`は、GCによって管理されるあるJulia型のインスタンスです。そのJulia型に対してより具体的な管理型がある場合、`Value::cast`を使って`Value`をキャストすることで変換できます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 1>(|mut frame| {
        let s = JuliaString::new(&mut frame, "Hello, World!").as_value();
        assert!(s.cast::<JuliaString>().is_ok());

        let module = Module::main(&frame).as_value();
        assert!(module.cast::<Module>().is_ok());
        assert!(module.cast::<JuliaString>().is_err());
    });
}
```

管理型だけがRustとJuliaの間でマッピングされる型ではありません。Rustのレイアウトが管理データのレイアウトと一致する多くの型があり、ほとんどのプリミティブ型がこれに含まれます。これらの型は`Unbox`トレイトを実装しており、`Value::unbox`を使って`Value`からデータを抽出できます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 1>(|mut frame| {
        let one = Value::new(&mut frame, 1usize);
        let unboxed = one.unbox::<usize>().expect("cannot be unboxed as usize");
        assert_eq!(unboxed, 1);
    });
}
```

`Unbox`または`Managed`を実装する適切な型がない場合、`Value`のフィールドに手動でアクセスできます。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 4>(|mut frame| {
        // Normally, this custom type would have been defined in some module.
        // Safety: Defining a new type is safe.
        let custom_type = unsafe {
            Value::eval_string(
                &mut frame,
                "struct CustomType
                    a::UInt8
                    b::Bool
                    CustomType() = new(0x1, false)
                end

                CustomType",
            )
            .expect("cannot create CustomType")
        };

        // Safety: the constructor of CustomType is safe to call
        let inst = unsafe {
            custom_type
                .call0(&mut frame)
                .expect("cannot call constructor of CustomType")
        };

        let a = inst.get_field(&mut frame, "a")
            .expect("no field named a")
            .unbox::<u8>()
            .expect("cannot unbox as u8");

        assert_eq!(a, 1);

        let b = inst.get_field(&mut frame, "b")
            .expect("no field named b")
            .unbox::<Bool>()
            .expect("cannot unbox as Bool")
            .as_bool();

        assert_eq!(b, false);
    });
}
```

この例では多くのことが行われていますが、その多くはセットアップコードに過ぎません。まず、`CustomType`を定義するJuliaコードを評価します。Juliaのコンストラクタは型にリンクされた関数に過ぎないので、評価したコードの結果を呼び出すことで`CustomType`のコンストラクタを呼び出すことができます。最後に、`Value::get_field`を使用してフィールドにアクセスし、その内容をアンボックスします[^1]。2番目のフィールドは`bool`ではなく`Bool`としてアンボックスされます。同様に、Juliaの`Char`型はjlrsの`Char`型にマップされます。これらの型は、RustとJuliaの間での潜在的な不一致を避けるために存在します。

[^1]: `Value::get_field`は名前でフィールドにアクセスします。タプルのフィールドにアクセスしたい場合は、`Value::get_nth_field`を使ってインデックスでアクセスする必要があります。インデックスは0から始まります。
