+++
title = 'AMD-V/SVM を有効化する'
date = 2025-06-11T20:00:00+09:00
slug = '45c588fea2ee6a5c30400e29e0b695e9'
tags = ['hypervisor']
showtoc = true
+++
AMD 向けのハイパーバイザを自作するにあたって、AMD の仮想化支援機能 AMD-V (あるいは SVM) を有効化する方法を調べたのでメモ。BIOS の設定で有効にするとかの話ではなく、そちらは有効になっている前提でハイパーバイザ実行時に有効化するお話。

AMD のドキュメント <a href="https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24593.pdf" target="_blank">AMD64 Architecture Programmer's Manual Volume 2: System Programming</a> の「15 Secure Virtual Machine」に SVM に関することが書かれている。こちらの「15.4 Enabling SVM」に記載の内容によると EFER.SVME bit を 1 にセットすると VMRUN などの仮想化関連の CPU 命令が使えるようになるとのこと。EFER とは Extended Feature Enable Register の略で MSR (Model-Specific Register) のアドレス ` C000_0080h` にアクセスすることで読み書きできる。こちらのレジスタは以下の構成となっており、SVME は12番目のビットであることがわかる。(APM Volume 2, 3.1.7)

![20250611-enable-svm-01.png](../image/20250611-enable-svm-01.png)

SVM を有効化する方法はわかったが、その前にそもそも有効化できるかどうかをチェックする必要がある。同じく「15.4 Enabling SVM」にチェックするための疑似コードが記載されている。

```
if (CPUID Fn8000_0001_ECX[SVM] == 0)
  return SVM_NOT_AVAIL;

if (VM_CR.SVMDIS == 0)
  return SVM_ALLOWED;

if (CPUID Fn8000_000A_EDX[SVML]==0)
  return SVM_DISABLED_AT_BIOS_NOT_UNLOCKABLE
  // the user must change a platform firmware setting to enable SVM
else return SVM_DISABLED_WITH_KEY;
  // SVMLock may be unlockable; consult platform firmware or TPM to obtain the key.
```

一つ目の if 文では CPUID 命令を使用している。CPUID 命令により取得できる情報については <a href="https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24594.pdf" target="_blank">AMD64 Architecture Programmer’s Manual Volume 3: General-Purpose and System Instructions</a> の「Appendix E Obtaining Processor Information Via the CPUID Instruction」を参照。`Fn8000_0001_ECX[SVM]` に関する説明は以下の通り。こちらのビットが立っていなかったら SVM は使用不可能ということになる。(APM Volume 3, E.4.2)

![20250611-enable-svm-02.png](../image/20250611-enable-svm-02.png)

二つ目の if 文では MSR のアドレス `C001_0114h` にアクセスすることで読み書き可能な VM_CR の SVMDIS bit を確認している。VM_CR MSR は以下の構成となっており、SVMDIS は4番目のビットであることがわかる。こちらのビットが立っていなければ SVM は有効化可能ということになる。(APM Volume 2, 15.30.1)

![20250611-enable-svm-03.png](../image/20250611-enable-svm-03.png)

そして最後の if 文で、SVMDIS bit が 1 だった場合に有効化する余地があるかどうかを確認している。コメントを読む限り BIOS レベルで無効化されているかそうでないかみたいなことのように見える。

以上をふまえて、C言語で書いたコードがこちら。最後の if 文の処理は省略している。

```c
typedef struct {
  uint32_t eax;
  uint32_t ebx;
  uint32_t ecx;
  uint32_t edx;
} CpuidRegisters;

CpuidRegisters cpuid(uint32_t leaf, uint32_t subleaf) {
  CpuidRegisters regs = {0};
  __asm__ volatile("cpuid"
                   : "=a"(regs.eax), "=b"(regs.ebx), "=c"(regs.ecx),
                     "=d"(regs.edx)
                   : "a"(leaf), "c"(subleaf));
  return regs;
}

uint64_t read_msr(uint32_t msr) {
  uint32_t eax;
  uint32_t edx;
  __asm__ volatile("rdmsr" : "=a"(eax), "=d"(edx) : "c"(msr));
  return ((uint64_t)edx << 32) | eax;
}

void write_msr(uint32_t msr, uint64_t value) {
  __asm__ volatile("wrmsr"
                   :
                   : "c"(msr), "a"((uint32_t)value),
                     "d"((uint32_t)(value >> 32)));
}

bool is_svm_supported() {
  // Fn8000_0001_ECX[2]: Feature Identifiers, SVM bit
  CpuidRegisters regs = cpuid(0x80000001, 0);
  if ((regs.ecx & (1 << 2)) == 0) {
    return false;
  }

  // VM_CR MSR (C001_0114h), SVMDIS bit
  if ((read_msr(0xC0010114) & (1 << 4)) != 0) {
    return false;
  }

  return true;
}

void enable_svm() {
  // Extended Feature Enable Register (EFER)
  uint64_t efer = read_msr(0xC0000080);
  // EFFR.SVME bit
  efer |= 1ULL << 12;
  write_msr(0xC0000080, efer);
}
```

いま作っているハイパーバイザの開発環境が QEMU on WSL とネストが深く SVM が使えるかどうか不安だったけど、`is_svm_supported()` の結果 `true` が返ってきたのでひとまず安心。
