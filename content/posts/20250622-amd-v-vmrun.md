+++
title = 'AMD-V におけるゲストモードへの移行と復帰の流れ'
date = 2025-06-22T17:00:00+09:00
slug = '79d9e88b3251ce982c2c7b36fa456f66'
tags = ['hypervisor']
showtoc = true
+++
AMD-V (SVM) を使ってゲスト側で HLT 命令をループして実行するだけのシンプルなコードを動かすことができたので、ゲストモードへの移行とホストモードへの復帰の流れをメモしておく。[前回の記事](https://gingk1212.github.io/posts/45c588fea2ee6a5c30400e29e0b695e9/)で書いた AMD-V の有効化はしてある前提。なお、今回参照したドキュメントは <a href="https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24593.pdf" target="_blank">AMD64 Architecture Programmer's Manual Volume 2: System Programming</a> の主に以下の項目。

- 15.5 VMRUN Instruction
- 15.6 #VMEXIT
- Appendix B VMCB Layout

AMD-V では、ゲストモードへの移行は VMRUN 命令により行われる。また、ホストモードへの復帰は #VMEXIT と呼ばれる (こちらは命令ではない)。流れを先に書いてしまうと以下の通り。

```
下準備
↓
VMRUN 命令実行 (1)
↓
(ゲストモードでコードが実行される)
↓
(何らかの要因で #VMEXIT 発生)
↓
EXIT 要因に応じて何らかの処理
↓
VMRUN 命令実行 (2)
↓
...
```

以下、それぞれについて記載していく。

## 下準備

まずは VMRUN 命令を実行するためにいくつかの下準備をする必要がある。

- VMCB 用の領域を確保して各フィールドに値を設定する
- ホストの状態を保存しておくための領域を確保してその物理アドレスを VM_HSAVE_PA MSR へ書き込む
- 必要に応じて、ホストの汎用レジスタの保存とゲストの汎用レジスタの復元を行う

VMCB は Virtual Machine Control Block の略で、ゲストの動作に必要な設定や CPU の状態を保持しておくための領域のことである。大きく Control Area と State Save Area に分かれており、Control Area にはたとえばゲスト側で何の命令が実行されたら #VMEXIT を発生させるかなどの情報が格納される。State Save Area にはゲストの状態、たとえば Segment Register, Control Register, RIP などを格納するために使われる。一覧はドキュメントの Appendix B VMCB Layout を参照。また、VMCB 用の領域は 4KB アラインされたアドレスに確保する必要があり、VMRUN を実行する際にこの領域の物理アドレスを RAX レジスタに格納しておくという決まりとなっている。

なお、シンプルなコードを動かすだけであれば以下を設定しておけば十分だった。

- Control Area
    - Intercept VMRUN instruction (これは必須でビットを立てておく必要がある)
    - Guest ASID
- State Save Area
    - CS
    - EFER
    - CR0, CR3, CR4
    - RIP

2点目についてはそのままで、VMRUN 実行時に CPU 側でホストの状態を保存しておいてくれるので、その保存先 (host state-save area) をこちらで確保して設定しておく必要がある。#VMEXIT が発生した際にはこの領域からホストの状態が復元される。確保しておくべきサイズは 4KB であり、VM_HSAVE_PA MSR へ物理アドレスを設定しておく。

3点目については、RAX レジスタ以外の汎用レジスタの中身は VMRUN/#VMEXIT 時に CPU が何もしてくれないので、ホスト/ゲストそれぞれで保存/復元したいのであれば自分で対応する必要がある、という話。必要最小限のシンプルなコードを動かすだけであれば不要だが、本格的にゲスト側で処理を実行する際は必要不可欠。

## VMRUN 命令実行 (1)

先述した通り、RAX レジスタに VMCB の物理アドレスを格納したうえで、VMRUN 命令を実行する。VMRUN 命令を実行した際に、CPU はホストの状態の保存 (to host state-save area) とゲストの状態の復元 (from VMCB) を行ってくれる。そして VMCB 内の RIP フィールドに設定してあるアドレスにジャンプしてゲストモードでコードを実行し始める。

VMLOAD 命令についても触れておく。VMRUN 命令は VMCB からすべての状態を復元するわけではない。VMLOAD 命令は VMRUN 命令だけでは復元されない状態を復元してくれる、つまり VMRUN 命令による復元の補完的役割を持つ。実行すべきタイミングは VMRUN 命令の前。こちらも RAX レジスタに VMCB の物理アドレスを格納したうえで実行する必要がある。なぜ VMRUN 命令とは別で VMLOAD 命令が用意されているかはわからない。VMRUN 命令で完結させてしまえばいいのにと思ってしまうけど。

## 何らかの要因で #VMEXIT 発生

#VMEXIT が発生すると、CPU はゲストの状態を VMCB に書き戻し、ホストの状態を host state-save area から復元してくれる。そして、VMRUN の次の命令からホスト側のコードの実行を再開する (Intel VT-x の場合は VMCS に再開するアドレスを設定しておくらしい)。

#VMEXIT が発生した際、必要に応じて VMSAVE 命令も実行しておく。先ほどの VMLOAD 命令に対応するもので、VMCB へのゲスト状態の書き戻しを補完してくれる命令となる。つまり流れは以下の通り。

```
VMLOAD (必要に応じて)
↓
VMRUN
↓
#VMEXIT 発生
↓
VMSAVE (必要に応じて)
```

## EXIT 要因に応じて何らかの処理

EXIT した要因は VMCB 内の EXITCODE, EXITINFO1, EXITINFO2, EXITINTINFO というフィールドに書き込まれているので、これらを参照して必要な処理を実行すればよい。ドキュメントの「Appendix C SVM Intercept Exit Codes」に EXITCODE の一覧が記載されている。

## VMRUN 命令実行 (2)

#VMEXIT が発生したあと再度ゲストモードへ移行する際の話も書いておく。

VMRUN 命令を実行する前に、先述した通り汎用レジスタの保存/復元は必要に応じてやっておく必要がある。

あとは、VMCB 内の RIP フィールドに「#VMEXIT 発生時にゲスト側で実行されていた命令の次の命令のアドレス」を格納しておく必要もある。というのも、VMCB の RIP には #VMEXIT 発生時に実行していた命令のアドレスが書き戻されるからである。たとえば、ゲスト側で HLT 命令が実行された際に #VMEXIT を発生させるよう設定していたとして、何もしないと VMRUN → HLT → #VMEXIT → VMRUN → HLT → #VMEXIT → ... というふうに延々と HLT 命令を繰り返すことになってしまう。そのため、再度 VMRUN を実行する前に VMCB 内の RIP フィールドに次の命令のアドレスを格納してあげる必要がある。次の命令のアドレスは親切にも VMCB に項目が用意されていて、こちらも #VMEXIT 発生時に CPU が書き戻してくれるためそのままこちらを RIP フィールドに書き込んであげればいいだけである。こんなイメージ。

```c
vmcb->rip = vmcb->nrip;
```

## おわりに

ページングや割り込み、I/O などなどゲストモードへの移行に関して他にもたくさん話題はあると思うけど、あくまでシンプルなコードを動かすための第一歩的な内容を書いた。最後に、Writing Hypervisor in Zig に沿って開発中のハイパーバイザの途中経過を貼っておく。ゲストモードに移行して HLT を実行して #VMEXIT して HLT と出力してまた移行して HLT を実行して、を繰り返している図 (これだけ見てもなにがなんだかだけど)。

![20250622-amd-v-vmrun-01.gif](../image/20250622-amd-v-vmrun-01.gif)
