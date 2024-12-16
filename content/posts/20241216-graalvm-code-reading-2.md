+++
title = 'GraalVM Native Image のソースコードを雑に読んだ (2)'
date = 2024-12-16T09:00:00+09:00
slug = 'd48a997e17dfd7088237e529b69653d9'
tags = ['GraalVM']
showtoc = true
+++
<a href="https://gingk1212.github.io/posts/f68be20f03f370e97cd6f1d81794dcdf/" target="_blank">前回</a>の続き。今回は `[3/8] Building universe`, `[4/8] Parsing methods` というステージを取り上げる。

## 環境

- OS: Linux (WSL 2 + Ubuntu 24.04.1)
- GraalVM: 24.2.0-dev (<a href="https://github.com/oracle/graal/tree/320d02ebb8671b9b7035ff86130fafa859ea012f" target="_blank">320d02ebb867</a>)
- JDK: 21.0.2 (<a href="https://github.com/graalvm/labs-openjdk-21/releases/tag/jvmci-23.1-b33" target="_blank">jvmci-23.1-b33</a>)

## 3. Building universe

> In this stage, a universe with all types, fields, and methods is built, which is then used to create the native binary.

処理は `doRun` メソッドの以下の try ブロックの中。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L575" target="_blank">NativeImageGenerator.java#L575</a>

```java
try (ReporterClosable c = reporter.printUniverse()) {
    ...
}
```

前回も何度か参照した記事 <a href="https://gingk1212.github.io/posts/eee1ddb6f0decf0d2823cf27998be3b5/" target="_blank">GraalVM における universe とは</a> でとりあげた `HostedUniverse` の Javadoc に以下の記載がある。

> The {@link HostedUniverse} manages the {@link HostedType types}, {@link HostedMethod methods}, and {@link HostedField fields} that the ahead-of-time (AOT) compilation operates on. These elements are created by the {@link UniverseBuilder} after the static analysis has finished.

hosted universe は静的解析が終わった後に `UniverseBuilder` によって生成されると書いてあるが、上記の try ブロックの中に該当するコードが存在しているため、hosted universe はこのステージで生成されていると考えてよさそう。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L594" target="_blank">NativeImageGenerator.java#L594</a>

```java
new UniverseBuilder(aUniverse, bb.getMetaAccess(), hUniverse, hMetaAccess, HostedConfiguration.instance().createStrengthenGraphs(bb, hUniverse),
                bb.getUnsupportedFeatures()).build(debug);
```

`build` メソッドをのぞいてみたところ、いろいろなことをしているので詳細はわからなかったが、analysis univese から hosted universe をつくっているということはなんとなく読み取れた。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/meta/UniverseBuilder.java#L141" target="_blank">UniverseBuilder.java#L141</a>

```java
public void build(DebugContext debug) {
    ...
    for (AnalysisType aType : aUniverse.getTypes()) {
        makeType(aType);
    }
    ...
    for (AnalysisField aField : aUniverse.getFields()) {
        makeField(aField);
    }
    for (AnalysisMethod aMethod : aUniverse.getMethods()) {
        assert aMethod.isOriginalMethod();
        Collection<MultiMethod> allMethods = aMethod.getAllMultiMethods();
        HostedMethod origHMethod = null;
        if (allMethods.size() == 1) {
            origHMethod = makeMethod(aMethod);
        } else {
            ConcurrentHashMap<MultiMethod.MultiMethodKey, MultiMethod> multiMethodMap = new ConcurrentHashMap<>();
            for (MultiMethod method : aMethod.getAllMultiMethods()) {
                HostedMethod hMethod = makeMethod((AnalysisMethod) method);
                hMethod.setMultiMethodMap(multiMethodMap);
                MultiMethod previous = multiMethodMap.put(hMethod.getMultiMethodKey(), hMethod);
                assert previous == null : "Overwriting multimethod key";
                if (method.equals(aMethod)) {
                    origHMethod = hMethod;
                }
            }
        }
        assert origHMethod != null;
        HostedMethod previous = hUniverse.methods.put(aMethod, origHMethod);
        assert previous == null : "Overwriting analysis key";
    }
    ...
```
`makeType` メソッドでは `hUniverse.types` に、`makeField` メソッドでは `hUniverse.fields` に `put` していた。なお、`HostedUniverse` は以下のように各要素を analysis universe と対応する形で保持している。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/meta/HostedUniverse.java#L292-L294" target="_blank">HostedUniverse.java#L292-L294</a>

```java
protected final Map<AnalysisType, HostedType> types = new HashMap<>();
protected final Map<AnalysisField, HostedField> fields = new HashMap<>();
protected final Map<AnalysisMethod, HostedMethod> methods = new HashMap<>();
```

このステージでは上記の `UniverseBuilder` による hosted universe 作成以外にもいろいろなことをしている。が、ちょっと眺めただけでは何をしているかよくわからなかったのでいったん割愛する。

## 4. Parsing methods

> In this stage, the Graal compiler parses all reachable methods. The progress indicator is printed periodically at an increasing interval.

処理は以下の箇所。初登場の `CompileQueue` クラス。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L401" target="_blank">CompileQueue.java#L401</a>

```java
public void finish(DebugContext debug) {
    ...
    try (ProgressReporter.ReporterClosable ac = reporter.printParsing()) {
        parseAll();
    }
    ...
```

`finish` メソッドは `NativeImageGenerator` クラスの `doRun` メソッドから呼ばれている。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L664" target="_blank">NativeImageGenerator.java#L664</a>

```java
protected void doRun(Map<Method, CEntryPointData> entryPoints, JavaMainSupport javaMainSupport, String imageName, NativeImageKind k, SubstitutionProcessor harnessSubstitutions) {
    ...
    CompileQueue compileQueue;
    try (StopTimer t = TimerCollection.createTimerAndStart(TimerCollection.Registry.COMPILE_TOTAL)) {
        compileQueue = HostedConfiguration.instance().createCompileQueue(debug, featureHandler, hUniverse, runtimeConfiguration, DeoptTester.enabled());
        if (ImageSingletons.contains(RuntimeCompilationCallbacks.class)) {
            ImageSingletons.lookup(RuntimeCompilationCallbacks.class).onCompileQueueCreation(bb, hUniverse, compileQueue);
        }
        compileQueue.finish(debug);
        BuildPhaseProvider.markCompileQueueFinished();
    ...
```

ちなみに、この後のステージである `[5/8] Inlining methods`, `[6/8] Compiling methods` も `finish` メソッドの中で処理されていた

`parseAll` メソッドの中身は以下の通り。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L609" target="_blank">CompileQueue.java#L609</a>

```java
protected void parseAll() throws InterruptedException {
    /*
     * We parse ahead of time compiled methods before deoptimization targets so that we remove
     * deoptimization entrypoints which are determined to be unneeded. This both helps the
     * performance of deoptimization target methods and also reduces their code size.
     */
    runOnExecutor(this::parseAheadOfTimeCompiledMethods);
    runOnExecutor(this::parseDeoptimizationTargetMethods);
}
```

`parseAheadOfTimeCompiledMethods` メソッドと `parseDeoptimizationTargetMethods` メソッドの中身を見てみるとともに `HostedMethod` ごとに `ensureParsed` というメソッドを呼び出しており、さらに処理を追っていくと `defaultParseFunction` メソッドにたどり着く。かなり端折って引用したものがこちら。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L1023" target="_blank">CompileQueue.java#L1023</a>

```java
private void defaultParseFunction(DebugContext debug, HostedMethod method, CompileReason reason, RuntimeConfiguration config, ParseHooks hooks) {
    ...
    StructuredGraph graph = graphTransplanter.transplantGraph(debug, method, reason);
    ...
    method.compilationInfo.encodeGraph(graph);
    ...
```

パース (構文解析) のようなことはしているようには見えない。何らかの移植により `StructuredGraph` 型のグラフを取得して `encodeGraph` メソッドに渡している。このメソッドはグラフをエンコードしたうえで `CompilationInfo` クラスが持つ `CompilationGraph` 型のフィールドにセットしていた。なお、`CompilationGraph` クラスはエンコードされたグラフを表す `EncodedGraph` 型のフィールドを保持している。

ここで再び `HostedUniverse` の Javadoc から引用する。

> Having a separate analysis universe and hosted universe complicates some things. For example, {@link StructuredGraph graphs} parsed for static analysis need to be "transplanted" from the analysis universe to the hosted universe (see code around {@code AnalysisToHostedGraphTransplanter#replaceAnalysisObjects}).

文脈としては analysis universe と hosted universe が分かれていると複雑になることもある、というものであるがそこは置いておいて、静的解析のためにパースされたグラフ (`StructuredGraph`) を analysis universe から hosted universe に移植する必要がある、と記載されている。上記の `transplantGraph` メソッドがまさにこれのことと思われる。実際に、`transplantGraph` は `AnalysisToHostedGraphTransplanter` クラスのメソッドであり、中で `replaceAnalysisObjects` メソッドを呼び出していた。

また、別の箇所では以下の記載もあった。今までグラフと呼んでいたもの、つまり `StructuredGraph` は Graal IR graphs というもののことらしい。

> It is however quite convenient to have parsed {@link StructuredGraph Graal IR graphs} that reference JVMCI objects from a consistent universe.

以上をふまえると、実際のパース自体は静的解析時にすでに行われていて、このステージではパースして得られたグラフの移植のみを行っている、と考えることができる。というわけで、少し戻り `[2/8] Performing analysis` ステージの処理をもう一度見てみることにする。

`AnalysisMethod` クラスを眺めてみると、いかにもパース処理をしていそうな `parseGraph` というメソッドを見つけた。こちらのメソッドにブレークポイントを設定していつ呼ばれているかを確認してみる。なお、実際は以下のように if 文を追記して `System.out.println` の行にブレークポイントを設定した。ビルドしたのは前回も使用した `SampleApp` クラス。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/meta/AnalysisMethod.java#L1099" target="_blank">AnalysisMethod.java#L1099</a>

```java
private AnalysisParsedGraph parseGraph(BigBang bb, Object expectedValue) {
    if (name.equals("hoge")) {
        System.out.println("detected");
    }
    return setGraph(expectedValue, () -> AnalysisParsedGraph.parseBytecode(bb, this));
}
```

結果、案の定 `[2/8] Performing analysis` ステージの処理中に停止した。呼び出し元をたどっていくと前回も見た `runAnalysis` メソッドに行き着く。`AnalysisParsedGraph.parseBytecode` メソッド内で `hoge` メソッドがどのように処理されるかを追っていくと、以下の `apply` メソッドを呼び出している行にたどり着いた。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/flow/AnalysisParsedGraph.java#L144" target="_blank">AnalysisParsedGraph.java#L144</a>

```java
public static AnalysisParsedGraph parseBytecode(BigBang bb, AnalysisMethod method) {
    ...
    graph = new StructuredGraph.Builder(options, debug)
                    .method(method)
                    .recordInlinedMethods(bb.getHostVM().recordInlinedMethods(method))
                    .build();
    ...
    bb.getHostVM().createGraphBuilderPhase(bb.getProviders(method), config, OptimisticOptimizations.NONE, null).apply(graph);
    ...
```

`createGraphBuilderPhase` メソッドは `AnalysisGraphBuilderPhase` というクラスのインスタンスを生成しており、`apply` メソッドをさらにたどっていくと `GraphBuilderPhase` クラスの `run` メソッドに行き着いた。`BytecodeParser` を生成して `buildRootMethod` というメソッドを呼んでおり、まさにパース処理の本丸という感じがする。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/java/GraphBuilderPhase.java#L103" target="_blank">GraphBuilderPhase.java#L103</a>

```java
@Override
protected void run(StructuredGraph graph) {
    createBytecodeParser(graph, null, graph.method(), graph.getEntryBCI(), initialIntrinsicContext).buildRootMethod();
}
```

大枠はつかめたので今回はここまでにしておく。なお、`GraphBuilderPhase` の Javadoc には下記のように書かれている。つまり「パース = Java バイトコードから IR グラフを生成する」と捉えてよさそう。

> Parses the bytecodes of a method and builds the IR graph.

ちなみに実は今回は substratevm ディレクトリだけではなく compiler ディレクトリ配下のクラスも登場していた。登場したクラス tree 形式で一覧化したものがこちら。

```
graal
├── compiler
│   └── src
│       └── jdk.graal.compiler
│           └── src
│               └── jdk
│                   └── graal
│                       └── compiler
│                           ├── java
│                           │   ├── BytecodeParser.java
│                           │   └── GraphBuilderPhase.java
│                           └── nodes
│                               ├── EncodedGraph.java
│                               └── StructuredGraph.java
└── substratevm
    └── src
        ├── com.oracle.graal.pointsto
        │   └── src
        │       └── com
        │           └── oracle
        │               └── graal
        │                   └── pointsto
        │                       ├── flow
        │                       │   └── AnalysisParsedGraph.java
        │                       └── meta
        │                           └── AnalysisMethod.java
        └── com.oracle.svm.hosted
            └── src
                └── com
                    └── oracle
                        └── svm
                            └── hosted
                                ├── NativeImageGenerator.java
                                ├── code
                                │   ├── CompilationGraph.java
                                │   └── CompileQueue.java
                                │── meta
                                │   └── HostedMethod.java
                                └── phases
                                    └── AnalysisGraphBuilderPhase.java
```

最後に、GraalVM には Ideal Graph Visualizer (IGV) というグラフ可視化ツールがあるみたいなのだけど、このグラフが今回取り上げてきたグラフと同じものを指すのかがわかっていない。ツール自体は簡単に試せそうなので今度試してみたい。

- <a href="https://www.graalvm.org/latest/tools/igv/" target="_blank">Ideal Graph Visualizer</a>

## おわりに

次回は `[5/8] Inlining methods` 以降の処理を取り上げる。
