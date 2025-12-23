---
layout: post
title: "Rust向け自作アクセス権制御ライブラリ「mprotect-rs」の紹介"
tags: [x86_64, Linux, Ubuntu, Rust, CPU]
excerpt_separator: <!--more-->
---

メモリークリスマス！
前回に引き続き、今回も Intel MPK/PKU 関連のお話です。
自分はハードウェア支援のメモリ安全性やアクセス制御に興味があり、昨今、Rust 向けにこんなライブラリを試作しています。

- [yotiosoft/mprotect-rs: An implementation of mprotect() and pkey_mprotect() for Rust. This enables Rust to set access rights to each pages, using PTE flags or Intel MPK (Memory Protection Keys).](https://github.com/yotiosoft/mprotect-rs){:target="_blank"}

まだまだ開発途中で未完成ですが、今回はこのライブラリの簡単な紹介と、今後目指す理想像についてお話したいと思います。

<!--more-->

# 実現したいこと

このライブラリで実現したいことは、ざっくりまとめるとこんな感じです。

- ユーザ空間アプリケーションのアクセス権制御（read/write アクセス権の設定）を実現したい
- もし不正なアクセス（read-only なメモリ領域への write アクセスなど）が起きたら、それを未然に防止する仕組みを実現したい
- 以上の操作を Rust で簡単に実現できるようにしたい

<img src="../../../assets/img/post/2025-12-24-mprotect-rs/rw_251224.svg" alt="rw_251224" style="zoom:50%;" />

# 目的とアプローチ

このライブラリの目的は、Linux 環境の Rust 製ユーザアプリケーションに `mprotect()` と Intel PKU をはじめとするハードウェア支援のアクセス制御を容易に導入できるようにし、不正なメモリアクセスを防止することにあります。

Rust をご存知の方は、こんなことを疑問に思ったかもしれません。「Rust って既にメモリ安全言語じゃないか」と。
その疑問は正しいです。ただ、Rust のメモリ安全性はあくまでもコンパイル時点でコンパイラによって担保できる範囲内であり、unsafe code やレガシー言語による外部ライブラリなど、rustc による検証が及ばない範囲もあります。

そこで、機密データを置くメモリ領域をあらかじめメモリドメインに設定しておき、仮に unsafe code や外部ライブラリでバグにより脆弱性が発生したとしても、それをハードウェア的に防止するための仕組みが重要になると考えています。

同じような思想は、EuroSys'22 で発表されたこちらの「PKRU-safe」という論文にも現れています。こちらの論文では、レガシーな言語（C など）で書かれた外部ライブラリを Untrusted memory として Trusted memory から切り離してしまおうというアプローチを取っており、その実現に Intel PKS を利用しています。

- [PKRU-safe \| Proceedings of the Seventeenth European Conference on Computer Systems](https://dl.acm.org/doi/10.1145/3492321.3519582){:target="_blank"}

`mprotect()` と Intel PKU の違いは、前者はカーネルが PTE (Page Table Entry) のアクセス権限を更新することでアクセス制御を実現するのに対し、後者はユーザが PKRU というレジスタを更新することでアクセス制御を実現します。
Intel PKU はハードウェア依存で x86_64 の Skylake 世代でしか利用できませんが、アクセス制御にカーネルの介入が必要ない分、より高速なアクセス制御を実現できます。

## Intel MPK / Intel PKU とは

Intel x86_64 アーキテクチャで提供されている、ハードウェアレベルのメモリ保護機能です。

Intel MPK にはユーザ空間向けの Intel PKU (Protection Keys for Userspace) とカーネル空間向けの Intel PKS (Protection Keys for supervisor) があり、今回は前者の Intel PKU を扱います。近年の Intel CPU（2015年発売の Skylake 世代移行）であれば基本的に搭載されている機能ですので、実は Intel CPU であれば気軽に遊べます。

詳しくは前回の記事をご覧ください。

- [x86_64のメモリ保護機能「Intel MPK」で遊ぼう \| 為せばnull](https://blog.yotio.jp/2025/12/14/intel-mpk.html)

# 実現したこと

## コンパイラによるアクセス正当性チェックを実現

別に `mprotect()` や Intel PKU は、ライブラリ無しでも利用できます。前者は Linux システムコールですし、後者は非特権 CPU 命令です。
ですが、`mprotect()` や Intel PKU はある意味では安全であり、ある意味では危険です。「安全」な点は、メモリ安全、つまり不正なアクセスをトラップして防止できるという点にありますが、「危険」な点は、実際に不正なアクセスが発生してしまうと、プロセスが Segmentation fault を起こしてクラッシュするという点です。

これはフェイルセーフの観点から正しい動作ではありますが、アプリケーションとしてはクラッシュするのは極力避けたい側面もあります。もしコンパイル段階で静的解析によって safe code 内に不正なアクセス操作があること、例えば、read-only アクセスのメモリ領域に write しようとしているコードがあることが分かっているのであれば、コンパイル段階でコンパイルエラーとして扱った方が嬉しいでしょう。

ですので、mprotect-rs では借用チェックやトレイトを活用し、「read-only で宣言したスマートポインタは、read アクセスしかできない」「read-write で宣言したスマートポインタは、read/write アクセスが可能」といったように、ポインタレベルで read/write アクセスを制御し、違反するアクセスに関しては rustc が検知できるようにしています。

```rust
    // mutable な参照を取得し値を書き込む. read/write 可
    {
        // 参照を取得
        // このとき、Protection Key のアクセス権が ReadWrite に変更され、RegionGuard への read/write アクセスが許可される
        let write_guard = associated_region.set_access_rights::<PkeyPermissions::ReadWrite>().map_err(RuntimeError::MprotectError)?;
        let mut mut_ref_guard = write_guard.mut_ref_guard().map_err(|e| RuntimeError::PkeyGuardError(e))?;
        // write
        *mut_ref_guard = 123;
        // read
        println!("Value written via associated region deref(): {}", *mut_ref_guard);
    }
    // immutable な参照を取得し値を読む. read のみ可
    {
        // 参照を取得
        // このとき、Protection Key のアクセス権が ReadOnly に変更され、RegionGuard への read アクセスのみが許可される
        let read_guard = associated_region.set_access_rights::<PkeyPermissions::ReadOnly>().map_err(RuntimeError::MprotectError)?;
        let ref_guard = read_guard.ref_guard().map_err(|e| RuntimeError::PkeyGuardError(e))?;
        // read
        println!("Value read via associated region deref(): {}", *ref_guard);
        
        // write は borrow checker によりコンパイルエラーになる（ここ重要）
        //*ref_guard = 456;
    }
```

具体的には、それぞれ read 操作しかトレイト実装していないスマートポインタを `&T` で、read 操作と write 操作をトレイト実装したスマートポインタを `&mut T` で返すようにしています。

```rust
// read-only で返すスマートポインタ (GuardRef)
impl<'a, A: allocator::Allocator<T>, T> Deref for GuardRef<'a, A, T> {
    type Target = T;
    /// Dereferences the guarded reference if valid, panicking otherwise.
    fn deref(&self) -> &Self::Target {
        if self.is_valid() {
            &*self.ptr
        } else {
            panic!("Failed to deref GuardRef: invalid generation");
        }
    }
}

// read/write で返すスマートポインタ (GuardRefMut)
impl<'a, A: allocator::Allocator<T>, T> Deref for GuardRefMut<'a, A, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        if self.is_valid() {
            unsafe { &*self.ptr }
        } else {
            panic!("Failed to deref GuardRefMut: invalid generation");
        }
    }
}
impl<'a, A: allocator::Allocator<T>, T> DerefMut for GuardRefMut<'a, A, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        if self.is_valid() {
            unsafe { &mut *self.ptr }
        } else {
            panic!("Failed to deref_mut GuardRefMut: invalid generation");
        }
    }
}
```

もし read-only で参照を取得した場合、immutable なスマートポインタしか実装されていませんので、write 操作しようとすると下の画像のようにコンパイルエラーになります。

![image-20251224003858351](../../../assets/img/post/2025-12-24-mprotect-rs/image-20251224003858351.webp)

これによって、コンパイル段階で `mprotect()` や Intel PKU による Segmentation fault の発生を予防できるわけですね。

では、コンパイラで事前に検知できるのなら `mprotect()` や Intel PKU がいらないのではないか？といえばそんなことはありません。unsafe code や、C言語などで書かれた外部依存ライブラリに不正にアクセスできてしまうコードが含まれていた場合、それらは rustc による安全性チェックが行われませんので、unsafe code での不正アクセス発生時はやむを得ず `mprotect()` や Intel PKU によって Segmentation fault を起こすようにしています。

```rust
    // immutable な参照を取得し値を読む. read のみ可
    {
        // 参照を取得
        // このとき、Protection Key のアクセス権が ReadOnly に変更され、RegionGuard への read アクセスのみが許可される
        let read_guard = associated_region.set_access_rights::<PkeyPermissions::ReadOnly>().map_err(RuntimeError::MprotectError)?;
        let ref_guard = read_guard.ref_guard().map_err(|e| RuntimeError::PkeyGuardError(e))?;
        // read
        println!("Value read via associated region deref(): {}", *ref_guard);
        // write は borrow checker によりコンパイルエラーになる
        //*ref_guard = 456;

        // unsafe code で無理やり mutable な参照を取得しようとすると、実行時に segmentation fault になる
        unsafe {
            let mut_ref = ref_guard.ptr() as *const u32 as *mut u32;
            println!("Attempt to write via unsafe mutable reference...");
            *mut_ref = 789;  // ここで segmentation fault になる
        }
```

こちらのコードでは、read-only アクセスに向けて immutable で取得した参照に対して、unsafe code で無理やり値を書き込もうとしています。

![image-20251224005614578](../../../assets/img/post/2025-12-24-mprotect-rs/image-20251224005614578.webp)

コンパイルによる検証が実行されないのでコンパイル自体は通りますが、実行結果は、もちろん Segmentation fault です。不正なアクセスが Intel PKU によって防止できていることが確認できますね。

まとめると、

- safe code 内で起きる不正なアクセスはコンパイル段階で検知しよう
- unsafe code や 外部ライブラリの FFI などで起きうる不正なアクセスは `mprotect()` や Intel PKU に任せよう

という設計方針です。

## システムコールを呼び出さないアクセス制御を実現（Intel PKU のみ）

`mprotect()` はシステムコール呼び出しが必要、かつその都度 PTE を更新しなければならないので、それなりにランタイムオーバーヘッドがかさみます。
一方、Intel PKU はユーザモードで PKRU レジスタを更新するだけでアクセス権の変更が完了しますので、カーネルの介入、PTE の更新によるランタイムオーバーヘッドは生じません。

mprotect-rs では、この特徴を最大限に生かすために、Intel PKU のアクセス制御で済む操作はユーザ側で完結するような設計にしています。
Intel PKU では、各メモリ領域に対して最低でも一度は `mprotect_pkey()` システムコールの呼び出しが必要です。このシステムコールは PTE と Protection Key を紐づけるために呼び出す操作です。一方で、一度紐づければそれ以降はユーザ空間で PKRU レジスタを更新すればアクセス制御が完了できます。この特性を最大限に活かすべく、アクセス制御の度に `mprotect_pkey()` を呼び出さなくて済むよう、PKEY との紐づけは一度だけ実施し、それ以降は PKRU レジスタを書き換えるだけでアクセス権を変更できるようにしています。

```rust
// mprotect で確保したメモリ領域を持つ RegionGuard を生成
// デフォルトアクセス権は ReadWrite. 後に Intel PKU によりアクセス権を制御する
let mut region = RegionGuard::<allocator::Mmap, u32>::new(AccessPermissions::ReadWrite).map_err(RuntimeError::MprotectError)?;
// Intel PKU の Protection Key を生成
let pkey = PkeyGuard::new(PkeyPermissions::NoAccess).map_err(RuntimeError::MprotectError)?;
// RegionGuard と Protection Key を関連付ける
// 以降、RegionGuard のアクセス権は Protection Key により制御される
// 初期状態は NoAccess (-/-). アクセス不可
let mut associated_region = pkey.associate::<PkeyPermissions::NoAccess>(&mut region).map_err(RuntimeError::MprotectError)?;

...

// mutable な参照を取得
// このとき、Protection Key のアクセス権が ReadWrite に変更され、RegionGuard への read/write アクセスが許可される
let write_guard = associated_region.set_access_rights::<PkeyPermissions::ReadWrite>().map_err(RuntimeError::MprotectError)?;

...

// immutable な参照を取得
// 参照を取得
// このとき、Protection Key のアクセス権が ReadOnly に変更され、RegionGuard への read アクセスのみが許可される
let read_guard = associated_region.set_access_rights::<PkeyPermissions::ReadOnly>().map_err(RuntimeError::MprotectError)?;
```

内部的には、`set_access_rights()` で PKRU レジスタを更新します。

```rust
impl<'a, 'p, A: allocator::Allocator<T>, T, Rights> AssociatedRegionHandler<'p, A, T, Rights>
where 
    Rights: access_rights::Access,
{
    ...
    pub fn set_access_rights<NewRights: access_rights::Access>(&'a mut self) -> Result<AssociatedRegion<'a, A, T, NewRights>, super::MprotectError> 
    where
        NewRights: access_rights::Access,
    {
        // Apply new hardware access rights via PKRU
        unsafe {
            self.pkey_guard.pkey.set_access_rights(NewRights::new().value())?;  // ここで PKRU を更新
        }
        ...
    }
}

...

// PKRU レジスタを更新
pub unsafe fn set_access_rights(&self, access: PkeyAccessRights) -> Result<(), super::MprotectError> {
    let pkru_value = pkru::rdpkru();

    let new_pkru_bits = match access {
        PkeyAccessRights::EnableAccessWrite => 0b00,
        PkeyAccessRights::DisableAccess => 0b01,
        PkeyAccessRights::DisableWrite => 0b10,
    } << (self.key * 2);

    let new_pkru_value = pkru_value & !(0b11 << (self.key * 2)) | new_pkru_bits;
    pkru::wrpkru(new_pkru_value);

    Ok(())
}
```

## 入れ子スコープに対応

スコープごとにアクセス制御できるような設計にしています。つまり、現在のアクセス権は現在のスコープ内でのみ有効であり、スコープから出たらアクセス権は無効になります。

```rust
{
    // ここでは read/write 可
    let assoc_rw = assoc_for_mem.set_access_rights::<PkeyPermissions::ReadWrite>().map_err(RuntimeError::MprotectError)?;
    let mut value = assoc_rw.mut_ref_guard().map_err(RuntimeError::PkeyGuardError)?;
    println!("\tValue read via associated region deref(): {}", *value);
    {
        // ここでは read-only
        let assoc2_r = assoc2_for_mem2.set_access_rights::<PkeyPermissions::ReadOnly>().map_err(RuntimeError::MprotectError)?;
        let value = assoc2_r.ref_guard().map_err(RuntimeError::PkeyGuardError)?;
        println!("\tValue read via second associated region deref(): {}", *value);
    }
    // ここから再び read/write 可
    *value = 168;  // ok
}
```

入れ子状態のスコープに対応するために、アクセス権の変遷はメモリ領域ごとにスタックで管理しています。子のスコープから親のスコープに戻ったら、スタックを pull してアクセス権を親スコープの状態に戻すような設計です。

# 今後の展望

現状、`mprotect()` を使う場合と Intel PKU を使う場合とで異なるトレイト、異なる API 体系を利用する形になっています。しかし、実際の利用環境を考えると、必ずしもバイナリの配布先が Intel PKU に対応しているという保証はありません。古い Intel CPU で実行されているかもしれませんし、Intel PKU が無効化された環境かもしれませんし、あるいは AMD や Arm、RISC-V かもしれません。

その場合に、Intel PKU の対応チェックを実施し、対応していなければ `mprotect()` を使いたい、というケースもあるでしょう。こういったユースケースに対応するため、`mprotect()` を使う場合と Intel PKU を使う場合とで統一のインターフェイスを利用できるようにしたいと考えています。

まだまだこのライブラリは未完成です。来年中には一旦完成させてリリースしたいなと考えています。
