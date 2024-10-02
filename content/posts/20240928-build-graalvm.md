+++
title = 'GraalVM をビルドする'
date = 2024-09-28T09:00:00+09:00
slug = '34f2d9d91c05126ef05570b0993b9534'
tags = ['GraalVM']
showtoc = true
+++
最近 GraalVM に興味があり、ここ1、2週間ほどいろいろ触ってみている。ここ数年仕事で Java を使っているというのと自分がもともと低いレイヤーの技術、ソフトウェアに興味があるので趣味で触る題材としてちょうどよさそうというのが興味を持ったきっかけである。とはいえ興味があるのは GraalVM を使って何ができるかではなく、GraalVM 自体がどのように動いているのか、その仕組みについてである。GraalVM を使ってネイティブバイナリをビルドしてみた的な話題はよく見かけるが GraalVM 自体の動きなどについての話は多くは見かけないので自分用のメモがてら残しておく。今回は GraalVM のビルドについて。なお、GraalVM には大きく分けて JIT compiler, Native Image, 多言語プログラミング対応の3つの機能があるが、Native Image 機能に最も興味があるため今後は主に Native Image について記載していく。

- 公式サイト: <a href="https://www.graalvm.org/" target="_blank">graalvm.org</a>
- リポジトリ: <a href="https://github.com/oracle/graal" target="_blank">github.com/oracle/graal</a>

## 環境

WSL 2 + Ubuntu 24.04 を使用している。

```
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.1 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=noble
LOGO=ubuntu-logo
$ uname -a
Linux DESKTOP-JNQ4G48 5.15.153.1-microsoft-standard-WSL2 #1 SMP Fri Mar 29 23:14:13 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

## ドキュメント

ドキュメントは主に下記を参照した。(GraalVM のドキュメント、いまいちどこに何があるかわかりづらいと思うのは私だけだろうか・・)

- 公式
    - <a href="https://github.com/oracle/graal/blob/master/docs/reference-manual/native-image/contribute/DevelopingNativeImage.md" target="_blank">docs/reference-manual/native-image/contribute/DevelopingNativeImage.md</a>
    - <a href="https://github.com/oracle/graal/blob/master/compiler/README.md" target="_blank">compiler/README.md</a> (Native Image 機能の README ではないけど)
- その他参考にさせていただいたサイト:
    - <a href="https://www.sakatakoichi.com/entry/graalvmsubstratevmprepare" target="_blank">GraalVMのSubstrateVM (1): ネイティブイメージ関連のコードを読む準備 - Fight the Future</a>
    - <a href="https://foivos.zakkak.net/tutorials/getting_started_with_graalvm_development/" target="_blank">Getting started with GraalVM development | Foivos.Zakkak.Net</a>

## mx

ビルドを始めるにあたりさっそく登場するのが mx という初めて聞くツールである。GraalVM ではビルドなどの開発のためのツールとして Gradle や Maven ではなく mx を使用している。GraalVM プロジェクトのためだけに作られたものなのだろうか。たぶんそうなんだろう。

- リポジトリ: <a href="https://github.com/graalvm/mx" target="_blank">graalvm/mx</a>

まずは mx をインストールする。とはいってもクローンしてパスを通すだけである。

```
$ git clone https://github.com/graalvm/mx.git
$ export PATH=$PATH:$PWD/mx
$ mx --version
mx version 7.31.2
```

指定可能なコマンドは `mx` もしくは `mx help` で確認可能。後者は全量表示される。

## JVMCI 対応 JDK

次は JVMCI に対応した JDK を落としてきて `JAVA_HOME` に設定する。まだよくわかってないのだけど JVMCI は JVM Compiler Interface の略らしい。たぶん普通の JDK とは違い JVM の何らかのインタフェースが公開されていて使うことができる、みたいなものだと勝手に思っているが時間があったらちゃんと調べてみたい。JVMCI 対応 JDK は GitHub に<a href="https://github.com/orgs/graalvm/repositories?type=all&q=labs-openjdk" target="_blank">用意されている</a>ので好みのバージョンのリポジトリのリリースページからダウンロードしてくればよいのだが、mx でも専用のコマンドが用意されているので今回はそちらを使用した。

```
$ mx fetch-jdk
[1]   labsjdk-ce-17             | ce-17.0.7+4-jvmci-23.1-b02
[2]   labsjdk-ce-17-debug       | ce-17.0.7+4-jvmci-23.1-b02
[3]   labsjdk-ce-17-llvm        | ce-17.0.7+4-jvmci-23.1-b02
[4]   labsjdk-ce-19             | ce-19.0.1+10-jvmci-23.0-b04
[5]   labsjdk-ce-19-debug       | ce-19.0.1+10-jvmci-23.0-b04
[6]   labsjdk-ce-19-llvm        | ce-19.0.1+10-jvmci-23.0-b04
[7]   labsjdk-ce-20             | ce-20.0.1+9-jvmci-23.1-b02
[8]   labsjdk-ce-20-debug       | ce-20.0.1+9-jvmci-23.1-b02
[9]   labsjdk-ce-20-llvm        | ce-20.0.1+9-jvmci-23.1-b02
[10]  labsjdk-ce-21             | ce-21.0.2+13-jvmci-23.1-b33
[11]  labsjdk-ce-21-debug       | ce-21.0.2+13-jvmci-23.1-b33
[12]  labsjdk-ce-21-llvm        | ce-21.0.2+13-jvmci-23.1-b33
[13]  labsjdk-ce-latest         | ce-24+16-jvmci-b01
[14]  labsjdk-ce-latest-debug   | ce-24+16-jvmci-b01
[15]  labsjdk-ce-latest-llvm    | ce-24+16-jvmci-b01
[16]  Other version
Select JDK> 10
Install labsjdk-ce-21-jvmci-23.1-b33 to /home/gingk/.mx/jdks/labsjdk-ce-21-jvmci-23.1-b33? [Yn]:
Fetching labsjdk-ce-21-jvmci-23.1-b33 archive from https://github.com/graalvm/labs-openjdk-21/releases/download/jvmci-23.1-b33/labsjdk-ce-21.0.2%2B13-jvmci-23.1-b33-linux-amd64.tar.gz...
Installing labsjdk-ce-21-jvmci-23.1-b33 to /home/gingk/.mx/jdks/labsjdk-ce-21-jvmci-23.1-b33...
Run the following to set JAVA_HOME in your shell:
export JAVA_HOME=/home/gingk/.mx/jdks/labsjdk-ce-21-jvmci-23.1-b33
```

希望のバージョンを番号で選択する。今回は最新の LTS バージョンということで 21 を選択した。メッセージにも出ている通り、`JAVA_HOME` を設定する。

```
$ export JAVA_HOME=~/.mx/jdks/labsjdk-ce-21-jvmci-23.1-b33
```

## ツールチェーン

次に、ドキュメントに記載の通り、必要なツールチェーンをインストールしていく。

> For compilation, `native-image` depends on the local toolchain. Install
`glibc-devel`, `zlib-devel` (header files for the C library and `zlib`) and
`gcc`, using a package manager available on your OS. Some Linux distributions
may additionally require `libstdc++-static`.

ただ、自分の環境でビルドのために何が必要だったかがうろ覚えになってしまっているので、apt のヒストリーを見て自分がインストールしたものを記載しておく。必要最低限ではないかもしれない。

```
$ apt install build-essential
$ apt install zlib1g-dev
```

## ビルド

ようやく準備が整ったのでビルドしていく。Native Image 機能はリポジトリ内の `substratevm/` ディレクトリが対応しているため、このディレクトリ内でコマンドを実行していく。なお、Substrate VM については下記の通り。ドキュメントより引用。

> Substrate VM is an internal project name for the technology behind GraalVM Native Image.

リポジトリをクローンしてディレクトリ移動。なお、mx には suite という概念があり、コマンドは primary suite に対して実行される。primary suite はオプションで明示的に指定することもできるが、指定がない場合は今いるディレクトリが primary suite となる。つまり、下記の場合の primary suite は `substratevm` ということになる。詳しくは mx の <a href="https://github.com/graalvm/mx/blob/master/README.md" target="_blank">README</a> を参照されたし。

```
$ git clone https://github.com/oracle/graal.git
$ cd graal/substratevm
```

以下のコマンドでビルド。

```
$ mx build
JAVA_HOME: /home/gingk/.mx/jdks/labsjdk-ce-21-jvmci-23.1-b33
10 unsatisfied dependencies were removed from build (use -v to list them)
3 non-default dependencies were removed from build (use -v to list them, mx build --all to build them)
Compiling com.oracle.mxtool.compilerserver with javac(JDK 21)... [/home/gingk/work/202409_graalvm/mx/mxbuild/jdk21/com.oracle.mxtool.compilerserver/bin/com/oracle/mxtool/compilerserver/JavacDaemon.class does not exist]
Compiling com.oracle.mxtool.webserver with javac-daemon(JDK 21)... [/home/gingk/work/202409_graalvm/mx/mxbuild/jdk21/com.oracle.mxtool.webserver/bin/com/oracle/mxtool/webserver/WebServer.class does not exist]
...
...
Compiling jdk.graal.compiler.virtual.bench with javac-daemon(JDK 21)... [dependency jdk.graal.compiler.microbenchmarks updated]
Archiving GRAAL_TEST_PREVIEW_FEATURE... [dependency jdk.graal.compiler.hotspot.jdk21.test updated]
Archiving GRAAL_COMPILER_WHITEBOX_MICRO_BENCHMARKS... [dependency jdk.graal.compiler.virtual.bench updated]
```

なお、GraalVM のバージョンはこちらのコマンドで確認できる。今回は 24.2.0-dev を使用している。

```
$ mx graalvm-version
24.2.0-dev
```

また、primary suite と関連する suite のコミットハッシュはこちらのコマンドで確認できる。

```
$ mx sversions
b9aa8b8b24ab  substratevm /home/gingk/work/202409_graalvm/graal
b9aa8b8b24ab  compiler /home/gingk/work/202409_graalvm/graal
b9aa8b8b24ab  truffle /home/gingk/work/202409_graalvm/graal
b9aa8b8b24ab  sdk /home/gingk/work/202409_graalvm/graal
b9aa8b8b24ab  regex /home/gingk/work/202409_graalvm/graal
```

## 使ってみる

無事にビルドできたので試しに使ってみる。こちらも mx コマンドが使える。

準備

```
$ cat HelloWorld.java
class HelloWorld {

    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
$ $JAVA_HOME/bin/javac HelloWorld.java
```

バイナリ生成

```
$ mx native-image HelloWorld
========================================================================================================================
GraalVM Native Image: Generating 'helloworld' (executable)...
========================================================================================================================
[1/8] Initializing...                                                                                    (4.4s @ 0.17GB)
 Java version: 21.0.2+13, vendor version: GraalVM CE 21.0.2-dev+13.1
 Graal compiler: optimization level: 2, target machine: x86-64-v3
 C compiler: gcc (linux, x86_64, 13.2.0)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
 1 user-specific feature(s):
 - com.oracle.svm.thirdparty.gson.GsonFeature
------------------------------------------------------------------------------------------------------------------------
Build resources:
 - 26.18GB of memory (84.7% of 30.91GB system memory, determined at start)
 - 24 thread(s) (100.0% of 24 available processor(s), determined at start)
[2/8] Performing analysis...  [****]                                                                     (2.4s @ 0.45GB)
    3,159 reachable types   (68.2% of    4,629 total)
    3,666 reachable fields  (42.8% of    8,565 total)
   14,669 reachable methods (43.1% of   34,051 total)
      998 types,    12 fields, and   143 methods registered for reflection
       57 types,    57 fields, and    52 methods registered for JNI access
        4 native libraries: dl, pthread, rt, z
[3/8] Building universe...                                                                               (0.8s @ 0.44GB)
[4/8] Parsing methods...      [*]                                                                        (0.4s @ 0.48GB)
[5/8] Inlining methods...     [***]                                                                      (0.3s @ 0.43GB)
[6/8] Compiling methods...    [**]                                                                       (2.2s @ 0.57GB)
[7/8] Laying out methods...   [*]                                                                        (0.8s @ 0.40GB)
[8/8] Creating image...       [*]                                                                        (0.8s @ 0.69GB)
   4.79MB (36.89%) for code area:     8,306 compilation units
   7.19MB (55.31%) for image heap:   93,161 objects and 56 resources
   1.01MB ( 7.80%) for other data
  12.99MB in total
------------------------------------------------------------------------------------------------------------------------
Top 10 origins of code area:                                Top 10 object types in image heap:
   3.46MB java.base                                            1.27MB byte[] for code metadata
 950.48kB svm.jar (Native Image)                               1.21MB byte[] for java.lang.String
 113.97kB java.logging                                       908.41kB java.lang.String
  69.28kB org.graalvm.nativeimage.base                       745.55kB java.lang.Class
  49.71kB jdk.proxy2                                         354.41kB heap alignment
  49.06kB jdk.proxy1                                         279.47kB byte[] for general heap data
  26.22kB jdk.internal.vm.ci                                 271.48kB com.oracle.svm.core.hub.DynamicHubCompanion
  20.32kB org.graalvm.collections                            259.78kB java.util.HashMap$Node
  12.17kB jdk.proxy3                                         215.76kB java.lang.Object[]
   8.55kB jdk.graal.compiler                                 178.99kB java.lang.String[]
   2.34kB for 3 more packages                                  1.57MB for 879 more object types
------------------------------------------------------------------------------------------------------------------------
Recommendations:
 HEAP: Set max heap for improved and more predictable memory usage.
 CPU:  Enable more CPU features with '-march=native' for improved performance.
------------------------------------------------------------------------------------------------------------------------
                        0.8s (5.8% of total time) in 59 GCs | Peak RSS: 1.62GB | CPU load: 11.53
------------------------------------------------------------------------------------------------------------------------
Build artifacts:
 /home/gingk/work/202409_graalvm/graal/substratevm/helloworld (executable)
========================================================================================================================
Finished generating 'helloworld' in 12.6s.
```

バイナリ実行

```
$ ./helloworld
Hello, World!
```

`Hello, World!` が出力された。

なお、 `mx native-image HelloWorld` の実態としてはビルド時に生成された `native-image` を実行しているにすぎない。詳細オプションを付与して実行した際に出力されるメッセージからわかる。

```
$ mx -V native-image HelloWorld
...
[415347: started subprocess 415828: ['/home/gingk/work/202409_graalvm/graal/sdk/mxbuild/linux-amd64/GRAALVM_0292E55835_JAVA21/graalvm-0292e55835-java21-24.2.0-dev/bin/native-image', 'HelloWorld', '-Dllvm.bin.dir=/home/gingk/work/202409_graalvm/graal/sdk/mxbuild/linux-amd64/LLVM_TOOLCHAIN/bin/']]
...
```

上記出力にも含まれているが、ビルドして生成された `native-image` がどこに生成されるかは下記のコマンドで確認可能。

```
$ mx graalvm-home
/home/gingk/work/202409_graalvm/graal/sdk/mxbuild/linux-amd64/GRAALVM_0292E55835_JAVA21/graalvm-0292e55835-java21-24.2.0-dev
```

また、`sdk` ディレクトリに下記のシンボリックリンクができているはずなので、こちらを使うとより簡単にアクセスできる。

```
$ ls -l ../sdk/latest_graalvm*
lrwxrwxrwx 1 gingk gingk 45 Sep 28 01:27 ../sdk/latest_graalvm -> mxbuild/linux-amd64/GRAALVM_0292E55835_JAVA21
lrwxrwxrwx 1 gingk gingk 82 Sep 28 01:27 ../sdk/latest_graalvm_home -> mxbuild/linux-amd64/GRAALVM_0292E55835_JAVA21/graalvm-0292e55835-java21-24.2.0-dev
```

ということで最後に直接実行してみる。

```
$ ../sdk/latest_graalvm_home/bin/native-image HelloWorld
========================================================================================================================
GraalVM Native Image: Generating 'helloworld' (executable)...
========================================================================================================================
[1/8] Initializing...                                                                                    (2.8s @ 0.18GB)
 Java version: 21.0.2+13, vendor version: GraalVM CE 21.0.2-dev+13.1
 Graal compiler: optimization level: 2, target machine: x86-64-v3
 C compiler: gcc (linux, x86_64, 13.2.0)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
 1 user-specific feature(s):
 - com.oracle.svm.thirdparty.gson.GsonFeature
------------------------------------------------------------------------------------------------------------------------
Build resources:
 - 26.25GB of memory (84.9% of 30.91GB system memory, determined at start)
 - 24 thread(s) (100.0% of 24 available processor(s), determined at start)
[2/8] Performing analysis...  [****]                                                                     (2.4s @ 0.32GB)
    3,159 reachable types   (68.2% of    4,632 total)
    3,666 reachable fields  (42.8% of    8,565 total)
   14,669 reachable methods (43.0% of   34,146 total)
      998 types,    12 fields, and   143 methods registered for reflection
       57 types,    57 fields, and    52 methods registered for JNI access
        4 native libraries: dl, pthread, rt, z
[3/8] Building universe...                                                                               (0.8s @ 0.39GB)
[4/8] Parsing methods...      [*]                                                                        (0.4s @ 0.40GB)
[5/8] Inlining methods...     [***]                                                                      (0.2s @ 0.51GB)
[6/8] Compiling methods...    [**]                                                                       (2.3s @ 0.69GB)
[7/8] Laying out methods...   [*]                                                                        (0.9s @ 0.41GB)
[8/8] Creating image...       [*]                                                                        (0.8s @ 0.70GB)
   4.79MB (36.89%) for code area:     8,306 compilation units
   7.19MB (55.31%) for image heap:   93,165 objects and 56 resources
   1.01MB ( 7.80%) for other data
  12.99MB in total
------------------------------------------------------------------------------------------------------------------------
Top 10 origins of code area:                                Top 10 object types in image heap:
   3.46MB java.base                                            1.27MB byte[] for code metadata
 950.48kB svm.jar (Native Image)                               1.21MB byte[] for java.lang.String
 113.97kB java.logging                                       908.50kB java.lang.String
  69.28kB org.graalvm.nativeimage.base                       745.55kB java.lang.Class
  49.71kB jdk.proxy2                                         354.19kB heap alignment
  49.06kB jdk.proxy1                                         279.56kB byte[] for general heap data
  26.22kB jdk.internal.vm.ci                                 271.48kB com.oracle.svm.core.hub.DynamicHubCompanion
  20.32kB org.graalvm.collections                            260.11kB java.util.HashMap$Node
  12.17kB jdk.proxy3                                         215.76kB java.lang.Object[]
   8.55kB jdk.graal.compiler                                 178.99kB java.lang.String[]
   2.34kB for 3 more packages                                  1.57MB for 879 more object types
------------------------------------------------------------------------------------------------------------------------
Recommendations:
 HEAP: Set max heap for improved and more predictable memory usage.
 CPU:  Enable more CPU features with '-march=native' for improved performance.
------------------------------------------------------------------------------------------------------------------------
                        0.8s (7.0% of total time) in 58 GCs | Peak RSS: 1.48GB | CPU load: 13.45
------------------------------------------------------------------------------------------------------------------------
Build artifacts:
 /home/gingk/work/202409_graalvm/graal/substratevm/helloworld (executable)
========================================================================================================================
Finished generating 'helloworld' in 11.2s.
```

問題なく実行できた。`mx` の初期化処理がないためか、こちらのほうが起動が速い。

## おわりに

今回は GraalVM の Native Image 機能をビルドした。今後はコードを読んだり実際に動かしたりして挙動をより深堀りしていきたい。
