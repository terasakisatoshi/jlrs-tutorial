# 管理されたデータと関数

前のセクションでは、`println("Hello, World!")` を評価して Julia から `"Hello, World!"` を出力しました。この方法で Julia を使用することで多くのことが達成できますが、柔軟性に欠け、多くの制限があります。これらの制限の一つは、文字列フォーマットや他の厄介な回避策を除けば、`println` が呼び出される引数を変更できないことです。

私たちが本当にやりたいのは、任意の引数で Julia の関数を呼び出すことです。まずは `println(1)` から始めましょう。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 3>(|mut frame| {
        let one = Value::new(&mut frame, 1usize);
        let println_fn = Module::base(&frame)
            .global(&mut frame, "println")
            .expect("println not found in Base");

        // Safety: calling println with an integer is safe
        unsafe { println_fn.call1(&mut frame, one).expect("println threw an exception") };
    });
}
```

フレームの容量は `3` に設定されています。これは、`&mut frame` が管理されたデータをルートするために3回使用されるためです。

フレームの最初の使用は `Value::new` の呼び出しで発生します。これは Rust から Julia へのデータ変換を行います。Julia ではこれをボクシングと呼びますが、Rust のボクシングと混同しないように、ここでは「値の作成」または「管理されたデータへの変換」と呼びます。`IntoJulia` を実装している任意の型は、この関数で管理されたデータに変換できます。jlrs はプリミティブ型、ポインタ型、および32フィールド以下のタプルに対するこのトレイトの実装を提供しています。

ほとんどの関数はモジュール内のグローバルです。`println` は `Base` モジュールで定義されています。Julia のモジュールは `Module` 型を介してアクセスできます。これは `Value` と同様に管理された型です。`Module::base` と `Module::main` 関数は、それぞれ `Base` と `Main` モジュールへのアクセスを提供します。これらの関数は、スコープ外で存在しないようにするためにフレームへの不変参照を取りますが、ルートする必要はなく、これはフレームの使用としてカウントされません。Julia モジュール内のグローバルは `Module::global` でアクセスできます。このメソッドを呼び出してその結果をルートする際に、フレームを2回目に使用します。[1]

最後に、フレームと1つの引数で `println_fn` を呼び出します。これがフレームの3回目で最後の使用です。任意の `Value` は呼び出し可能である可能性があり、`Call` トレイトは任意の数の引数でそれらを呼び出すメソッドを提供します。`Call::call1` のような特殊化されたメソッドは、3つ以下の引数で関数を呼び出すために存在し、`Call::call` は任意の数の引数を受け入れます。すべての引数は `Value` でなければなりません。

Julia関数を呼び出すことが危険である理由は、主にJuliaコードを評価することが危険である理由と同じです。何も防ぐものがないため、`unsafe_load`を不正なポインタで呼び出すことが可能です。他のリスクとしては、スレッドセーフティや、Rustから直接アクセスされるデータを可変にエイリアスすることが挙げられますが、これらは静的に防ぐことができません。実際には、ほとんどのJuliaコードは、Rustから呼び出すのと同じくらい安全です。

注意すべき点として、関数を呼び出すことはJuliaコードを評価するよりも効率的ですが、各引数は`Value`として渡されます。これは、関数呼び出しごとに適切なメソッドへの動的ディスパッチが行われることを意味し、小さな関数を呼び出す場合には大きなオーバーヘッドを引き起こす可能性があります。実際には、できるだけ多くの処理をJuliaで行い、Rustから呼び出すために必要なコードをできるだけシンプルに保つのが最善です。これにより、Juliaが最適化する機会を得られ、jlrsが公開する低レベルインターフェースの冗長性を避けることができます。

とはいえ、私たちは`1`を出力したかったのではなく、`Hello, World!`を出力したかったのです。最も明白な方法で、上記のコードの`1usize`を`"Hello, World!"`に置き換えようとすると、`&str`が`IntoJulia`を実装していないため、コンパイルに失敗することがわかります。別の管理型である`JuliaString`を使用する必要があります。これはJuliaの`String`型に対応しています。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 3>(|mut frame| {
        let s = JuliaString::new(&mut frame, "Hello, World!").as_value();
        let println_fn = Module::base(&frame)
            .global(&mut frame, "println")
            .expect("println not found in Base");

        // Safety: calling println with a string is safe
        unsafe { println_fn.call1(&mut frame, s).expect("println threw an exception") };
    });
}
```

これまでに、`Value`、`Module`、`JuliaString`という3つの管理型に出会いましたが、今後さらに多くの管理型を目にすることになるでしょう。すべての管理型は`Managed`トレイトを実装しており、そのスコープをエンコードする少なくとも1つのライフタイムを持っています。`Managed::as_value`メソッドを使用して、管理データを`Value`に変換することができます。

[^1]: ここでフレームを再度使用する必要はありませんでしたが、それはこの章の範囲外です。
