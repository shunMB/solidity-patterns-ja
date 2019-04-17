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

In line 3 the enum `Stages`, which contains the four stages the auction will be going through, is defined. A state variable is initialized with the initial stage in line 10. The time of creation is stored in line 12 and will be important for the timed transitions. The function modifier defined in line 14 is checking the current stage versus the allowed stage of a function. The stage provided as a parameter is the stage that the contract has to be in, in order to be able to execute the function logic. If a function implements the modifier `transitionAfter()`, the internal method `nextStage()` is called at the end of the function and the contract transitions in the next stage. Timed transitions are handled by using the modifier specified in line 24. The individual if-clauses check if the contract should already be in the next stage by comparing the current time with the time the transition is supposed to happen (e.g. `creationTime + 6 days`), while also taking the current stage into account.

The four public methods starting from line 34 are only callable in their respective stages, which is achieved by the `atStage()` modifier provided with the concrete stage. Transitions for the first two stages are timely. Notice that the `timedTransitions` modifier is included in the first three functions and not only in the first two. This is because the actual transition is happening when calling the function of the next stage. For example: calling the `bid()` function 8 days after contract creation will first transition to the next stage, because the `timedTransitions` modifier triggers. Afterwards, however, the `atStage()` modifier will detect that the stage is not matching anymore and will revert the whole transaction, including the stage transition. In this case the `timedTransitions` modifier is making sure that `bid()` cannot be called after 6 days have passed. The persisting stage transition is happening the first time the function of the next stage is called, in this case `reveal()`. The process is the same as before only that this time the stage is recognized as correct and the transaction is not reverted. This complex behavior is why the order of the modifiers mentioned in the Implementation section is so important.

The transition from stage three to four is done with the `transitionAfter` modifier used in the function in line 42. After the execution of the function, the contract automatically goes into the next and final stage, which only allows the `cleanup()` method to be called.

## 結論

One consequence of applying the State Machine Pattern is the partitioning of contract behavior into the distinct stages. It allows functions to be only called at the intended times. Additionally the pattern guides the contract through its different stages by providing several options for initiating stage transitions that can be used depending on the context.

Some consequences that should be kept in mind, emerge from the different options for stage transitioning. While timed transitions have the benefit of a clear policy for every participant, the usage of block numbers or timestamps is not entirely harmless. Miners have the potential ability to influence timestamps to a certain degree. It should therefore be avoided to use timed transitions for very time sensitive cases. To be completely safe, a contract should be robust against a timestamp deviating by up to 900 seconds from the actual time. But also the manual transition of stages by the contract owner is prone to manipulation, as the owner can simply decide to change stages in his favor or abandon the contract and freeze all invested funds.      

## 既知の使われ方

Several contracts applying this pattern in some form or another could be observed. One example is the betting contract from [Ethorse](https://github.com/ethorse/ethorse-core/blob/master/contracts/Betting.sol), which allows bets on the price development of cryptocurrencies. In this example, the stages are stored in a struct in the form of booleans, with the current stage being set to `true`. Transitions are made in a timely fashion via timestamps. A different implementation that relies mostly on manual transitions is the auction contract of [Pocketinns](https://github.com/pocketinns/PocketinnsContracts/blob/master/DutchAuction.sol), a community driven marketplace ecosystem. In this contract the owner has the ability to change stages at his own will. Even though it is described as an emergency measure in the contract, it opens the door for severe manipulation and any transaction with such a contract should be done with care. 

[**< Back**](https://fravoll.github.io/solidity-patterns/)
