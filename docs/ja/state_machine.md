# State Machine

## 目的

状態の変化に応じてコントラクトの振る舞いを変える。

## 動機

いくつかの中間状態にまたがる初期状態から、そのライフサイクルにわたる最終状態への状態遷移が必要なコントラクトについて考えてみましょう。
それぞれの状態において、コントラクトは異なる方法で異なる機能をユーザーに提供するように振る舞います。
こういった振る舞いはオークション、ギャンブル、クラウドファンディングなど、さまざまなユースケースで見られます。[Solidityドキュメント](http://solidity.readthedocs.io/en/v0.4.21/common-patterns.html#state-machine)であっても、一般的なパターンの1つとしてこのパターンを定義しています。

ある状態から別の状態に移行する方法はいくつかあります。ある状態が関数の終わりで終了することもあります。その場合は、指定された時間の経過後に状態が遷移することになります。さまざまな可能性については、「実装」のセクションで詳しく説明します。

上記の機能性を有するパターンは、Gammaらによって既に作成されています(1995)。しかし、ブロックチェーンでのその実装は特に興味深いです。これは、ブロックチェーン自体がトランザクションと組み合わせた初期状態が出力とする新しい状態を持つ状態遷移システムであるためです。ブロックチェーンの状態（トランザクションの前後の状態）と状態の混同を避けるために、これ以降、明示的に定義されるコントラクトにおける状態をステージと呼びます。

## 適用範囲

State Machineパターンを使用できるのは、
* スマートコントラクトがそのライフサイクルにおいていくつかの状態遷移を必要とする場合
* いくつかのステージ間でスマートコントラクトの関数がアクセス可能である場合
* ステージ遷移が明確に定義されるべきであり、すべての参加者にとってそのことが回避可能ではない場合
です。

## 想定するコントラクトの参加者

State Machineパターンには2人の参加者がいます。最初の参加者は、事前定義されたステージを経て遷移し、それぞれのステージで意図された機能のみが呼び出されることを保証するコントラクトの実装です。もう一方の参加者はコントラクトオーナーまたは対話型ユーザーであり、直接または間接的に時間移行を通じてステージ遷移を開始できます。


## 実装

このState Machineパターンの実装では、**ステージ表現**、**関数の対話制御**、**ステージ遷移**の3つのコンポーネントをカバーします。

Solidityのさまざまな段階をモデル化するために、enumを利用することができます。列挙型はユーザー定義のデータ型です。可能性のあるすべてのステージを含む1つのenumが宣言された後、そのenumのインスタンスを使用して現在のステージを格納し、それに新しいステージを割り当てることによって次のステージに移行できます。列挙型はすべての整数型との間で明示的に変換可能であるため、次のステージへの移行はステージインスタンスに整数1を追加することで実現できます。

特定のステージへのアクセスを関数で制限するには、[アクセス制限パターン](./access_restriction.md)を使用して実現できます。関数修飾子は、呼び出された関数を実行する前に、コントラクトステージが必要な段階と等しいかどうかを確認します。関数が不適切なステージで呼び出された場合、トランザクションは[Guard Checkパターン](./guard_check.md)を使用して元に戻されます。

さまざまな状況やニーズに対応するために、あるステージから次のステージに遷移する方法はいくつかあります。一つの方法は関数呼び出しの際に遷移を行うことです。関数がステージ遷移を行うために存在するか、ビジネスロジックを実行します。そしてこのステージ遷移はプロセスの自然な部分です。例えばルーレットの契約では、ハウスはすべての賞金を支払うための関数を呼び出すことができます。そしてそれは `GameEnded`から` WinnersPaid`への段階的な移行で終わります。このような場合、ステージの変更は、新しいステージを状態変数に割り当てることによって、関数の終わりで遷移を開始する修飾子を使用することによって、またはヘルパー関数によって直接実装されます。ヘルパー関数は、呼び出されるたびにステージを1ずつ増分する内部関数です。直接の関数呼び出しに依存しないもう1つの選択肢は、自動時限遷移です。ステージが継続すると想定される期間、またはステージ移行を実行する必要がある将来の時点は、契約に格納されます。関連するすべての関数呼び出しで呼び出される修飾子は、現在のタイムスタンプをチェックし、その時点に既に達している場合には次のステージに移行します。 Solidityの修飾語句の順番が重要であることに言及することは重要です。この知識を念頭に置いて、現在のステージをチェックするmodifierの前に、時間付きトランジションのmodifierを記述して、時間付きステージの潜在的な変更がすでにステージチェックで考慮されるようにする必要があります。

実装が完了したら、意図しない動作から利益を得ようとしたり、単にコントラクトをクラックしたりする可能性がある悪意のあるエンティティによる予期しないステージ変更の可能性を排除するために、厳しいテストが必要です。

## サンプルコード

このサンプルコントラクトはブラインドオークションのステートマシーンを紹介します。これは[Solidityドキュメント](http://solidity.readthedocs.io/en/v0.4.21/common-patterns.html#state-machine)によって提供されているサンプルコードをもとに作られました。
以下では、時間トリガーでの遷移と同様の関数内のステージ遷移を特徴としています。これは拡張可能な使用例であるため、ステートマシンに関連するコードのみが提示されています。入札情報などのstorage変数を含むオークションに関するロジックは省略されてます。

```Solidity
// This code has not been professionally audited, therefore I cannot make any promises about
// safety or correctness. Use at own risk.
contract StateMachine {
    
    enum Stages {
        AcceptingBlindBids,
        RevealBids,
        WinnerDetermined,
        Finished
    }

    Stages public stage = Stages.AcceptingBlindBids;

    uint public creationTime = now;

    modifier atStage(Stages _stage) {
        require(stage == _stage);
        _;
    }
    
    modifier transitionAfter() {
        _;
        nextStage();
    }
    
    modifier timedTransitions() {
        if (stage == Stages.AcceptingBlindBids && now >= creationTime + 6 days) {
            nextStage();
        }
        if (stage == Stages.RevealBids && now >= creationTime + 10 days) {
            nextStage();
        }
        _;
    }

    function bid() public payable timedTransitions atStage(Stages.AcceptingBlindBids) {
        // Implement biding here
    }

    function reveal() public timedTransitions atStage(Stages.RevealBids) {
        // Implement reveal of bids here
    }

    function claimGoods() public timedTransitions atStage(Stages.WinnerDetermined) transitionAfter {
        // Implement handling of goods here
    }

    function cleanup() public atStage(Stages.Finished) {
        // Implement cleanup of auction here
    }
    
    function nextStage() internal {
        stage = Stages(uint(stage) + 1);
    }
}
```
3行目では、オークションが進行する4つの段階を含む列挙型`Stages`が定義されています。状態変数は、10行目の初期ステージで初期化されます。作成時刻は12行目に保管され、時限遷移にとって重要になります。 14行目で定義されている関数修飾子は、現在のステージと関数の許可ステージの対比です。パラメータとして提供されているステージは、ファンクションロジックを実行できるようにするために契約が入っている必要があるステージです。関数が修飾子`transitionAfter()`を実装する場合、内部メソッド`nextStage()`が関数の終わりで呼び出され、コントラクトは次の段階で遷移します。個々のif文は、現在の時刻と移行が起こると考えられる時刻（例えば `creationTime + 6日 `）とを比較することによって、現在のステージも考慮に入れながら、契約がすでに次の段階にあるべきかどうかをチェックします。

34行目から始まる4つのパブリックメソッドは、それぞれの該当するステージ内でのみ呼び出すことができます。これは `atStage()`修飾子によって実現されます。最初の2つの段階への移行はタイムリーです。 `timedTransitions`は最初の2つだけではなく、最初の3つの関数にも含まれていることに注意してください。これは、次の段階の関数を呼び出すときに実際の遷移が起こっているためです。例：契約の作成から8日後に`bid()`関数を呼び出すと、最初に次の段階に移行します。なぜなら、 `timedTransitions`修飾子が起動するからです。しかしその後、 `atStage()`修飾子はステージがもう一致していないことを検出し、ステージ遷移を含むトランザクション全体を変換します。この場合、 `timedTransitions`修飾子は、6日が経過した後に`bid()`を呼び出せないことを確認しています。次の段階の関数、この場合は `reveal()`が呼び出されるのはこれが初めてです。この段階が正しいと認識され、トランザクションが元に戻されないことだけが、プロセスは前と同じです。この複雑な振る舞いが、「実装」セクションに記載されている修飾子の順序が非常に重要な理由です。

ステージ3からステージ4への移行は、42行目の関数で使用されている`transitionAfter`修飾子で行われます。関数の実行後、コントラクトは自動的に次の最後のステージに入ります。これは`cleanup()`のみを許可します。呼び出されるメソッド


## 結論

State Machineパターンを適用した結果として、コントラクトの動作を別々の段階に分割することができます。意図した時間にだけ関数を呼び出せるようにします。さらに、このパターンは、さまざまな段階でコントラクトを導き、状況に応じて使用できる状態遷移を開始するためのオプションを提供します。
ステージ遷移の際にいくつかのことを気に留めて置かなければなりません。時限遷移にはすべての参加者にとって明確な方針があるという利点がありますが、ブロック番号またはタイムスタンプの使用は完全に無害ではありません。マイナーはある程度タイムスタンプに影響を与える可能性があります。したがって、非常に時間に敏感な場合には、時限遷移を使用しないでください。完全に安全であるためには、コントラクトは実際の時刻から最大900秒ずれているタイムスタンプに対して頑強であるべきです。しかし、オーナー自身が状態を変更するか、もしくはコントラクトを放棄して投資されたすべての資金を凍結することを単純に決定できることなど、コントラクトオーナーが手動でで状態遷移を実行できることに注意が必要です。


## 既知の使われ方

何らかの形でこのパターンを適用するいくつかのコントラクトを見つけることができます。 例の1つは賭けを行うコントラクトである[Ethorse](https://github.com/ethorse/ethorse-core/blob/master/contracts/Betting.sol)です。これは暗号通貨の価格開発に対する賭けを可能にします。この例では、ステージはブール値の形式で構造体に格納され、現在のステージは "true"に設定されています。遷移はタイムスタンプを介してタイムリーに行われます。
ほとんど手動による移行に依存している別の実装は、コミュニティ主導の市場エコシステムである[Pocketinns](https://github.com/pocketinns/PocketinnsContracts/blob/master/DutchAuction.sol)のオークションコントラクトです。このコントラクトでは、コントラクトオーナーは自分の意思でステージを変更することができます。たとえコントラクトの中で緊急措置として説明されているとしても、そういった操作を行うことが可能です。そのようなコントラクトとの取引はどんなものであっても注意して行うべきでしょう。

[**< Back**](https://github.com/shunMB/solidity-patterns-ja/)