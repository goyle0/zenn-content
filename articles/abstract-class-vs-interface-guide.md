---
title: "【C#/Java】抽象クラスとインターフェースの使い分けガイド（2025年版）"
emoji: "🎯"
type: "tech"
topics: ["csharp", "java", "oop", "設計", "初心者向け"]
published: true
---

## はじめに

オブジェクト指向プログラミングにおいて、「抽象クラス」と「インターフェース」のどちらを使うべきか迷うことはありませんか？

この記事では、C#とJava両方の視点から、それぞれの特徴と使い分けのガイドラインを解説します。

### 対象読者

- オブジェクト指向の基本を学んでいる方
- 抽象クラスとインターフェースの違いがわかりにくいと感じている方
- 設計時にどちらを選ぶべきか悩んでいる方

## 抽象クラスとインターフェースの基本的な違い

### 比較表

| 項目 | 抽象クラス | インターフェース |
|------|-----------|-----------------|
| 継承/実装の数 | 1つのみ継承可能 | 複数実装可能 |
| 状態（フィールド） | 持てる | 持てない |
| コンストラクタ | 持てる | 持てない |
| アクセス修飾子 | 自由に設定可能 | 基本的にpublic |
| デフォルト実装 | 可能 | 可能（C# 8.0+ / Java 8+） |

:::message
**ポイント**: C# 8.0以降、Java 8以降では、インターフェースでもデフォルト実装が可能になりました。ただし、状態（フィールド）を持てないという根本的な違いは変わりません。
:::

## 使い分けのガイドライン

### 1. 共有する「実装」がある場合 → 抽象クラス

複数のクラスで共通の処理（具体的なコード）を共有したい場合は、**抽象クラス**が適しています。

```csharp
// C#の例
public abstract class Animal
{
    // 共通の状態
    protected string Name { get; set; }
    
    // 共通の実装
    public void Sleep()
    {
        Console.WriteLine($"{Name}は眠っています...");
    }
    
    // 派生クラスで実装を強制
    public abstract void MakeSound();
}

public class Dog : Animal
{
    public Dog(string name) => Name = name;
    
    public override void MakeSound()
    {
        Console.WriteLine("ワンワン！");
    }
}
```

```java
// Javaの例
public abstract class Animal {
    protected String name;
    
    public void sleep() {
        System.out.println(name + "は眠っています...");
    }
    
    public abstract void makeSound();
}

public class Dog extends Animal {
    public Dog(String name) {
        this.name = name;
    }
    
    @Override
    public void makeSound() {
        System.out.println("ワンワン！");
    }
}
```

:::message
**例え話**: 抽象クラスは「設計図＋部品」のようなもの。共通部品（実装）を最初から持っているので、派生クラスはそれを使いながら、足りない部分だけを追加すればOKです。
:::

### 2. 共有する「振る舞いの契約」だけがある場合 → インターフェース

「何ができるか」だけを定義し、「どうやるか」は実装クラスに任せたい場合は、**インターフェース**が適しています。

```csharp
// C#の例
public interface IMovable
{
    void Move(int x, int y);
}

public interface IAttackable
{
    void Attack(string target);
}

// 複数のインターフェースを実装可能
public class Player : IMovable, IAttackable
{
    public void Move(int x, int y)
    {
        Console.WriteLine($"({x}, {y})に移動");
    }
    
    public void Attack(string target)
    {
        Console.WriteLine($"{target}を攻撃！");
    }
}
```

```java
// Javaの例
public interface Movable {
    void move(int x, int y);
}

public interface Attackable {
    void attack(String target);
}

public class Player implements Movable, Attackable {
    @Override
    public void move(int x, int y) {
        System.out.println("(" + x + ", " + y + ")に移動");
    }
    
    @Override
    public void attack(String target) {
        System.out.println(target + "を攻撃！");
    }
}
```

:::message
**例え話**: インターフェースは「資格」のようなもの。「運転免許」を持っていれば車を運転できる契約があるだけで、どんな車に乗るかは本人次第です。
:::

### 3. 複数の継承が必要な場合 → インターフェース

C#もJavaも、クラスは**1つしか継承できません**（単一継承）。しかし、インターフェースは**複数実装できます**。

```csharp
// C#の例：複数のインターフェースを実装
public class Duck : Animal, ISwimmable, IFlyable
{
    public void Swim() => Console.WriteLine("泳いでいます");
    public void Fly() => Console.WriteLine("飛んでいます");
    public override void MakeSound() => Console.WriteLine("ガーガー！");
}
```

```java
// Javaの例
public class Duck extends Animal implements Swimmable, Flyable {
    @Override
    public void swim() { System.out.println("泳いでいます"); }
    
    @Override
    public void fly() { System.out.println("飛んでいます"); }
    
    @Override
    public void makeSound() { System.out.println("ガーガー！"); }
}
```

### 4. 将来的な拡張性を考慮する場合 → インターフェース＋デフォルト実装

:::message alert
**重要**: これが最も実践的な使い分けポイントです！
:::

#### 問題：抽象クラスにメソッドを追加すると…

抽象クラスに新しい抽象メソッドを追加すると、**すべての派生クラスで実装が必要**になります。

```csharp
// 抽象クラスに新しいメソッドを追加
public abstract class Logger
{
    public abstract void Log(string message);
    public abstract void LogError(string message);  // 新規追加 → 全派生クラスに影響！
}
```

#### 解決策：インターフェースのデフォルト実装を活用

C# 8.0以降、Java 8以降では、インターフェースに**デフォルト実装**を追加できます。これにより、既存の実装クラスを壊さずに機能を拡張できます。

```csharp
// C# 8.0以降
public interface ILogger
{
    void Log(string message);
    
    // デフォルト実装付きメソッド（新規追加しても既存クラスに影響なし）
    void LogInfo(string message) => Log($"[INFO] {message}");
    void LogError(string message) => Log($"[ERROR] {message}");
}

// 既存のクラスは変更不要
public class FileLogger : ILogger
{
    public void Log(string message)
    {
        // ファイルにログを書き込む処理
    }
    // LogInfo、LogErrorはデフォルト実装が使われる
}
```

```java
// Java 8以降
public interface Logger {
    void log(String message);
    
    // デフォルト実装
    default void logInfo(String message) {
        log("[INFO] " + message);
    }
    
    default void logError(String message) {
        log("[ERROR] " + message);
    }
}

// 既存のクラスは変更不要
public class FileLogger implements Logger {
    @Override
    public void log(String message) {
        // ファイルにログを書き込む処理
    }
    // logInfo、logErrorはデフォルト実装が使われる
}
```

## 判断フローチャート

```
機能を追加・共有したい
         │
         ▼
  状態（フィールド）が必要？
         │
    ┌────┴────┐
   YES        NO
    │          │
    ▼          ▼
 抽象クラス   複数継承が必要？
              │
         ┌────┴────┐
        YES        NO
         │          │
         ▼          ▼
   インターフェース  将来の拡張性重視？
                    │
               ┌────┴────┐
              YES        NO
               │          │
               ▼          ▼
         インターフェース  抽象クラス
        （デフォルト実装付き）
```

## 実践的な選択基準まとめ

| シチュエーション | 推奨 | 理由 |
|-----------------|------|------|
| 共通の状態・実装を共有したい | 抽象クラス | フィールドとコンストラクタを持てる |
| 「何ができるか」の契約だけ定義したい | インターフェース | 実装の自由度が高い |
| 複数の機能を組み合わせたい | インターフェース | 多重実装が可能 |
| 将来の拡張性を重視したい | インターフェース | デフォルト実装で既存コードを壊さない |
| 密接に関連するクラス群の基底 | 抽象クラス | is-a関係を明確に表現できる |

## 注意点：デフォルト実装の制限

インターフェースのデフォルト実装には、抽象クラスと比べて以下の制限があります。

| 制限事項 | 説明 |
|---------|------|
| 状態を持てない | フィールド（インスタンス変数）を定義できない |
| コンストラクタなし | 初期化処理を定義できない |
| 継承されない（C#） | 実装クラスから直接呼び出せない（インターフェース型経由のみ） |

```csharp
// C#での注意点
public class FileLogger : ILogger
{
    public void Log(string message) { /* ... */ }
}

var logger = new FileLogger();
// logger.LogInfo("test");  // コンパイルエラー！

ILogger ilogger = logger;
ilogger.LogInfo("test");    // OK（インターフェース型経由）
```

## まとめ

### 結論

- **抽象クラス**: 共通の実装や状態を持つ、密接に関連したクラス群の基底として使用
- **インターフェース**: 契約の定義、複数機能の組み合わせ、将来の拡張性を重視する場合に使用

### 筆者の見解

実務では、**将来的な機能追加**を考慮してインターフェースを選択するケースが多いです。

抽象クラスに新しい抽象メソッドを追加すると、すべての派生クラスに影響が出てしまいます。一方、インターフェースのデフォルト実装を活用すれば、既存のコードを壊さずに機能を拡張できます。

**迷ったらインターフェース**を選び、必要に応じてデフォルト実装を追加する方針がおすすめです。

## 参考資料

- [Microsoft Docs - Abstract and Sealed Classes](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/abstract-and-sealed-classes-and-class-members)
- [Microsoft Docs - Default Interface Methods](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8#default-interface-methods)
- [Oracle Java Documentation - Abstract Methods and Classes](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)
- [Baeldung - Interface With Default Methods vs Abstract Class](https://www.baeldung.com/java-interface-default-method-vs-abstract-class)
