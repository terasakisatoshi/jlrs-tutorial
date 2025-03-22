# Julia

jlrsは現在、Julia 1.6からJulia 1.11までをサポートしています。最新の安定版を使用することをお勧めします。juliaupを使用することもできますが、手動でJuliaをインストールすることをお勧めします。その理由は、jlrsを正常にコンパイルするためには、Juliaのヘッダーファイルとライブラリへのパスを知っている必要があり、これはjuliaupでは達成が難しい場合があるからです。

これらのパスを認識させるためのプラットフォーム依存の方法はいくつかあります：

#### Linux

`julia`が`/path/to/bin/julia`にある場合、メインのヘッダーファイルは`/path/to/include/julia/julia.h`に、ライブラリは`/path/to/lib/libjulia.so`にあると予想されます。`julia`を`PATH`に追加したくない場合は、代わりに`JULIA_DIR`環境変数を設定できます。`JULIA_DIR=/path/to`の場合、ヘッダーとライブラリは前述のパスに存在する必要があります。

`libjulia.so`を含むディレクトリはライブラリ検索パスに含まれている必要があります。これが満たされていない場合、ライブラリが`/path/to/lib/libjulia.so`にある場合は、`LD_LIBRARY_PATH`環境変数に`/path/to/lib/`を追加する必要があります。

#### Windows

`julia`が`X:\path\to\bin\julia.exe`にある場合、メインのヘッダーファイルは`X:\path\to\include\julia\julia.h`に、ライブラリは`X:\path\to\bin\libjulia.dll`にあると予想されます。代わりに`JULIA_DIR`環境変数を設定できます。`JULIA_DIR=X:\path\to`の場合、ヘッダーとライブラリは前述のパスに存在する必要があります。

`libjulia.dll`を含むディレクトリは、Juliaが埋め込まれている場合、実行時に`Path`に含まれている必要があります。

#### MacOS

`julia`が`/path/to/bin/julia`にある場合、メインのヘッダーファイルは`/path/to/include/julia/julia.h`に、ライブラリは`/path/to/lib/libjulia.dylib`にあると予想されます。`julia`を`PATH`に追加したくない場合は、代わりに`JULIA_DIR`環境変数を設定できます。`JULIA_DIR=/path/to`の場合、ヘッダーとライブラリは前述のパスに存在する必要があります。

`libjulia.dylib`を含むディレクトリはライブラリ検索パスに含まれている必要があります。これが満たされていない場合、ライブラリが`/path/to/lib/libjulia.dylib`にある場合は、`DYLD_LIBRARY_PATH`環境変数に`/path/to/lib/`を追加する必要があります。

It seems like you haven't pasted any content yet. Please provide the Markdown content you would like translated, and I'll assist you with the translation.
