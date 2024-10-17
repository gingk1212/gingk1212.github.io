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
