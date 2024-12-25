+++
title = 'GraalVM Native Image のソースコードを雑に読んだ (3)'
date = 2024-12-25T09:00:00+09:00
slug = '49278a58f529469d8f52dfcb36cf3e3e'
tags = ['GraalVM']
showtoc = true
+++
<a href="https://gingk1212.github.io/posts/d48a997e17dfd7088237e529b69653d9/" target="_blank">前回</a>の続き。今回は残りのステージ (`[5/8] Inlining methods`, `[6/8] Compiling methods`, `[7/8] Laying out methods`, `[8/8] Creating image`) を取り上げる。

## 環境

- OS: Linux (WSL 2 + Ubuntu 24.04.1)
- GraalVM: 24.2.0-dev (<a href="https://github.com/oracle/graal/tree/320d02ebb8671b9b7035ff86130fafa859ea012f" target="_blank">320d02ebb867</a>)
- JDK: 21.0.2 (<a href="https://github.com/graalvm/labs-openjdk-21/releases/tag/jvmci-23.1-b33" target="_blank">jvmci-23.1-b33</a>)

## 5. Inlining methods

> In this stage, trivial method inlining is performed. The progress indicator visualizes the number of inlining iterations.

該当する処理は以下の箇所。前回の `[4/8] Parsing methods` 同様 `finish` メソッドの中。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L429" target="_blank">CompileQueue.java#L429</a>

```java
public void finish(DebugContext debug) {
    ...
    try (ProgressReporter.ReporterClosable ac = reporter.printInlining()) {
        inlineTrivialMethods(debug);
    }
    ...
```

`inlineTrivialMethods` メソッドでは、`HostedMethod` ごとに `TrivialInlineTask` を実行しているようだ。なお、実行には例によって `CompletionExecutor` を使用している。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L714" target="_blank">CompileQueue.java#L714</a>

```java
protected void inlineTrivialMethods(DebugContext debug) throws InterruptedException {
    ...
    runOnExecutor(() -> {
        universe.getMethods().forEach(method -> {
            assert method.isOriginalMethod();
            for (MultiMethod multiMethod : method.getAllMultiMethods()) {
                HostedMethod hMethod = (HostedMethod) multiMethod;
                if (hMethod.compilationInfo.getCompilationGraph() != null) {
                    executor.execute(new TrivialInlineTask(hMethod));
                }
            }
        });
    });
    ...
```

`TrivialInlineTask` クラスの `run` メソッドでは `doInlineTrivial` というメソッドを呼び出しているのみだった。このメソッドからなんとなく重要そうな箇所を抜き出してみるとこんな感じ。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L793" target="_blank">CompileQueue.java#L793</a>

```java
private void doInlineTrivial(DebugContext debug, HostedMethod method) {
    ...
    boolean inliningPotential = false;
    ...
    if (!inliningPotential) {
        return;
    }
    ...
    try (var s = debug.scope("InlineTrivial", graph, method, this)) {
        ...
        new TrivialInlinePhase(decoder, method).apply(graph);

        if (inliningPlugin.inlinedDuringDecoding) {
            CanonicalizerPhase.create().apply(graph, providers);
            ...
```

前半部分でインライン化できそうかどうかを判定しているように見える。そして後半部分の `TrivialInlinePhase` クラス、`CanonicalizerPhase` クラスで実際にインライン化していると推測。`TrivialInlinePhase` クラスの `run` メソッドを見てみると `PEGraphDecoder` クラスの `decode` メソッドを呼んでいた。`PEGraphDecoder` の Javadoc には以下のように書かれているので、やはりここでインライン化が行われていそうである。

> A graph decoder that performs partial evaluation, i. e., that performs method inlining and canonicalization/simplification of nodes during decoding.

一方 `CanonicalizerPhase` については、軽くのぞいてはみたものの何をしているのかよくわからかなかった。その名の通り何らかの正規化をやっているのだろうけれど。

## 6. Compiling methods

> In this stage, the Graal compiler compiles all reachable methods to machine code. The progress indicator is printed periodically at an increasing interval.

該当する処理は以下の箇所。こちらも `finish` メソッドの中。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L438" target="_blank">CompileQueue.java#L438</a>

```java
public void finish(DebugContext debug) {
    ...
    try (ProgressReporter.ReporterClosable ac = reporter.printCompiling()) {
        compileAll();
        notifyAfterCompile();
    }
    ...
```

`compileAll` メソッドをたどっていくと以下の箇所に行き着く。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/code/CompileQueue.java#L1297" target="_blank">CompileQueue.java#L1297</a>

```java
private CompilationResult defaultCompileFunction(DebugContext debug, HostedMethod method, CompilationIdentifier compilationIdentifier, CompileReason reason, RuntimeConfiguration config) {
    ...
    CompilationResult result = backend.newCompilationResult(compilationIdentifier, method.getQualifiedName());
    ...
    GraalCompiler.compile(new GraalCompiler.Request<>(graph,
                    method,
                    providers,
                    backend,
                    null,
                    optimisticOpts,
                    null,
                    suites,
                    lirSuites,
                    result,
                    new HostedCompilationResultBuilderFactory(),
                    false));
    ...
    return result;
    ...
}
```

この `defaultCompileFunction` メソッドは `HostedMethod` ごとに呼ばれている。`CompilationResult` クラスの Javadoc は以下の通り。これにコンパイルされて生成された機械語も入ってくるらしい。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/code/CompilationResult.java#L69" target="_blank">CompilationResult.java#L69</a>

```java
/**
 * Represents the output from compiling a method, including the compiled machine code, associated
 * data and references, relocation information, deoptimization information, etc.
 */
public class CompilationResult {
```

`compile` メソッドはこちら。フロントエンドとバックエンドで処理が分かれているらしいことはわかる。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/core/GraalCompiler.java#L143" target="_blank">GraalCompiler.java#L143</a>

```java
/**
 * Services a given compilation request.
 *
 * @return the result of the compilation
 */
@SuppressWarnings("try")
public static <T extends CompilationResult> T compile(Request<T> r) {
    ...
    emitFrontEnd(r.providers, r.backend, r.graph, r.graphBuilderSuite, r.optimisticOpts, r.profilingInfo, r.suites);
    r.backend.emitBackEnd(r.graph, null, r.installedCodeOwner, r.compilationResult, r.factory, r.entryPointDecorator, null, r.lirSuites);
    ...
    return r.compilationResult;
...
}
```

この辺り、まったくわからなかったのでいろいろ調べてみたところ、以下の記事が大変参考になりました。

- <a href="https://www.sakatakoichi.com/entry/2017/12/12/190000" target="_blank">GraalでのHIRとLIR - Fight the Future</a>

こちらによると、フロントエンドとバックエンドでは以下の処理をやっているとのこと。

- フロントエンド
    - バイトコードから HIR (High-level Intermediate Representation) を生成
    - HIR を最適化
- バックエンド
    - HIR から LIR (Low-level Intermediate Representation) を生成
    - レジスタ割り付け
    - LIR から機械語を生成

`emitFrontEnd` メソッドを見てみると、これまでに作成してきたグラフ ( `StructuredGraph` )に対して HighTier, MiddleTier, LowTier の3段階に分けて何らかの処理を施しているように見える。HIR がグラフのことだとすると、「バイトコードから HIR を生成」はすでに実施済みのはずなので、「HIR を最適化」を実行しているのだろうか。ちょっとまだわからなかった。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/core/GraalCompiler.java#L260" target="_blank">GraalCompiler.java#L260</a>

```java
/**
 * Builds the graph, optimizes it.
 */
@SuppressWarnings("try")
public static void emitFrontEnd(Providers providers, TargetProvider target, StructuredGraph graph, PhaseSuite<HighTierContext> graphBuilderSuite, OptimisticOptimizations optimisticOpts,
                ProfilingInfo profilingInfo, Suites suites) {
    HighTierContext highTierContext = new HighTierContext(providers, graphBuilderSuite, optimisticOpts);
    ...
    suites.getHighTier().apply(graph, highTierContext);
    ...
    MidTierContext midTierContext = new MidTierContext(providers, target, optimisticOpts, profilingInfo);
    suites.getMidTier().apply(graph, midTierContext);
    ...
    LowTierContext lowTierContext = new LowTierContext(providers, target);
    suites.getLowTier().apply(graph, lowTierContext);
    ...
```

`emitBackEnd` メソッド側も見てみる。推測でしかないが、名前から察するに `emitLIR` メソッドで LIR を生成し、`emitCode` メソッドで機械語を生成しているのではなかろうか。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/core/gen/LIRCompilerBackend.java#L80" target="_blank">LIRCompilerBackend.java#L80</a>

```java
public static void emitBackEnd(StructuredGraph graph, Object stub, ResolvedJavaMethod installedCodeOwner, Backend backend, CompilationResult compilationResult,
                CompilationResultBuilderFactory factory, EntryPointDecorator entryPointDecorator, RegisterConfig registerConfig, LIRSuites lirSuites) {
    ...
    LIRGenerationResult lirGen = emitLIR(backend, graph, stub, registerConfig, lirSuites, entryPointDecorator);
    ...
    emitCode(backend,
                    graph.getAssumptions(),
                    graph.method(),
                    graph.getMethods(),
                    graph.getSpeculationLog(),
                    bytecodeSize,
                    lirGen,
                    compilationResult,
                    installedCodeOwner,
                    factory,
                    entryPointDecorator);
    ...
```

とりあえずこれくらいにしておく。コンパイル関連は知識が乏しすぎてちょっと厳しい・・。なお、今回登場したクラスは `CompileQueue` を除きすべて compiler ディレクトリ配下にあるものだった。てっきり compiler 配下のものは JIT 関連でしか使われていないのかと思っていたが、Native Image 側からもがっつり使われていたのであった。

## 7. Laying out methods

> In this stage, compiled methods are laid out. The progress indicator is printed periodically at an increasing interval.

該当する処理は以下の箇所。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L670" target="_blank">NativeImageGenerator.java#L670</a>

```java
protected void doRun(Map<Method, CEntryPointData> entryPoints, JavaMainSupport javaMainSupport, String imageName, NativeImageKind k, SubstitutionProcessor harnessSubstitutions) {
    ...
    NativeImageCodeCache codeCache;
    ...
    try (ProgressReporter.ReporterClosable ac = reporter.printLayouting()) {
        codeCache = NativeImageCodeCacheFactory.get().newCodeCache(compileQueue, heap, loader.platform,
                        ImageSingletons.lookup(TemporaryBuildDirectoryProvider.class).getTemporaryBuildDirectory());
        codeCache.layoutConstants();
        codeCache.layoutMethods(debug, bb);
        codeCache.buildRuntimeMetadata(debug, bb.getSnippetReflectionProvider());
    }
    ...
```

`layoutConstants` メソッドは `codeCache` の `constantReasons`, `dataSection` フィールドにデータを追加していた。名前から察するに ELF で言うところの .data や .bss、.rodata セクションなどに配置されるデータを取り扱っているのだろうか。

`layoutMethods` メソッドは `HostedMethod` のオフセットアドレスを計算してセットしていくという処理を主にやっているようだった。そして最後にコードエリアのサイズを計算してセットしている。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/image/LIRNativeImageCodeCache.java#L140" target="_blank">LIRNativeImageCodeCache.java#L140</a>

```java
public void layoutMethods(DebugContext debug, BigBang bb) {
    ...
    method.setCodeAddressOffset(curPos);
    ...
    setCodeAreaSize(totalSize);
    ...
```

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/meta/HostedMethod.java#L105" target="_blank">HostedMethod.java#L105</a>

```java
/**
 * The address offset of the compiled code relative to the code of the first method in the
 * buffer.
 */
private int codeAddressOffset;
```

最後の `buildRuntimeMetadata` はけっこう長いメソッドでぱっと見何をしているかわからなかったのでいったん断念。また、runtime metadata が以下のドキュメントで取り上げられている reachability metadata と同じものを指しているかについてもわかっていない。

- <a href="https://www.graalvm.org/latest/reference-manual/native-image/metadata/" target="_blank">Reachability Metadata</a>

## 8. Creating image

> In this stage, the native binary is created and written to disk. Debug info is also generated as part of this stage (if requested).

該当の処理は以下の `printCreationStart` から `printCreationEnd` の間。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L689" target="_blank">NativeImageGenerator.java#L689</a>

```java
protected void doRun(Map<Method, CEntryPointData> entryPoints, JavaMainSupport javaMainSupport, String imageName, NativeImageKind k, SubstitutionProcessor harnessSubstitutions) {
    ...
    reporter.printCreationStart();
    ...
    reporter.printCreationEnd(image.getImageFileSize(), heap.getLayerObjectCount(), image.getImageHeapSize(), codeCache.getCodeAreaSize(), numCompilations, image.getDebugInfoSize());
    ...
```

ビルド時にメッセージに出力される内容については以下で説明されている。

- <a href="https://www.graalvm.org/latest/reference-manual/native-image/overview/BuildOutput/#creating-image" target="_blank">Native Image Build Output</a>

`Feature` 関連を除くと、主な処理はこんな感じ。まずは `heap` をビルドしてから `image` をビルド、書き出している。

```java
protected void doRun(Map<Method, CEntryPointData> entryPoints, JavaMainSupport javaMainSupport, String imageName, NativeImageKind k, SubstitutionProcessor harnessSubstitutions) {
    ...
    verifyAndSealShadowHeap(codeCache, debug, heap);

    buildNativeImageHeap(heap, codeCache);
    ...
    createAbstractImage(k, hostedEntryPoints, heap, hMetaAccess, codeCache);
    ...
    image.build(imageName, debug);
    ...
    Path tmpDir = ImageSingletons.lookup(TemporaryBuildDirectoryProvider.class).getTemporaryBuildDirectory();
    LinkerInvocation inv = image.write(debug, generatedFiles(HostedOptionValues.singleton()), tmpDir, imageName, beforeConfig);
    ...
```

ほぼ何も見てないが、個々の処理を追うのはだいぶ骨が折れそうだったためここまでにしておく。

## おわりに

最後の方の失速感は否めないが、雑に読んだシリーズ終了。なんとなーくどこで何をしているかがわかってきた。今後はもう少しポイントを絞って挙動を追っていきたい。
