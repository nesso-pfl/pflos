## Standalone 
- Rust で書く理由
  - パターンマッチや所有権などの強力なシステムが使える
  - C や C++ にありがちな [undefined behavior](https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs) や [メモリ安全](https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention) の問題に悩まされずに開発できる
- `#![no_std]` を記述した
  - OS の機能に依存しないベアメタルなプログラムを書く必要があるため
- Language items を知った
  - コンパイラの内部に必要な関数や型のこと
  - `#[lang = "item"]` の構文でコンパイラに情報を提供する
- `eh_personality` は、[stack unwinding](https://www.bogotobogo.com/cplusplus/stackunwinding.php) を実装する language item
  - 複雑なので今回は使わないで行く
  - つまり panic 時にスタックの巻き戻しを行わず、即座にプログラムを終了する
- `#![no_main]` で、プログラムの entry point を書き換え可能にする
  - 通常 rust プログラムは main 関数の実行前に `crt0` という C のプログラムが entry point になるが、それを無効化している
  - entry point: 最初に実行されるプログラム
  - ほとんどの OS では、 `_start` 関数が entry point になる
- `#[no_mangle]` で、 `hoge` 関数がコンパイル時に命名が暗号化されるのを防ぐ
  - linker に entry point を伝えるときに名前が変わっていると困るので

### Boot Process
1. マザーボード ROM （特別なフラッシュメモリ） 内のファームウェアを実行
  - power-on self-test を実行する
    - 利用可能な RAM を指定する
    - CPU やハードウェアの事前初期化を行う
1. ファームウェアはブート可能なディスクを探す
1. 見つかったら処理をブートローダに移譲する
1. ブートローダがディスク上のカーネルイメージを見つけ出し、メモリ上に展開する
1. ブートローダが memory map を BIOS から見つけ出し、OS カーネルに渡す

### ファームウェアの種類
- x86 には BIOS と UEFI の２つのファームウェア規格がある
- BIOS はシンプルで古い x86 マシンをサポートしているが、デフォルトでリアルモードなので必要に応じてモードを切り替えてやる必要がある
- UEFI は比較的新しく機能が豊富だが、セットアップが複雑だと言われることがある

### x86 アーキテクチャの動作モード
- x86 ではいくつかの動作モードがある
  - real mode: 16-bit 互換
  - protected mode: 32-bit 互換
  - long mode: 64-bit 互換


## Target system の設定
- [rustc](https://doc.rust-lang.org/nightly/rustc/what-is-rustc.html) のドキュメントが参考になる
- 今回、 `target-config.json` に書いていった
- `data-layout`: integer, floating point, pointer 型のサイズを定義
- `pre-link-args`: Linker に渡すパラメータを指定
- `panic-storategy`: `abort` を指定し、panic 時はプログラムを即座に終了するようにしている
  - Cargo.toml の `panic = "abort"` を包括する
  - core ライブラリをリコンパイルする際にも効果を発揮する
- `disable-redzone`: スタック最適化をしてくれるが、interruption を安全に行いたいので無効化する
  - 例外処理などを行う際に stack pointer が移動し red zone を上書きする、この時に関数の一時変数が消えて困ることになるんだと思う（多分）
  - 通常の kernel は例外処理を別の stack で行うため、 red zone があっても問題にならない
- `features`: `-mmx,-sse,+soft-float` を指定した
  - `-` で機能を無効化し、 `+` で機能を有効化している
  - `mmx` `sse` は SIMD のサポートを決定する
  - x86_64 の floating point 操作はデフォルトで SIMD を必要とするので、回避策として `soft-float` を追加する

## kernel のビルド
- 自作の target system では、core を自作しないといけない
  - サポートされてるものは既に core が利用可能
- Cargo config の `build-std` で core を含む標準ライブラリのリコンパイルが可能になる
  - `unstable` マークをすることで、 nightly build でのみ利用することを指定する
- また、 `build-std-features` に `compiler-builtins-mem` を指定する
  - rust でデフォルトで無効なメモリ操作の関数を有効化する
  - これは普通 C ライブラリから提供されるものだが、今回 C は使わないので

## kernel の起動
1. kernel をコンパイルする
1. bootable disk に入れる
1. linker 使って kernel を bootloader に link する
1. ハードウェア上で kernel を起動する

を、するために...

- bootloader を実装する代わりに、 `bootloader` crate を利用する
- コンパイル後の kernel と bootloader を link するために `bootimage` crate を利用する
- `.cargo/config.toml` に `target.cfg` を追加
  - OS の指定と、 `cargo run` したときに実行されるコマンドを指定する
