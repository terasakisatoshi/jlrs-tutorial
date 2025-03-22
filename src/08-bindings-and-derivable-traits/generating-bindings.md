# バインディングの生成

`JlrsCore.jl`の`Reflect`モジュールにある`reflect`関数を使用して、Juliaの型のバインディングを生成できます。この関数は型のベクターを引数として呼び出すことができます。

```julia
julia> using JlrsCore.Reflect

julia> struct MyStruct
           a::Int8
           b::Tuple{Int8, UInt8}
       end

julia> struct MyWrapper
           ms::MyStruct
       end

julia> reflect([MyWrapper])
#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, IntoJulia, ValidField, IsBits, ConstructType, CCallArg, CCallReturn)]
#[jlrs(julia_type = "Main.MyStruct")]
pub struct MyStruct {
    pub a: i8,
    pub b: ::jlrs::data::layout::tuple::Tuple2<i8, u8>,
}

#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, IntoJulia, ValidField, IsBits, ConstructType, CCallArg, CCallReturn)]
#[jlrs(julia_type = "Main.MyWrapper")]
pub struct MyWrapper {
    pub ms: MyStruct,
}
```

ご覧のとおり、`MyWrapper`だけを含めることで、`MyStruct`のバインディングも再帰的に生成されます[^1]。生成されたバインディングは可能な限りすべてのトレイトを派生し、生成元の型へのパスが注釈として付けられています。このパスが実行時に型が存在するパスと一致することが重要です。

`reflect`には2つのキーワードパラメータ、`f16`と`complex`があります。これらのいずれかを`true`に設定すると、同名の機能が有効になっている場合、`Float16`を`half::f16`に、`Complex{T}`を`num::Complex<T>`にそれぞれマッピングします。

`reflect`が処理できないものが3つあります：

1. ジェネリックパラメータを参照する`Union`型のフィールドを持つ型。
2. ジェネリックパラメータを参照する`Tuple`型のフィールドを持つ型。
3. アトミックフィールドを持つ型[^2]。

このリストには一般的な`Union`フィールドは含まれていません。すべての可能なバリアントが既知であれば、レイアウトは静的であり、有効なレイアウトを生成できます：

```julia
julia> using JlrsCore.Reflect

julia> struct MyBitsUnionStruct
           u::Union{Int16, Tuple{UInt8, UInt8, UInt8, UInt8, UInt8}}
       end

julia> struct MyUnionStruct
           u::Union{Int16, Vector{UInt8}}
       end

julia> reflect([MyBitsUnionStruct, MyUnionStruct])
#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, ValidField, ConstructType, CCallArg)]
#[jlrs(julia_type = "Main.MyBitsUnionStruct")]
pub struct MyBitsUnionStruct {
    #[jlrs(bits_union_align)]
    _u_align: ::jlrs::data::layout::union::Align2,
    #[jlrs(bits_union)]
    pub u: ::jlrs::data::layout::union::BitsUnion<5>,
    #[jlrs(bits_union_flag)]
    pub u_flag: u8,
}

#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, ValidField, ConstructType, CCallArg)]
#[jlrs(julia_type = "Main.MyUnionStruct")]
pub struct MyUnionStruct<'scope, 'data> {
    pub u: ::std::option::Option<::jlrs::data::managed::value::ValueRef<'scope, 'data>>,
}
```

インラインされたユニオンのサポートは表現に限定されており、`BitsUnion`型は不透明なバイトの塊です。

パラメトリック型のバインディングが要求された場合、最も一般的なバインディングが生成されます：

```julia
julia> using JlrsCore.Reflect

julia> struct MyParametricStruct{T}
           a::T
       end

julia> reflect([MyParametricStruct{UInt8}])
#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, ValidField, IsBits, ConstructType, CCallArg, CCallReturn)]
#[jlrs(julia_type = "Main.MyParametricStruct")]
pub struct MyParametricStruct<T> {
    pub a: T,
}
```

`MyParametricStruct{UInt8}`のバインディングを要求したにもかかわらず、`MyParametricStruct{T}`のバインディングが得られました。

レイアウトに影響を与えない型パラメータは省略されます。この場合、別個のレイアウト型と型コンストラクタが生成され、`HasLayout`トレイトでリンクされます：

```julia
julia> using JlrsCore.Reflect

julia> struct MyElidedStruct{T}
           a::UInt8
       end

julia> reflect([MyElidedStruct{UInt8}])
#[repr(C)]
#[derive(Clone, Debug, Unbox, ValidLayout, Typecheck, ValidField, IsBits)]
#[jlrs(julia_type = "Main.MyElidedStruct")]
pub struct MyElidedStruct {
    pub a: u8,
}

#[derive(ConstructType, HasLayout)]
#[jlrs(julia_type = "Main.MyElidedStruct", constructor_for = "MyElidedStruct", scope_lifetime = false, data_lifetime = false, layout_params = [], elided_params = ["T"], all_params = ["T"])]
pub struct MyElidedStructTypeConstructor<T> {
    _t: ::std::marker::PhantomData<T>,
}
```

[^1]: 要求されたレイアウトに再帰的にインラインされたすべての型が含まれます。

[^2]: アトミックフィールドを持つ型には型コンストラクタが生成されます。
