# ccallの基本

これまで、RustアプリケーションにJuliaを組み込んでRustからJuliaを呼び出すことに焦点を当ててきましたが、この章では逆の側面、つまりJuliaからRustを呼び出すことを始めます。Juliaの素晴らしい機能の一つに`ccall`インターフェースがあります。これはC ABIを使用する関数、つまりRustの`extern "C"`関数を呼び出すことができます[^1]。この章のほとんどではjlrsを使用しません。Juliaは実行時に任意の動的ライブラリをロードできます。

この章の目的は、`ccall`に関する基本的な情報をカバーすることです。より詳細な情報については、Juliaマニュアルの[Calling C and Fortran Code]章を参照してください。

JuliaからRustを呼び出すことを始めるために、最初に動的ライブラリを作成する前に、最終的な埋め込み例を見ていきます。関数ポインタをJuliaに公開し、`ccall`でそれを呼び出します。

```rust,ignore
use std::ffi::c_void;

use jlrs::prelude::*;

unsafe extern "C" fn add(a: f64, b: f64) -> f64 {
    a + b
}

static JULIA_CODE: &str = "function call_rust(ptr::Ptr{Cvoid}, a::Float64, b::Float64)
    ccall(ptr, Float64, (Float64, Float64), a, b)
end";

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 5>(|mut frame| {
        let ptr = Value::new(&mut frame, add as *mut c_void);

        let a = Value::new(&mut frame, 1.0f64);
        let b = Value::new(&mut frame, 2.0f64);

        // Safety: we're just defining a function.
        let func = unsafe { Value::eval_string(&mut frame, JULIA_CODE) }
            .expect("an exception occurred");

        // Safety: Immutable types are passed and returned by value, so `add`
        // has the correct signature for the `ccall` in `call_rust`. All
        // `add` does is add `a` and `b`, which is perfectly safe.
        let res = unsafe { func.call3(&mut frame, ptr, a, b) }
            .expect("an exception occurred")
            .unbox::<f64>()
            .expect("not an f64");

        assert_eq!(res, 3.0f64);
    });
}
```

この例が行うことは、2つの数を加算して結果を返す`add`を呼び出すだけです。この関数をまずvoidポインタに変換することで`Value`に変換できます。戻り値と引数の型が静的に知られている必要があるため、Rustから直接`ccall`を呼び出すことはできません。そのため、定義を評価することで与えられた引数で関数ポインタを`ccall`する関数を作成します。

[^1]: ABIは、関数呼び出しがバイナリレベルでどのように機能するかなどを定義します。

[Calling C and Fortran Code]: https://docs.julialang.org/en/v1/manual/calling-c-and-fortran-code/
