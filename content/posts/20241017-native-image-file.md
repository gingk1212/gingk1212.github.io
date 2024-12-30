+++
title = 'native-image ファイルの正体 - GraalVM'
date = 2024-10-17T09:40:12+09:00
slug = 'a29e6d8462910862f392aa7e0bc07af9'
tags = ['GraalVM']
showtoc = true
+++
GraalVM をビルドすると生成される native-image ファイル。このファイルを実行することでバイナリファイルを生成することができるが、native-image ファイル自身はどのように生成されているのだろうか。native-image を生成するためにも自身のバイナリ生成機能を使っている？ コンパイラ自身はどのようにコンパイルされているのか的な。

## 正体

なんてことはない、native-image ファイルはただのシェルスクリプトだった。勝手に ELF などの実行形式だと思い込んでいたのだけど、そうではなかった。substratevm ディレクトリで build して生成された native-image ファイルを `file` コマンドで確認した結果。

```
$ file ../sdk/latest_graalvm_home/lib/svm/bin/native-image
../sdk/latest_graalvm_home/lib/svm/bin/native-image: Bourne-Again shell script, ASCII text executable, with very long lines (5506)
```

生成元のテンプレートと思われるファイルも発見した。このテンプレート中の `<>` で囲まれている箇所を解決したものが native-image ファイル、ということのようである。

- <a href="https://github.com/oracle/graal/blob/master/sdk/mx.sdk/vm/launcher_template.sh" target="_blank">launcher_template.sh</a>

## なにをしているのか

ではこのスクリプトはなにをしているのか。

以下が native-image ファイルの最終行。見やすいように改行を追加している。java コマンドでクラスを指定して実行していることがわかる。

```
exec "${location}/../../../bin/java" \
  -XX:MaxHeapSize=214748365 -XX:+UnlockExperimentalVMOptions -XX:+EnableJVMCI \
  "${jvm_args[@]}" ${app_path_arg} "${cp_or_mp}" ${main_class} "${launcher_args[@]}"
```

`${main_class}` を出力してみた。つまり、native-image ファイルは `NativeImage` クラスを java コマンドで実行しているにすぎない。

```
--module org.graalvm.nativeimage.driver/com.oracle.svm.driver.NativeImage
```

## さらにその先

native-image には `--verbose` という詳細出力を有効にするオプションがある。こちらを指定して実行してみる。

```
$ native-image --verbose HelloWorld
...
Executing [
...
/path/to/graal/sdk/mxbuild/linux-amd64/GRAALVM_0292E55835_JAVA21/graalvm-0292e55835-java21-24.2.0-dev/bin/java \
...
--module \
org.graalvm.nativeimage.builder/com.oracle.svm.hosted.NativeImageGeneratorRunner \
...
]
========================================================================================================================
GraalVM Native Image: Generating 'helloworld' (executable)...
========================================================================================================================
[1/8] Initializing...                                                                                    (2.6s @ 0.17GB)
...
```

なんと、`NativeImage` クラス自身も java コマンドを実行していた。今度は以下のクラスが指定されている。

```
org.graalvm.nativeimage.builder/com.oracle.svm.hosted.NativeImageGeneratorRunner
```

なお、以下の箇所で `ProcessBuilder` クラスの `start` メソッドにより java コマンドを実行しているようである。

- <a href="https://github.com/oracle/graal/blob/a5eb4a5fdcee9ef5cd9c108a853fa9839815793c/substratevm/src/com.oracle.svm.driver/src/com/oracle/svm/driver/NativeImage.java#L1792" target="_blank">NativeImage.java#L1792</a>

```java
p = pb.inheritIO().start();
imageBuilderPid = p.pid();
return p.waitFor();
```

おそらくここから本格的にバイナリファイル生成処理に移っていくと思われるが、今回はここまで。

## 追記

2024.12.31 追記

native-image ファイルはシェルスクリプト、と記載したが、native-image ファイルを実行形式としてビルドすることも可能だった。たとえば substratevm ディレクトリで、`mx graalvm-show` コマンドを実行した結果が以下。

```
$ mx graalvm-show
...
Launchers:
 - native-image (bash, rebuildable)
 - native-image-configure (bash, rebuildable)
 - polyglot (bash, rebuildable)
Libraries:
 - libjvmcicompiler.so (skipped, rebuildable)
 - libnative-image-agent.so (skipped, rebuildable)
 - libnative-image-diagnostics-agent.so (skipped, rebuildable)
 - libsvmjdwp.so (skipped, rebuildable)
No standalone
```

`native-image (bash, rebuildable)` と記載されている通り、このまま `mx build` を実行すると native-image はシェルスクリプトとして生成される。では、今度は `--print-env` というオプションを指定して実行してみる。

```
$ mx graalvm-show --print-env
...
Inferred env file:
DYNAMIC_IMPORTS=/compiler,/regex,/sdk,/substratevm,/truffle
COMPONENTS=antlr4,cmp,dis,icu4j,lg,llp,nfi,nfi-libffi,ni,nic,nil,nju,poly,rgx,sdk,sdkc,sdkl,sdkni,svm,svmjdwp,svml,svmnfi,svmsl,svmt,tfl,tfla,tflc,tflm,tflp,tflsm,truffle-json,xz
EXCLUDE_COMPONENTS=libpoly
NATIVE_IMAGES=false
NON_REBUILDABLE_IMAGES=False
```

すると、先ほどの出力結果に加えて上記の通り環境変数の設定状況も表示される。気になるのは `NATIVE_IMAGES=false` という部分。こちらを true にして再度実行。

```
$ NATIVE_IMAGES=true mx graalvm-show --print-env
...
Launchers:
 - native-image (native, rebuildable)
 - native-image-configure (native, rebuildable)
 - polyglot (native, rebuildable)
Libraries:
 - libjvmcicompiler.so (native, rebuildable)
 - libnative-image-agent.so (native, rebuildable)
 - libnative-image-diagnostics-agent.so (native, rebuildable)
 - libsvmjdwp.so (native, rebuildable)
No standalone
Inferred env file:
DYNAMIC_IMPORTS=/compiler,/regex,/sdk,/substratevm,/truffle
COMPONENTS=antlr4,cmp,dis,icu4j,lg,llp,nfi,nfi-libffi,ni,nic,nil,nju,poly,rgx,sdk,sdkc,sdkl,sdkni,svm,svmjdwp,svml,svmnfi,svmsl,svmt,tfl,tfla,tflc,tflm,tflp,tflsm,truffle-json,xz
EXCLUDE_COMPONENTS=libpoly
NATIVE_IMAGES=lib:jvmcicompiler,lib:native-image-agent,lib:native-image-diagnostics-agent,lib:svmjdwp,native-image,native-image-configure,polyglot
NON_REBUILDABLE_IMAGES=False
```

すると、先ほどは `bash` と表示されていた箇所が軒並み `native` に変わっている。ついでに Libraries の項目も `skipped` から `native` に変わっている。`NATIVE_IMAGES` の個別指定もできそうなので試してみる。

```
$ NATIVE_IMAGES=native-image mx graalvm-show --print-env
...
Launchers:
 - native-image (native, rebuildable)
 - native-image-configure (bash, rebuildable)
 - polyglot (bash, rebuildable)
Libraries:
 - libjvmcicompiler.so (skipped, rebuildable)
 - libnative-image-agent.so (skipped, rebuildable)
 - libnative-image-diagnostics-agent.so (skipped, rebuildable)
 - libsvmjdwp.so (skipped, rebuildable)
No standalone
Inferred env file:
DYNAMIC_IMPORTS=/compiler,/regex,/sdk,/substratevm,/truffle
COMPONENTS=antlr4,cmp,dis,icu4j,lg,llp,nfi,nfi-libffi,ni,nic,nil,nju,poly,rgx,sdk,sdkc,sdkl,sdkni,svm,svmjdwp,svml,svmnfi,svmsl,svmt,tfl,tfla,tflc,tflm,tflp,tflsm,truffle-json,xz
EXCLUDE_COMPONENTS=libpoly
NATIVE_IMAGES=native-image
NON_REBUILDABLE_IMAGES=False
```

案の定、native-image のみ変更された。それでは `NATIVE_IMAGES=native-image` を指定したうえでビルドを実行する。

```
$ NATIVE_IMAGES=native-image mx build
```

生成されたファイルを確認。

```
$ file ../sdk/latest_graalvm_home/bin/native-image
../sdk/latest_graalvm_home/bin/native-image: symbolic link to ../lib/svm/bin/native-image
```

どうやらシンボリックリンクになっているようなので本体のほうを確認。

```
$ file ../sdk/latest_graalvm_home/lib/svm/bin/native-image
../sdk/latest_graalvm_home/lib/svm/bin/native-image: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=56742ae6a7b3dd83171eb482a731dd3a43ab302e, for GNU/Linux 3.2.0, stripped
```

想定通り、シェルスクリプトではなく ELF ファイルが生成されている。
