---
title: "ストラテジーパターンの基本と応用"
type: "Tech"
description: "コンボボックスにa,b,cの選択肢があり、それぞれの選択肢を選んだ場合の動作は異なります。
故にストラテジーパターンが使えると想定出来ます。
a,b,cの選択肢はenumとして定義され、enumに対してどのようなストラテジーを生成するかはファクトリーパターンが使えるように思えます。
  • https://claude.ai/chat/6fe1df62-3630-4fb9-b08f-5402b191f8e2
  • https://claude.ai/chat/34adc467-1ba0-4597-8752-1143b21669f7"
tags: ["C#","DesignPattern","Java",".NET"]
date: "2025-04-23T00:00:00"
---

## 概要

コンボボックスなどで `a`, `b`, `c` のような選択肢を選ぶ UI において、それぞれの選択肢に応じた異なる処理を実装する場合、**ストラテジーパターン**と**ファクトリーパターン**の活用が有効です。さらに、**enum をキーとした処理の切り替え**と、**依存性注入（DI）** を組み合わせることで柔軟かつテストしやすい設計が実現できます。  
以下では、実装方法を段階的に紹介します。  

## ステップ1：Context クラスを使った基本的なストラテジーパターン
**Strategy インターフェースと各実装**  
```c#
public interface IStrategy
{
    void Execute();
}

public class StrategyA : IStrategy
{
    public void Execute() => Console.WriteLine("Aの処理");
}

public class StrategyB : IStrategy
{
    public void Execute() => Console.WriteLine("Bの処理");
}

public class StrategyC : IStrategy
{
    public void Execute() => Console.WriteLine("Cの処理");
}
```

**Context クラス**  
```c#
public class Context
{
    private IStrategy _strategy;

    public void SetStrategy(IStrategy strategy)
    {
        _strategy = strategy;
    }

    public void ExecuteStrategy()
    {
        _strategy?.Execute();
    }
}
```

**利用例**  
```c#
var context = new Context();
context.SetStrategy(new StrategyA());
context.ExecuteStrategy();

context.SetStrategy(new StrategyB());
context.ExecuteStrategy();
```
> 🔍 明示的に Strategy を差し替えたいときや、柔軟に動的切り替えしたいときに有効なパターン。  

### **Contextクラスを使う場面**
`Context` クラスを使う利点は、以下のような場合に現れます：  
1. **ストラテジーを動的に切り替えたい場合**  
   実行時にストラテジーを柔軟に変更したい場合、`Context` クラスの `setStrategy()` を利用することで、選択を管理しやすくなります。  
2. **ストラテジー以外の処理が多い場合**  
   ストラテジーの実行以外にも、共通の前処理・後処理を含める必要がある場合、`Context` クラスでそれらを統一的に管理できます。  
3. **外部からの依存を減らす場合**  
   クライアントコードが直接 `Strategy` を扱うのではなく、`Context` を介することで依存を緩和し、拡張性を高められます。  

### **今回のケースでは不要な理由**
今回のように、選択肢が明確（`enum` で定義）であり、それぞれの動作が固定されている場合、`Context` クラスの役割が薄くなります。むしろ以下のような理由から `Context` を省略する方が適切です：  
1. **選択と処理の対応関係が単純明快**  
   `enum` を使うことで、選択肢とその処理を直接結びつけられるため、中間の管理者（`Context`）が不要です。  
2. **動的な切り替えが不要**  
   実行時にストラテジーを変更する必要がないため、`Context` を使って柔軟性を持たせるメリットがありません。  
3. **コードの簡潔化**  
   必要以上にクラスを増やすと設計が複雑になり、メンテナンス性が下がる可能性があります。  

### **設計をどう選ぶか**
- **動的にストラテジーを切り替える必要がある場合**  
  → `Context` を使うストラテジーパターンが適している。  
- **選択肢が固定的で、単純な対応関係がある場合**  
  → `enum` に直接処理を紐づける設計が適している。  

### **動的切り替えが必要な場合の例**
以下は、`Context` を使った動的な切り替えの例です：  
```java
// Contextクラス
public class Context {
    private Strategy strategy;

    public void setStrategy(Strategy strategy) {
        this.strategy = strategy;
    }

    public void executeStrategy() {
        if (strategy == null) {
            throw new IllegalStateException("Strategyが設定されていません");
        }
        strategy.execute();
    }
}

// 利用例
public class Main {
    public static void main(String[] args) {
        Context context = new Context();

        // 選択肢Aを選んだ場合
        context.setStrategy(new StrategyA());
        context.executeStrategy();  // Output: 選択肢 A の動作を実行

        // 動的に選択肢Bに変更
        context.setStrategy(new StrategyB());
        context.executeStrategy();  // Output: 選択肢 B の動作を実行
    }
}
```

### **動的切り替えが不要な場合（今回のケース）**
今回のように、`enum` によって処理が決定される場合、以下のようにシンプルな設計で十分です：  
```java

// ストラテジーを保持するenum
public enum OptionType {
    A(new StrategyA()),
    B(new StrategyB()),
    C(new StrategyC());

    private final Strategy strategy;

    OptionType(Strategy strategy) {
        this.strategy = strategy;
    }

    public Strategy getStrategy() {
        return strategy;
    }
}

// 使用例
public class Main {
    public static void main(String[] args) {
        OptionType selectedOption = OptionType.A;

        // enumから直接ストラテジーを取得して実行
        selectedOption.getStrategy().execute();  // Output: 選択肢 A の動作を実行
    }
}
```

### **結論**
- **シンプルなケース**（処理が固定的）：`enum` を利用して `Strategy` を直接紐づける設計が最適。  
- **柔軟性が必要なケース**（動的切り替え）：`Context` クラスを使用して、動的な切り替えを可能にする設計が最適。  
今回のような要件では、`enum` に処理を紐づけた設計が一番適切ですね！  

## ステップ2 : **ファクトリーパターンを用いた**ストラテジーパターン
**Enum の定義**  
```c#
public enum OptionType
{
    A,
    B,
    C
}
```

**Strategy インターフェースと各実装**  
```c#
public interface IStrategy
{
    void Execute();
}

public class StrategyA : IStrategy
{
    public void Execute() => Console.WriteLine("Aの処理");
}

public class StrategyB : IStrategy
{
    public void Execute() => Console.WriteLine("Bの処理");
}

public class StrategyC : IStrategy
{
    public void Execute() => Console.WriteLine("Cの処理");
}
```

**ファクトリーパターンを用いた StrategyFactory**  
```c#
public class StrategyFactory
{
    public static IStrategy Create(OptionType option)
    {
        return option switch
        {
            OptionType.A => new StrategyA(),
            OptionType.B => new StrategyB(),
            OptionType.C => new StrategyC(),
            _ => throw new ArgumentException("無効なオプション")
        };
    }
}
```

**利用例**  
```c#
var option = OptionType.B;
var strategy = StrategyFactory.Create(option);
strategy.Execute();
```

> 🔍 この段階では DI を使っていません。シンプルですが、依存が強くなります。  

## ステップ3 : StrategySelector による DI 対応 (インターフェースベース)
**IStrategy に OptionType を持たせる**  
```c#
public interface IStrategy
{
    OptionType TargetOption { get; }
    void Execute();
}
```

**StrategySelector クラス**  
```c#
public class StrategySelector
{
    private readonly IEnumerable<IStrategy> _strategies;

    public StrategySelector(IEnumerable<IStrategy> strategies)
    {
        _strategies = strategies;
    }

    public IStrategy GetStrategy(OptionType option)
    {
        return _strategies.FirstOrDefault(s => s.TargetOption == option)
               ?? throw new InvalidOperationException($"No strategy found for option: {option}");
    }
}
```

**DI 登録**  
```c#
services.AddTransient<IStrategy, StrategyA>();
services.AddTransient<IStrategy, StrategyB>();
services.AddTransient<IStrategy, StrategyC>();
services.AddSingleton<StrategySelector>();
```

**利用**  
```c#
var selector = serviceProvider.GetRequiredService<StrategySelector>();
var strategy = selector.GetStrategy(OptionType.B);
strategy.Execute();
```

## ステップ4：Func<OptionType, IStrategy> を使った柔軟な DI
**DI 登録**  
```c#
services.AddTransient<StrategyA>();
services.AddTransient<StrategyB>();
services.AddTransient<StrategyC>();

services.AddSingleton<Func<OptionType, IStrategy>>(sp =>
{
    var strategies = new List<IStrategy>
    {
        sp.GetRequiredService<StrategyA>(),
        sp.GetRequiredService<StrategyB>(),
        sp.GetRequiredService<StrategyC>()
    };

    var dict = strategies.ToDictionary(s => s.TargetOption);
    return option => dict.TryGetValue(option, out var strategy)
        ? strategy
        : throw new InvalidOperationException($"Strategy not found: {option}");
});
```

**利用**  
```c#
var resolver = serviceProvider.GetRequiredService<Func<OptionType, IStrategy>>();
var strategy = resolver(OptionType.C);
strategy.Execute();
```

## ステップ5：.NET 8+ の KeyedService を使った最先端の実装
**DI 登録**  
```c#
services.AddKeyedTransient<IStrategy, StrategyA>(OptionType.A);
services.AddKeyedTransient<IStrategy, StrategyB>(OptionType.B);
services.AddKeyedTransient<IStrategy, StrategyC>(OptionType.C);
```

**利用**  
```c#
var strategy = serviceProvider.GetRequiredKeyedService<IStrategy>(OptionType.B);
strategy.Execute();
```

## パターン比較まとめ
| パターン                   | 特徴           | 向いているケース      |
| ---------------------- | ------------ | ------------- |
| Factoryクラス（newあり）      | 単純・明確        | 小規模、DI不要な場面   |
| Context クラス            | 明示的な切り替え     | 動的に戦略を変更したい場面   |
| StrategySelector + DI   | 拡張しやすい・テスト可能   | 中～大規模・明示的制御   |
| Funcデリゲート              | 柔軟・効率的       | より抽象度の高い処理分離   |
| KeyedService (.NET 8+)   | 最新・公式対応      | .NET 8+ 環境前提   |

## まとめ
処理の切り替えを enum で管理しつつ、DI コンテナやストラテジーパターンを活用することで、保守性・可読性・テスト性が高い実装が可能になります。  
- 小規模：Factory クラスで十分  
- 拡張性重視：StrategySelector + DI  
- 柔軟に処理注入：Funcデリゲート  
- 最新構成：KeyedService（.NET 8+）  
目的や規模に応じて、適切なアプローチを選択しましょう。  

[https://chatgpt.com/share/68088d12-d628-8013-917a-67552071e042](https://chatgpt.com/share/68088d12-d628-8013-917a-67552071e042)  
