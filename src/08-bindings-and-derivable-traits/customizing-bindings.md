# バインディングのカスタマイズ

`reflect`を使用すると、生成されたバインディングの型やフィールドの名前は、Juliaの対応するものと同じになります。Juliaの型も特定のパスで定義されていることが期待されます。これが問題であるか、望ましくない場合、これらの名前を調整することができます。

`reflect`は文字列を返すのではなく、`Layouts`という型のインスタンスを返します。これは`renamestruct!`、`renamefields!`、および`overridepath!`関数と共に使用できます。これらの関数は`Reflect`モジュールによってエクスポートされており、`renamestruct!`はRustの型の名前を変更し、`renamefields!`は生成された型のフィールドの名前を変更し、`overridepath!`はJuliaで型オブジェクトが定義されているパスを上書きします。

```julia
julia> using JlrsCore.Reflect

julia> struct MyZST end

julia> layouts = reflect([MyZST]);

julia> renamestruct!(layouts, MyZST, "MyZeroSizedType")

julia> layouts
#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, IntoJulia, ValidField, IsBits, ConstructType)]
#[jlrs(julia_type = "Main.MyZST", zero_sized_type)]
pub struct MyZeroSizedType {
}
```

```julia
julia> using JlrsCore.Reflect

julia> struct Food
           burger::Bool
       end

julia> layouts = reflect([Food]);

julia> renamefields!(layouts, Food, [:burger => "hamburger"])

julia> layouts
#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, IntoJulia, ValidField, IsBits, ConstructType, CCallArg, CCallReturn)]
#[jlrs(julia_type = "Main.Food")]
pub struct Food {
    pub hamburger: ::jlrs::data::layout::bool::Bool,
}
```

```julia
julia> using JlrsCore.Reflect

julia> struct MyZeroSizedType end

julia> layouts = reflect([MyZeroSizedType]);

julia> overridepath!(layouts, MyZeroSizedType, "Main.A.MyZeroSizedType")

julia> layouts
#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, IntoJulia, ValidField, IsBits, ConstructType)]
#[jlrs(julia_type = "Main.A.MyZeroSizedType", zero_sized_type)]
pub struct MyZeroSizedType {
}
```
