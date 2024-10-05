+++
title = 'GraalVM の Issue を追ってみる'
date = 2024-10-05T18:00:00+09:00
slug = '58825db4be65e7d43485139a76127336'
tags = ['GraalVM']
showtoc = true
+++
GraalVM の挙動を深掘りするにあたり何かとっかかりがあったほうがやりやすいと思い、手ごろな Issue がないか漁ってみたところ見つけたのがこちら。10/5現在まだ解決されていない。

- <a href="https://github.com/oracle/graal/issues/9680" target="_blank">Native images don't print "Helpful NullPointerExceptions" (JEP 358) #9680</a>

今回はあくまで GraalVM により詳しくなるのが目的なので解決は目指さないがあわよくばなにか貢献できればくらいで。

## 環境

- OS: Linux (WSL 2 + Ubuntu 24.04.1)
- GraalVM: 24.2.0-dev (<a href="https://github.com/oracle/graal/tree/85203705d949d9f5e6ad83986bde0015c7f2821a" target="_blank">85203705d949</a>)
- JDK: 21.0.2 (<a href="https://github.com/graalvm/labs-openjdk-21/releases/tag/jvmci-23.1-b33" target="_blank">jvmci-23.1-b33</a>)

## Issue の内容

NullPointerException (以下 NPE) がスローされた際に Helpful NullPointerExceptions が出力されない、という内容。Helpful NullPointerExceptions は <a href="https://openjdk.org/jeps/358" target="_blank">JEP 358</a> で取り込まれた機能で、NPE 発生時に何が null になっているかをメッセージで出力してくれるというもの。Issue と同じ内容のコードをコンパイルし java コマンドにより実行すると確かにメッセージが表示される。

```java
class HelloWorld {

    static final String name = null;

    public static void main(String[] args) {
        System.out.println("Hello world! " + name.length());
    }
}
```

```
$ javac HelloWorld.java
$ java HelloWorld
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.length()" because "HelloWorld.name" is null
        at HelloWorld.main(HelloWorld.java:6)
```

そして native-image を使ってバイナリを生成し実行してみると、確かにメッセージが表示されない。再現した。

```
$ native-image HelloWorld
$ ./helloworld
Exception in thread "main" java.lang.NullPointerException
        at HelloWorld.main(HelloWorld.java:6)
```

## GDB を使ってみる

<a href="https://github.com/oracle/graal/blob/master/docs/reference-manual/native-image/guides/debug-native-executables-with-gdb.md" target="_blank">Debug Native Executables with GDB</a> を参考に GDB で実行してみる。まずはデバッグ用のバイナリを生成。

```
$ javac -g HelloWorld.java
$ native-image -g -O0 HelloWorld
```

生成されるファイルはこんな感じ。

```
$ ls -1p
HelloWorld.class
HelloWorld.java
gdb-debughelpers.py
helloworld
helloworld.debug
sources/
```

上記の通り、バイナリファイルのほかに `gdb-debughelpers.py`, `helloworld.debug` というファイルや `sources/` というディレクトリも生成される。Python ファイルについては <a href="https://github.com/oracle/graal/blob/master/docs/reference-manual/native-image/guides/debug-native-executables-with-python-helper.md" target="_blank">Debug Native Executables with a Python Helper Script</a> に説明がある。GDB 実行時に読み込むことで、たとえば `String` 型の変数に格納されている文字列を表示することができるようになるなど、Native Image のデバッグがよりやりやすくなるとのこと。今回は Python ファイルも読み込んで GDB 実行してみる。

```
$ gdb -iex "set auto-load safe-path gdb-debughelpers.py" -q helloworld
Reading symbols from helloworld...
Reading symbols from /home/gingk/work/202409_graalvm/helloworld/helloworld.debug...
(gdb) info functions ::main
All functions matching regular expression "::main":
(gdb) 
```

なんと、`main` 関数が見つからない。原因がわからずけっこう困ったが、もしかしたら `main` 関数が例外をスローするだけになっており中身がなさすぎるからか？と思いコードを書き換えて再度トライしたところ無事 `main` 関数が見つかった。詳細はここでは追わないが、そういうものなのだろう。

```java
class HelloWorld {

    static final String name = null;

    public static void main(String[] args) {
        System.out.println("Hello world!");
        System.out.println(name.length());
    }
}
```

```
$ gdb -iex "set auto-load safe-path gdb-debughelpers.py" -q helloworld
Reading symbols from helloworld...
Reading symbols from /home/gingk/work/202409_graalvm/helloworld/helloworld.debug...
(gdb) info functions ::main
All functions matching regular expression "::main":

File HelloWorld.java:
6:      void HelloWorld::main(java.lang.String[]*);
(gdb) 
```

さっそく `main` 関数にブレークポイントを張って例外発生時の挙動を追ってみる。

```
(gdb) b HelloWorld::main
Breakpoint 1 at 0x90000: file HelloWorld.java, line 6.
(gdb) r
Starting program: /home/gingk/work/202409_graalvm/helloworld/helloworld

This GDB supports auto-downloading debuginfo from the following URLs:
  <https://debuginfod.ubuntu.com>
Enable debuginfod for this session? (y or [n]) y
Debuginfod has been enabled.
To make this setting permanent, add 'set debuginfod enabled on' to .gdbinit.
Downloading separate debug info for system-supplied DSO at 0x7ffff7fc3000
Downloading separate debug info for /lib/x86_64-linux-gnu/libz.so.1                                                         [Thread debugging using libthread_db enabled]                                                                               Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[New Thread 0x7ffff747f6c0 (LWP 90246)]

Thread 1 "helloworld" hit Breakpoint 1, HelloWorld::main(java.lang.String[]*) (args=java.lang.String[0])
    at HelloWorld.java:6
6               System.out.println("Hello world!");
(gdb) n
Hello world!
7               System.out.println(name.length());
(gdb) s
com.oracle.svm.core.snippets.ImplicitExceptions::throwNewNullPointerException() ()
    at com/oracle/svm/core/snippets/ImplicitExceptions.java:330
330             vmErrorIfImplicitExceptionsAreFatal(false);
(gdb) n
331             throw new NullPointerException();
(gdb) n
Exception in thread "main" java.lang.NullPointerException
        at HelloWorld.main(HelloWorld.java:7)
        at java.base@21.0.2/java.lang.invoke.LambdaForm$DMH/sa346b79c.invokeStaticInit(LambdaForm$DMH)
[Thread 0x7ffff747f6c0 (LWP 90246) exited]
[Inferior 1 (process 90201) exited with code 01]
(gdb) 
```

NPE が発生するコードを実行すると `ImplicitExceptions` というクラスの中の `throwNewNullPointerException` というメソッドに飛んで、その中で NPE をスローしているということはわかった。

## `throw new NullPointerException()` を追う

ここからは `throw new NullPointerException()` をさらに深掘りしていく。

```
(gdb) n
331             throw new NullPointerException();
(gdb) s
java.lang.NullPointerException::NullPointerException() (this=java.lang.NullPointerException = {...})
    at java/lang/NullPointerException.java:59
59              super();
(gdb) n
60          }
(gdb)
com.oracle.svm.core.snippets.ImplicitExceptions::throwNewNullPointerException() ()
    at com/oracle/svm/core/snippets/ImplicitExceptions.java:331
331             throw new NullPointerException();
(gdb) s
java.lang.NullPointerException::NullPointerException() (this=java.lang.NullPointerException = {...})
    at java/lang/NullPointerException.java:59
59              super();
(gdb) n
Exception in thread "main" java.lang.NullPointerException
        at HelloWorld.main(HelloWorld.java:7)
        at java.base@21.0.2/java.lang.invoke.LambdaForm$DMH/sa346b79c.invokeStaticInit(LambdaForm$DMH)
[Thread 0x7ffff747f6c0 (LWP 269201) exited]
[Inferior 1 (process 269197) exited with code 01]
(gdb) 
```

見てもらえばわかるとおり、step 実行するとなぜか一度戻ってきてまた NullPointerException.java:59 に飛ぶというよくわからない挙動をしている。ただ、メッセージを出力しているのは二回目のほうみたいなのでまずはそちらを深掘りしてみる。

根気強く追っていくと下記の箇所にたどり着いた。`e.printStackTrace(System.err)` を実行すると目的のメッセージが出力されていることから、このメソッドの中をさらに追っていくと原因がわかるかもしれない。

```
(gdb) n
java.lang.ThreadGroup::uncaughtException(java.lang.Thread*, java.lang.Throwable*) (this=java.lang.ThreadGroup = {...},
    t=java.lang.Thread = {...}, e=java.lang.NullPointerException = {...}) at java/lang/ThreadGroup.java:698
698                     e.printStackTrace(System.err);
(gdb) n
java.lang.NullPointerException
        at HelloWorld.main(HelloWorld.java:7)
        at java.base@21.0.2/java.lang.invoke.LambdaForm$DMH/sa346b79c.invokeStaticInit(LambdaForm$DMH)
```

`printStackTrace` メソッドの中をさらに追っていくと今度は下記のメソッドにたどり着いた。このメソッドの中で呼ばれている `getExtendedNPEMessage` メソッドにより Helpful なメッセージを取得しているようだが、GDB で確認すると `extendedMessageState` が 0 になっているため `getExtendedNPEMessage` 自体が呼ばれていないことがわかった。

- <a href="https://github.com/graalvm/labs-openjdk-21/blob/jvmci-23.1-b33/src/java.base/share/classes/java/lang/NullPointerException.java#L113" target="_blank">NullPointerException.java#L113</a>

```java
public String getMessage() {
    String message = super.getMessage();
    if (message == null) {
        synchronized(this) {
            if (extendedMessageState == 1) {
                // Only the original stack trace was filled in. Message will
                // compute correctly.
                extendedMessage = getExtendedNPEMessage();
                extendedMessageState = 2;
            }
            return extendedMessage;
        }
    }
    return message;
}
```

`extendedMessageState` という変数は `NullPointerException` クラス内の下記のメソッドでセットされているようだが、このメソッド自体が呼ばれていないのだろうか。

- <a href="https://github.com/graalvm/labs-openjdk-21/blob/jvmci-23.1-b33/src/java.base/share/classes/java/lang/NullPointerException.java#L81" target="_blank">NullPointerException.java#L81</a>

```java
public synchronized Throwable fillInStackTrace() {
    // If the stack trace is changed the extended NPE algorithm
    // will compute a wrong message. So compute it beforehand.
    if (extendedMessageState == 0) {
        extendedMessageState = 1;
    } else if (extendedMessageState == 1) {
        extendedMessage = getExtendedNPEMessage();
        extendedMessageState = 2;
    }
    return super.fillInStackTrace();
}
```

## `fillInStackTrace` メソッドの謎

> 見てもらえばわかるとおり、step 実行するとなぜか一度戻ってきてまた NullPointerException.java:59 に飛ぶというよくわからない挙動をしている。ただ、メッセージを出力しているのは二回目のほうみたいなのでまずはそちらを深掘りしてみる。

上記のとおり先ほどは二回目のほうを深掘りしたが、二回目の道中では `fillInStackTrace` メソッドは呼ばれていなかった。今度は先ほどスキップしていた一回目のほうの深掘りをしてみる。 (というかこの一回目、二回目は一体なんなのだろうか。一回目は `new NullPointerException()` で、二回目は `throw` のほうとか？ このあたりはまだよくわかっていない)

例によって根気強く追っていくと、なんと `fillInStackTrace` メソッドに行き着いた。ただし、そのクラスは `NullPointerException` クラスではなく `JavaLangSubstitutions` というクラスだった。

- <a href="https://github.com/oracle/graal/blob/a7cae0179c3516c6073179f08a48a151c96ad0d7/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/jdk/JavaLangSubstitutions.java#L603" target="_blank">JavaLangSubstitutions.java#L603</a>

```java
/**
 * {@link NullPointerException} overrides {@link Throwable#fillInStackTrace()} with a
 * {@code synchronized} method which is not permitted in a {@link VMOperation}. We hand over to
 * {@link Target_java_lang_Throwable#fillInStackTrace(int)} which already handles this properly.
 */
@Substitute
@Platforms(InternalPlatform.NATIVE_ONLY.class)
Target_java_lang_Throwable fillInStackTrace() {
    return SubstrateUtil.cast(this, Target_java_lang_Throwable.class).fillInStackTrace(0);
}
```

Substitutions という名の通り、GraalVM では何らかの理由でそのまま使用できない JDK 側のコードをこちらで代わりに実装して処理している、ということなのだろうか。`fillInStackTrace` メソッドもその対象になっていて上書かれてしまっているため `NullPointerException` 側のコードが呼ばれることがないと。

ちなみに、これは Issue の<a href="https://github.com/oracle/graal/issues/9680#issuecomment-2356599598" target="_blank">コメント</a>にも書いてあることだが、`getExtendedNPEMessage` メソッドも `JavaLangSubstitutions` に書かれている。中身は空っぽ。

- <a href="https://github.com/oracle/graal/blob/a7cae0179c3516c6073179f08a48a151c96ad0d7/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/jdk/JavaLangSubstitutions.java#L609" target="_blank">JavaLangSubstitutions.java#L609</a>

```java
    @Substitute
    @SuppressWarnings("static-method")
    private String getExtendedNPEMessage() {
        return null;
    }
```

つまり、この Issue を解決するには `fillInStackTrace` と `getExtendedNPEMessage` を何とかする必要があると思われる。

## 参考: JDK 側の実装

JDK 側の実装がどうなっているのか気になっている調べたところ、ピンポイントのコミットを見つけた。

- <a href="https://github.com/openjdk/jdk/commit/d8c6516c921e1f437c175875c3157ee249a5ca3c" target="_blank">openjdk/jdk@d8c6516</a>

軽くしか見てないのだけど、bytecodeUtils.cpp の `BytecodeUtils::get_NPE_message_at` というメソッドにより Helpful なメッセージを取得しているらしいということはわかった。

## おわりに

原因らしきところまでは特定できたのでこれくらいにしておく。今回は生成したバイナリのデバッグの仕方や例外発生時の挙動、あとは長くなるので省略しているが途中途中でバックトレースなども眺めながら進めていたので `main` メソッドにたどり着くまでの各メソッドの呼ばれ方についてもちょっとだけ知ることができた。こんな感じで気になる点や Issue などのとっかかりを見つけてまたやっていきたい。
