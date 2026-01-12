---
title: "【C#】Facade（ファサード）パターン入門 - 複雑なシステムをシンプルに使う方法"
emoji: "🏛️"
type: "tech"
topics: ["csharp", "デザインパターン", "dotnet", "オブジェクト指向", "初心者向け"]
published: true
---

## はじめに

Facade（ファサード）パターンは、**複雑なサブシステムのインターフェースを単純化する**デザインパターンです。

:::message
**Facade（ファサード）** とは、フランス語で「建物の正面」を意味します。
建物の正面玄関のように、**複雑な内部構造を隠して、シンプルな入り口を提供する**のがこのパターンの役割です。
:::

### このパターンで解決できること

- クライアントがサブシステムの複雑さを意識せずに利用できる
- システムの使用が容易になる
- システムの依存関係が減少する

### 対象読者

- デザインパターンを学び始めた方
- C#でオブジェクト指向設計を学んでいる方
- 複雑なシステムを整理したい方

---

## Facadeパターンの基本構成

Facadeパターンは、以下の2つの要素で構成されます。

### 1. Facade（ファサード）

- クライアントからのリクエストを**適切なサブシステムのオブジェクトにルーティング**する
- クライアントはこのFacadeを通じてサブシステムとやり取りする
- **複雑な処理の「窓口」** となる役割

### 2. Subsystems（サブシステム）

- Facadeの背後にある**複雑な機能を持つクラス群**
- Facadeがこれらのクラスの複雑さをクライアントから隠蔽する
- 各サブシステムは自身の役割に専念できる

:::details 例え話で理解する
**銀行の窓口** をイメージしてください。

- **Facade** = 銀行の窓口担当者
- **Subsystems** = 口座管理部門、ローン部門、投資部門など

お客様（クライアント）は窓口担当者に依頼するだけで、裏側の複雑な部門間のやり取りを意識する必要がありません。
:::

---

## C#での実装例

以下は、FacadeパターンをC#で実装した簡単な例です。

### サブシステムの定義

```csharp
// サブシステム A
public class SubsystemA
{
    public string OperationA1() => "Subsystem A, Method A1\n";
    public string OperationA2() => "Subsystem A, Method A2\n";
}

// サブシステム B
public class SubsystemB
{
    public string OperationB1() => "Subsystem B, Method B1\n";
    public string OperationB2() => "Subsystem B, Method B2\n";
}
```

### Facadeクラスの定義

```csharp
// Facadeクラス
public class Facade
{
    protected SubsystemA _subsystemA;
    protected SubsystemB _subsystemB;

    public Facade(SubsystemA subsystemA, SubsystemB subsystemB)
    {
        this._subsystemA = subsystemA ?? throw new ArgumentNullException(nameof(subsystemA));
        this._subsystemB = subsystemB ?? throw new ArgumentNullException(nameof(subsystemB));
    }

    /// <summary>
    /// サブシステムのメソッドを簡略化して提供するFacadeメソッド
    /// </summary>
    public string Operation()
    {
        string result = "Facade initializes subsystems:\n";
        result += _subsystemA.OperationA1();
        result += _subsystemB.OperationB1();
        result += "Facade orders subsystems to perform the action:\n";
        result += _subsystemA.OperationA2();
        result += _subsystemB.OperationB2();
        return result;
    }
}
```

### クライアントコード

```csharp
class Program
{
    static void Main(string[] args)
    {
        // サブシステムを作成
        SubsystemA subsystemA = new SubsystemA();
        SubsystemB subsystemB = new SubsystemB();

        // Facadeを作成
        Facade facade = new Facade(subsystemA, subsystemB);

        // クライアントはサブシステムと直接やり取りせず、
        // Facadeを通じて操作を実行
        Console.WriteLine(facade.Operation());
    }
}
```

### 実行結果

```
Facade initializes subsystems:
Subsystem A, Method A1
Subsystem B, Method B1
Facade orders subsystems to perform the action:
Subsystem A, Method A2
Subsystem B, Method B2
```

---

## ポイント解説

### なぜFacadeパターンを使うのか？

| 観点 | Facadeパターンなし | Facadeパターンあり |
|------|-------------------|-------------------|
| クライアントの複雑さ | 全サブシステムの仕様を理解する必要がある | Facadeのメソッドだけ知っていればOK |
| 依存関係 | クライアントが多数のクラスに依存 | Facadeクラスのみに依存 |
| 保守性 | サブシステム変更時、クライアントも修正が必要 | Facade内で吸収可能 |

### Facadeパターンの特徴

1. **Facadeクラスは複雑な実装を持たない**
   - あくまでサブシステム内部に仕事を投げるだけ

2. **サブシステムの直接使用を妨げない**
   - 必要であれば、サブシステムの機能を直接利用することも可能

3. **サブシステムはFacadeを意識しない**
   - サブシステムからFacadeを呼び出すことはない

---

## 活用シーン

Facadeパターンは、以下のような場面で特に有効です。

- **既存のシステムをリファクタリングする際**
  - 複雑になったシステムを整理して、使いやすいAPIを提供

- **複数のライブラリやフレームワークを統合する際**
  - 外部ライブラリの複雑な初期化処理をまとめる

- **レガシーコードとの橋渡し**
  - 古いシステムに対して、新しいインターフェースを提供

---

## まとめ

- Facadeパターンは、**複雑なサブシステムへのシンプルな窓口を提供する**パターン
- クライアントはサブシステムの内部構造や複雑さを意識せずに操作可能
- システムの構造がより清潔で、理解しやすく、保守しやすくなる

:::message alert
**注意点**
Facadeに機能を詰め込みすぎると、Facade自体が「神クラス（God Class）」になってしまう可能性があります。適切な粒度で設計することが重要です。
:::

---

## 参考資料

- [Refactoring.Guru - Facade パターン](https://refactoring.guru/ja/design-patterns/facade)
- [Wikipedia - Facade パターン](https://ja.wikipedia.org/wiki/Facade_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)
