# ライブラリのテスト

動的ライブラリをテストするには、主にテストアプリケーションにJuliaを埋め込む必要がありますが、いくつか注意点があります。

- Juliaを埋め込むにはランタイム機能を有効にする必要がありますが、通常の状況ではライブラリクレートにそのような機能を有効にしてはいけません。
- 統合テストをコンパイルするには、`rlib`をビルドする必要があります。
- ライブラリがJuliaにエクスポートする関数を呼び出す前に、生成された初期化関数を呼び出す必要があります。
- エクスポートされた関数のみを呼び出すことができ、Juliaから呼び出される生成された`extern "C"`関数を呼び出すことはできません。
- テストにはパニック時のアンワインドが必要であり、通常は中止したいと考えます。

最初の2つの制限に対処するために、`Cargo.toml`を少し編集する必要があります。

```toml
[lib]
crate-type = ["cdylib", "rlib"]

[features]
rt = ["jlrs/local-rt"] # Or any other runtime feature
```

`rt`機能はデフォルトで有効にしてはいけません。コードをテストする際にのみ有効にする必要があります：`cargo test --features rt`。いつものように、Juliaはプロセスごとに一度しか初期化できないという制限が適用されます。テストディレクトリ内の各統合テストファイルには、Juliaを初期化し、生成された初期化関数を呼び出す単一のテストのみを含める必要があります。クレートも同様に、Juliaを初期化する1つのテスト関数を使用できるため、統合テストを使用するのが最も簡単です。

次のライブラリをテストしてみましょう：

```rust,ignore
use jlrs::{
    data::{
        managed::value::typed::{TypedValue, TypedValueRet},
        types::foreign_type::OpaqueType,
    },
    prelude::*,
    weak_handle,
};

#[derive(Debug)]
pub struct OpaqueInt {
    a: i32,
}

unsafe impl OpaqueType for OpaqueInt {}

impl OpaqueInt {
    pub fn new(a: i32) -> TypedValueRet<OpaqueInt> {
        match weak_handle!() {
            Ok(handle) => TypedValue::new(handle, OpaqueInt { a }).leak(),
            Err(_) => panic!("not called from Julia"),
        }
    }

    pub fn get_a(&self) -> i32 {
        self.a
    }

    pub fn set_a(&mut self, a: i32) {
        self.a = a;
    }
}

julia_module! {
    become testing_libraries_tutorial_init_fn;

    struct OpaqueInt;

    in OpaqueInt fn new(a: i32) -> TypedValueRet<OpaqueInt> as OpaqueInt;
    in OpaqueInt fn get_a(&self) -> i32;
    in OpaqueInt fn set_a(&mut self, a: i32);
}
```

ライブラリを`testing_libraries_tutorial`と呼びます。次のようにテストできます：

```rust,ignore
use jlrs::prelude::*;
use testing_libraries_tutorial::{testing_libraries_tutorial_init_fn, OpaqueInt};

fn create_opaque_int<'target, Tgt: Target<'target>>(target: &Tgt) {
    target.local_scope::<_, 1>(|mut frame| {
        let opaque_int_ref = OpaqueInt::new(0);

        // Safety: we immediately root the unrooted data.
        let opaque_int = unsafe { opaque_int_ref.root(&mut frame) };

        // Safety: this data hasn't been released to Julia yet
        let tracked = unsafe { opaque_int.track_shared() }.expect("already tracked");

        let a = tracked.get_a();
        assert_eq!(a, 0);
    });
}

fn mutate_opaque_int<'target, Tgt: Target<'target>>(target: &Tgt) {
    target.local_scope::<_, 1>(|mut frame| {
        let opaque_int_ref = OpaqueInt::new(0);

        // Safety: we immediately root the unrooted data.
        let mut opaque_int = unsafe { opaque_int_ref.root(&mut frame) };

        {
            // Safety: this data hasn't been released to Julia yet
            let mut tracked = unsafe { opaque_int.track_exclusive() }.expect("already tracked");
            tracked.set_a(1);
        }

        {
            // Safety: this data hasn't been released to Julia yet
            let tracked = unsafe { opaque_int.track_shared() }.expect("already tracked");
            let a = tracked.get_a();
            assert_eq!(a, 1);
        }
    });
}

#[test]
fn it_works() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 0>(|frame| {
        // Safety: we only call the init function once, all exported types
        // will be created in the `Main` module. The second argument must
        // be set to 1.
        unsafe { testing_libraries_tutorial_init_fn(Module::main(&frame), 1) };

        create_opaque_int(&frame);
        mutate_opaque_int(&frame);
    })
}
```

これらのテストでは、`TypedValue`を追跡して内部データへの参照を取得し、型のメソッドを呼び出すことができることがわかります。これは、`#[untracked_self]`で注釈されていない場合の生成された`extern "C"`関数の動作と一致します。この注釈が存在し、追跡を避けたい場合は、`Value`の内部ポインタに直接アクセスし、`Value::data_ptr`を使用して適切にデリファレンスすることができます。
