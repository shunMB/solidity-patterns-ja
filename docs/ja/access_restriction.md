# Access Restriction

## 目的

適切な基準に従ってコントラクトの機能へのアクセスを制限する。

## 動機

ブロックチェーンは一般に公開されているため、コントラクトに対して完全なプライバシーを保証することはできません。誰もがブロックチェーンからあなたのコントラクトの状態を読み取ることを防ぐことはできないでしょう。できることは、他のコントラクトによるあなたのコントラクトの状態への読み取りアクセスを制限することです。これはあなたの状態変数を `private`として宣言することによって達成されます。関数を `private`と宣言することもできますが、そうすることで、契約範囲外の誰もがいかなる状況下でもそれを呼び出すことを妨げるでしょう。しかし、単にそれらを`public`と宣言するだけで、ネットワーク内のすべての参加者にアクセスできるようになります。ほとんどの場合、特定の規則が満たされる場合には機能へのアクセスを許可することが望まれます。多くの場合、アクセスは、コントラクトオーナーのように、定義されたエンティティのセットに制限されることになっています。他の制限は、特定の時点でのアクセス、またはアクセスしているエンティティがアクセスに対して料金を支払う意思がある場合にのみ許可するべきです。これらすべての制限、およびその他の制限は、アクセス制限パターンを実装することで実現できるため、スマートコントラクトへの不正アクセスに対するセキュリティを確保できます。

## 適用範囲

Access Restrictionパターンを使用できるのは、
* 特定の状況下のみでしか関数を呼び出しできないようにする場合
* いくつかの関数に似たような制限を適用させたい場合
* 不正アクセスに対するスマートコントラクトの安全性を高めたい場合
です。

## 想定するコントラクトの参加者

このパターンの参加者は、制限される機能の呼び出し元、および機能が属するコントラクトです。呼び出し側エンティティは、ユーザーまたは別のコントラクトのいずれかであり、それぞれのコントラクトアドレスにトランザクションを送信することによって機能を呼び出します。呼び出されたコントラクトに関与するアクターは、制限された機能と実際のアクセス制御を担当する追加のコンポーネントです。

## 実装

Access Restrictionパターンの実装には、[Guard Checkパターン](./guard_check.md)を使用しています。Guard Checkパターンによって提供される機能により、関数が呼び出されると必要な状況をチェックし、それらが満たされない場合は例外をスローします。チェックは対応する関数の先頭にも配置できると主張するかもしれません。ただし、これらのチェックは複数の機能に再利用されることが多いため、ジョブを修飾子に外部委譲することをお勧めします。これは、それらを必要とする機能に適用できます。修飾子はそれぞれの関数の入力パラメータから引数を取ることができ、それら自身の引数を与えられるか、あるいは本体にハードコーディングされた条件を持つことができ、それは再利用性を制限します。

これらの修飾子の構造は通常同じパターンに従います。初めに必要な条件がチェックされます。その後、実行は最初の関数に戻ります。この振る舞いは、修飾子のコード内のアンダースコア（ `_;`）によって示されます。関数の後に実行する必要がある追加の動作がある場合は、修飾子のアンダースコアの後に追加のコードを挿入することができます。このパターン、特に修飾子の先頭の条件チェックは、さまざまな方法でさまざまなアクセス制限を適用することができます。

## サンプルコード

次のコードは、コントラクトオーナーによるサンプルコントラクトに対する3種類のアクセス制限を特徴とします。[Solidityのドキュメントの例](http://solidity.readthedocs.io/en/v0.4.21/common-patterns.html#restricting-access)を参考にしています。コントラクトオーナーは、オーナーが最後に変更されてから1か月が経過した後で、自身のオーナー権限を譲渡するか、もしくは誰でも1イーサリアムで購入することができます。

```Solidity
// This code has not been professionally audited, therefore I cannot make any promises about
// safety or correctness. Use at own risk.
pragma solidity ^0.4.21;

contract AccessRestriction {

    address public owner = msg.sender;
    uint public lastOwnerChange = now;
    
    modifier onlyBy(address _account) {
        require(msg.sender == _account);
        _;
    }
    
    modifier onlyAfter(uint _time) {
        require(now >= _time);
        _;
    }
    
    modifier costs(uint _amount) {
        require(msg.value >= _amount);
        _;
        if (msg.value > _amount) {
            msg.sender.transfer(msg.value - _amount);
        }
    }
    
    function changeOwner(address _newOwner) public onlyBy(owner) {
        owner = _newOwner;
    }
    
    function buyContract() public payable onlyAfter(lastOwnerChange + 4 weeks) costs(1 ether) {
        owner = msg.sender;
        lastOwnerChange = now;
    }
}
```
5-6行目では、状態変数 `owner`と` lastOwnerChange`はコントラクトの作成時にコントラクトの作成者と現在のタイムスタンプで初期化されています。 8行目の最初の修飾子 `onlyBy(address _account)`は26行目の `changeOwner(..)`関数に結び付けられていて、その関数の起動側（ `msg.sender`）が与えられた変数と等しいことを確認しますこの場合は `owner`です。この修飾子を使用すると、現在の所有者以外の誰かによって保護された関数が呼び出されるたびに、例外がスローされます。

In line 5-6 the state variables `owner` and `lastOwnerChange` are initialized at contract creation time with the creator of the contract and the current timestamp. The first modifier `onlyBy(address _account)` in line 8 is attached to the `changeOwner(..)` function from line 26 and makes sure that the initiator of the function (`msg.sender`) is equal to the variable provided in the modifier call, which in this case is `owner`. The usage of this modifier leads to an exception being thrown, every time the guarded function is called by anybody besides the current owner.

The second modifier `onlyAfter(uint _time)` works in the same way, with the difference that it throws if the function it is attached to is called before the specified time. It is used in line 30 and is provided with the time of the last change of ownership plus four weeks (Four weeks are added instead of one month because Solidity does not support months as a unit of time.). Therefore, guaranteeing that the function call can only be successful after at least four weeks have passed since the last change.

The third and last modifier `costs(uint _amount)` in line 18 takes an amount of currency as an input and makes sure that the value provided with the calling transaction is at least as high as the specified amount, before jumping into the execution of the guarded function. This modifier differs from the other two, as it has code implemented after the function execution, which is triggered by the underscore in line 20. The additional if-clause starting in line 21, checks if more money than necessary was provided in the transaction and transfers the surplus amount back to the sender. This is a good example for the various possibilities modifiers can provide. Combined with the previous modifier the contract can only be bought by a new owner, after four weeks have passed since the last change in ownership, and if the transaction contains at least one ether. The `payable` modifier of the `buyContract()` function in line 30 is needed in order to be able to receive money with a transaction.

It should be noted that in this simple example, we could have omitted the modifiers and implement their code directly in the respective function bodies, without loosing any functionality and even reducing complexity. The benefit of outsourcing the functionality into modifiers becomes apparent, as soon as two or more functions share the same, or similar, requirements (like in the [State Machine pattern](./state_machine.md)), as the modifier allows for easy reusability.

## 結論

Several consequences have to be taken into account when applying the Access Restriction pattern. One controversial point is the readability of code. On the one hand, modifiers can make the code easier to understand, because the restriction criterion is clearly recognizable in the function header, especially if the modifiers are given meaningful names like in the provided sample code. On the other hand, execution flow is jumping from one line in the code to a completely different one, which makes it harder to follow and audit the code, and therefore simplifies the introduction of misleading (malicious) code. Because of this reason, the new smart contract programming language [Vyper](https://viper.readthedocs.io/en/latest/) is giving up on modifiers.

The advantages of the pattern are drawn from the fact that it is easy to adapt to different situations and highly reusable, while still providing a secure way to limit the access to functionality and therefore increase smart contract security altogether.
 
## 既知の使われ方
The most prominent example of this pattern is probably the [Ownable contract by OpenZeppelin](https://github.com/OpenZeppelin/zeppelin-solidity/blob/master/contracts/ownership/Ownable.sol).

Another example is the [core contract of the CrytoKitties DApp](https://etherscan.io/address/0x06012c8cf97bead5deae237070f9587f8e7a266d\#code), where there is not only one owner, but three. Namely the CEO, CFO and COO, who have different security levels and therefore different functions they are allowed to access.   

[**< Back**](https://fravoll.github.io/solidity-patterns/)
