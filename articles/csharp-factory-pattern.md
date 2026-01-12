---
title: "【C#】Factoryパターン入門 - オブジェクト生成をスマートに管理する"
emoji: "🏭"
type: "tech"
topics: ["csharp", "designpattern", "oop", "初心者向け"]
published: true
---

## はじめに

この記事では、オブジェクト指向プログラミングにおける代表的なデザインパターンの一つである**Factoryパターン**について解説します。

### この記事で学べること

- Factoryパターンの基本概念
- どんな場面で使うと効果的か
- C#での実装方法

### 対象読者

- C#の基本文法を理解している方
- デザインパターンを学び始めた方
- コードの保守性を向上させたい方

## Factoryパターンとは？

Factoryパターンは、**オブジェクトの作成ロジックを専用の「Factory」クラスにカプセル化**するデザインパターンです。

### 例え話で理解する

工場（Factory）を想像してみてください。

- お客さん（クライアントコード）は「車が欲しい」と注文するだけ
- 工場（Factoryクラス）が部品を組み立てて車を作る
- お客さんは車の作り方を知らなくても、完成品を受け取れる

つまり、**「作り方」と「使い方」を分離**できるのがFactoryパターンの特徴です。

## Factoryパターンが有効な場面

以下のような場合に特に効果を発揮します。

1. **オブジェクトの作成プロセスが複雑な場合**
   - 初期化に多くの手順が必要
   - 依存関係の設定が複雑

2. **クライアントが具体的なクラスに依存すべきでない場合**
   - インターフェースや抽象クラスに依存させたい
   - 実装の詳細を隠蔽したい

3. **生成するオブジェクトの型が実行時に決まる場合**
   - ユーザーの入力によって作成するオブジェクトが変わる
   - 設定ファイルの内容で動的に切り替えたい

## C#での実装例

以下に、動物を生成するシンプルなFactoryパターンの例を示します。

### 1. インターフェースの定義

まず、生成されるオブジェクトの共通インターフェースを定義します。

```csharp
public interface IAnimal
{
    void Speak();
}
```

### 2. 具体的なクラスの実装

インターフェースを実装した具体的なクラスを作成します。

```csharp
public class Dog : IAnimal
{
    public void Speak()
    {
        Console.WriteLine("The dog says: Woof!");
    }
}

public class Cat : IAnimal
{
    public void Speak()
    {
        Console.WriteLine("The cat says: Meow!");
    }
}
```

### 3. Factoryクラスの作成

オブジェクト生成を担当するFactoryクラスを作成します。

```csharp
public static class AnimalFactory
{
    public static IAnimal CreateAnimal(string type)
    {
        switch (type)
        {
            case "Dog":
                return new Dog();
            case "Cat":
                return new Cat();
            default:
                throw new ArgumentException("Invalid type", nameof(type));
        }
    }
}
```

### 4. 使用例

クライアントコードからの使用方法です。

```csharp
// Factoryを使ってオブジェクトを生成
IAnimal dog = AnimalFactory.CreateAnimal("Dog");
IAnimal cat = AnimalFactory.CreateAnimal("Cat");

// 使用
dog.Speak();  // 出力: The dog says: Woof!
cat.Speak();  // 出力: The cat says: Meow!
```

## ポイント解説

### なぜFactoryを使うのか？

Factoryを使わない場合と比較してみましょう。

**Factoryを使わない場合:**

```csharp
// クライアントが具体的なクラスを直接インスタンス化
IAnimal dog = new Dog();
IAnimal cat = new Cat();
```

**Factoryを使う場合:**

```csharp
// Factoryを通じてオブジェクトを取得
IAnimal dog = AnimalFactory.CreateAnimal("Dog");
IAnimal cat = AnimalFactory.CreateAnimal("Cat");
```

### Factoryを使うメリット

| メリット | 説明 |
|---------|------|
| **疎結合** | クライアントは具体的なクラスを知らなくてよい |
| **変更に強い** | 新しい動物を追加してもクライアントコードの変更が不要 |
| **一元管理** | オブジェクト生成のロジックが一箇所にまとまる |
| **テストしやすい** | モックオブジェクトへの差し替えが容易 |

## 実装時の注意点

:::message
**Factoryクラスの配置場所**

Factoryクラスは、通常**インターフェースや抽象クラスと同じ場所（名前空間）** に配置することが推奨されます。これにより、関連するコードがまとまり、見通しが良くなります。
:::

## まとめ

- Factoryパターンは**オブジェクト生成のロジックをカプセル化**するデザインパターン
- クライアントコードと具体的なクラスの**依存関係を減らせる**
- **実行時に生成するオブジェクトを切り替えたい場合**に特に有効
- C#では`switch`文や辞書を使って実装するのが一般的

### 次のステップ

Factoryパターンを理解したら、以下のパターンも学んでみましょう。

- **Abstract Factory パターン**: 関連するオブジェクト群を生成
- **Builder パターン**: 複雑なオブジェクトを段階的に構築
- **Singleton パターン**: インスタンスを1つに制限

## 参考資料

- [Refactoring Guru - Factory Method](https://refactoring.guru/design-patterns/factory-method)
- [Microsoft Docs - デザインパターン](https://docs.microsoft.com/ja-jp/dotnet/standard/design-guidelines/)
