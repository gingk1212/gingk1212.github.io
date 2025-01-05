+++
title = 'GraalVM をビルドする - Windows 編'
date = 2025-01-05T09:00:31+09:00
slug = '91b1beaa98eb4e171bb8e54c411f8998'
tags = ['GraalVM']
showtoc = true
+++
以前の記事 <a href="/posts/34f2d9d91c05126ef05570b0993b9534/" target="_blank">GraalVM をビルドする</a> では Linux 環境における GraalVM のビルド方法を紹介した。今回は Windows 環境におけるビルド方法についてまとめておく。大筋は Linux と変わらないのだけど、いろいろハマりポイントがあったのでその辺を中心に記載する。なお、ビルド対象は substratevm。

## 環境

- OS: Windows 11 Home, 23H2
- GraalVM: 25.0.0-dev (<a href="https://github.com/oracle/graal/tree/ff5ecc8e18e55d74b11d416f93bc059bfd3d93f7" target="_blank">ff5ecc8e18e5</a>)
- mx: 7.36.5
- Python: 3.13.1
- Visual Studio Community 2022: 17.12.3

## ビルド

最初に書いた通り大筋は以前の記事と同様。まずは mx の準備。<a href="https://github.com/graalvm/mx" target="_blank">graalvm/mx</a> をクローンしてパスを通す。Python がインストールされていなかったので Python もインストールした。

次に、JVMCI が有効化された JDK をダウンロード。以前と同様 `mx fetch-jdk` コマンドでダウンロードし、`JAVA_HOME` に設定。

次に、ツールチェーンをインストール。<a href="https://github.com/oracle/graal/blob/ff5ecc8e18e55d74b11d416f93bc059bfd3d93f7/docs/reference-manual/native-image/contribute/DevelopingNativeImage.md" target="_blank">こちら</a>に記載されている通り、Windows の場合は Visual Studio 2022 が必要となる。特に記載されていないが、インストール時に「C++ によるデスクトップ開発」をチェックを入れてインストールした (なくても問題ないかもしれない)。

準備が整ったのでビルドする。<a href="https://github.com/oracle/graal" target="_blank">oracle/graal</a> をクローンして substratevm ディレクトリに移動し、以下のコマンドを実行。

```
>mx build
...
Traceback (most recent call last):
  ...
  File "C:\Users\gingk\work\202412_graalvm\mx\src\mx\_impl\support\processes.py", line 59, in _check_output_str
    return subprocess.check_output(*args, **kwargs).decode()
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x82 in position 91: invalid start byte
```

残念ながらエラーに。

## ハマりポイント①

上記のエラーについて調べた結果、javac コマンドの出力結果が日本語であることが原因だった。日本語をデコードしようとしてエラーとなっているようだ。同様の issue も見つかった。

- <a href="https://github.com/graalvm/mx/issues/265" target="_blank">graalvm/mx#265</a>

issue のコメントに書かれている通り mx 側に変更を加えるのもよいが、`JAVA_TOOL_OPTIONS` という環境変数を設定することで javac の出力を英語にすることもできるようなのでそちらの方法を採用。

```
>set JAVA_TOOL_OPTIONS=-Duser.language=en
```

再度ビルド。

```
>mx build
...
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder failed
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder... [C:\Users\gingk\work\202412_graalvm\graal\substratevm\mxbuild\jdk25\svm-jvmfuncs-fallback-builder\gensrc\JvmFuncsFallbacks.c does not exist]
Error executing: dumpbin /SYMBOLS C:\Users\gingk\.mx\jdks\labsjdk-ce-latest-25+2-jvmci-b01\lib\static\windows-amd64\attach.lib
[WinError 2] 指定されたファイルが見つかりません。
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder: Failed due to error: 1
1 build tasks failed
```

まだエラー。

## ハマりポイント②

dumpbin というコマンドが見つからないと言われている。このツールは Visual Studio 2022 をインストールした際に併せてインストールされているみたいだったので、ターミナルとして Visual Studio 2022 に付属の Developer Command Prompt for VS 2022 を使用するようにした。ちゃんと dumpbin にパスが通っている。

```
>where dumpbin
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.42.34433\bin\Hostx64\x64\dumpbin.exe
```

再度ビルド。

```
>mx build
...
Exception in thread Thread-152 (redirect):-jvmfuncs-fallback-builder | JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder... [C:\Use
Traceback (most recent call last):
  File "C:\Users\gingk\AppData\Local\Programs\Python\Python313\Lib\threading.py", line 1041, in _bootstrap_inner
    self.run()
    ~~~~~~~~^^
  File "C:\Users\gingk\AppData\Local\Programs\Python\Python313\Lib\threading.py", line 992, in run
    self._target(*self._args, **self._kwargs)
    ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\gingk\work\202412_graalvm\mx\src\mx\_impl\mx.py", line 13447, in redirect
    f(line.decode())
      ~~~~~~~~~~~^^
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x83 in position 0: invalid start byte
...
```

またデコードでエラー。

## ハマりポイント③

今度は Visual Studio 2022 付属のコマンド cl.exe の出力メッセージが日本語になっていることが原因だった。Visual Studio Installer を起動して言語パックの日本語のチェックを外し、英語にチェックを入れて再度インストール。cl.exe の実行結果が英語になったことを確認したうえで再度ビルド。

```
>mx build
WARNING: symlinking not supported
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en
JAVA_HOME: C:\Users\gingk\.mx\jdks\labsjdk-ce-latest-25+2-jvmci-b01
9 unsatisfied dependencies were removed from build (use -v to list them)
3 non-default dependencies were removed from build (use -v to list them, mx build --all to build them)
mx build log written to C:\Users\gingk\work\202412_graalvm\graal\substratevm\mxbuild\buildlog-20241224-102338.html
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en
Picked up JAVA_TOOL_OPTIONS: -Duser.language=enntlr.v4.runtime | Note: Recompile with -Xlint:removal for details.
WARNING:   File "C:\Users\gingk\work\202412_graalvm\graal\substratevm\mx.substratevm\suite.py", line 648 in definition of com.oracle.svm.hosted:
Package java.lang.instrument is not concealed in module java.instrument
WARNING:   File "C:\Users\gingk\work\202412_graalvm\graal\substratevm\mx.substratevm\suite.py", line 648 in definition of com.oracle.svm.hosted:
Package java.lang.instrument is not concealed in module java.instrument
[0/357/357] done
mx build log written to C:\Users\gingk\work\202412_graalvm\graal\substratevm\mxbuild\buildlog-20241224-102852.html
```

ようやく成功。長かった。

## 動作確認

最後に、native-image コマンドを実行してバージョンを表示してみる。

```
>..\sdk\mxbuild\windows-amd64\GRAALVM_9DA8311EB5_JAVA25\graalvm-9da8311eb5-java25-25.0.0-dev\bin\native-image.cmd --version
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en
native-image 25 2025-09-16
OpenJDK Runtime Environment GraalVM CE None-devNone.1 (build 25+2-jvmci-b01)
OpenJDK 64-Bit Server VM GraalVM CE None-devNone.1 (build 25+2-jvmci-b01, mixed mode, sharing)
```
