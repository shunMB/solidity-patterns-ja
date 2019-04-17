# Solidityでのデザインパターン実装

このリポジトリはスマートコントラクト言語 solidity 0.4.20 のデザインパターン実装を試したものです。
新しいバージョンではいくつかの機能が変更している可能性があることに注意してください。
また、それぞれのパターンはコードサンプルと詳しい説明、その背景、それ以外の追加の情報などで構成されます。

ドキュメントはこちら: https://fravoll.github.io/solidity-patterns/

## デザインパターン

* **振る舞いに関するパターン**
  * [**Guard Check**](docs/guard_check.md): スマートコントラクトの振る舞いと入力パラメータを保証する。
  * [**State Machine**](docs/state_machine.md): 状態の変化に応じてコントラクトの振る舞いを変える。
  * [**Oracle**](docs/oracle.md): ブロックチェーン外のデータにアクセスする。
  * [**Randomness**](docs/randomness.md): ブロックチェーンの決定論的環境で定義された間隔の乱数を生成する。
* **セキュリティに関するパターン**
  * [**Access Restriction**](docs/access_restriction.md): 適切な基準に従ってコントラクトの機能へのアクセスを制限する。
  * [**Checks Effects Interactions**](docs/checks_effects_interactions.md): 外部コール後のハイジャックを狙う悪意あるコントラクトの攻撃領域を狭める。
  * [**Secure Ether Transfer**](docs/secure_ether_transfer.md): コントラクトから他アドレスへの安全なイーサリアムの送金
  * [**Pull over Push**](docs/pull_over_push.md): ユーザーへのイーサリアムの転送に関連するリスクを軽減する。
* **アップグレーダビリティに関するパターン**
  * [**Proxy Delegate**](docs/proxy_delegate.md): 依存関係を壊さずにスマートコントラクトをアップグレードする。
  * [**Eternal Storage**](docs/eternal_storage.md): スマートコントラクトのアップグレード後においてもコントラクトストレージを保持する。
* **経済性に関するパターン**
  * [**String Equality Comparison**](docs/string_equality_comparison.md): 多数の異なる入力に対して平均ガス消費量が最小になるように、2つの文字列が等しいことを確認する。
  * [**Tight Variable Packing**](docs/tight_variable_packing.md): 静的なサイズの変数を格納またはロードするときにガス消費量を最適化する。
  * [**Memory Array Building**](docs/memory_array_building.md): コントラクトのストレージ変数からデータを集約し、ガス効率の良い方法で取得する。
  

## 免責事項

ここで紹介されているすべてのパターンは開発途中です。そのため、自己責任にて利用してください。このデザインパターンにより何らかの被害を被ったとしてもこちらでは対応いたしかねます。
