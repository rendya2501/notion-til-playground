---
type: "Tech"
title: "Strategy パターンにおける検索最適化：FirstOrDefault から Dictionary への移行と DI コンテナでの実装例"
tags: ["C#","DesignPattern",".NET"]
date: "2025-04-20T00:00:00"
---

## はじめに

本記事は、.NET の DI コンテナを用いたストラテジーパターン実装の備忘録です。  
.NETのDIコンテナ `Microsoft.Extensions.DependencyInjection` でストラテジーパターンを実装した際、同一インターフェースを `IEnumerable<IStrategy>`  で注入出来ることを知ったので、  `FirstOrDefault`  で戦略を見つける形で実装していましたが、O(n)の計算量がかかるというので、O(1)で検索できる `Dictionary` で実装してみたところ実現出来たので、それらの実装方法をまとめることにしました。  

## 実行環境
- Win11  
- コンソールアプリ  
- .NET 8  
- ライブラリ  
  - `Microsoft.Extensions.DependencyInjection`  

## 実装の概要
- 入力ごとに異なる「処理戦略（Strategy）」を適用  
- キーには `enum` を使用（型安全で扱いやすい）  
- `Dictionary<StrategyKey, IStrategy>` によって戦略を O(1) で高速に取得  
- 戦略が見つからない場合は、`DefaultStrategy` を適用  

**Strategyの識別に使うenumの定義**  
``` c#
public enum StrategyKey
{
    TypeA,
    TypeB,
    TypeC,
    Unknown
}
```

**Strategyのインターフェースと実装**  
``` c#
// Strategy インターフェース
public interface IStrategy
{
    StrategyKey Key { get; }
    string Execute(string input);
}

// DefaultStrategy インターフェース
public interface IDefaultStrategy : IStrategy { }
```

**それぞれの処理戦略の実装**  
``` c#
// 通常のストラテジー
public class StrategyA : IStrategy
{
    public StrategyKey Key => StrategyKey.TypeA;
    public string Execute(string input) => $"[A Strategy] Input: {input}";
}

public class StrategyB : IStrategy
{
    public StrategyKey Key => StrategyKey.TypeB;
    public string Execute(string input) => $"[B Strategy] Input: {input}";
}

public class StrategyC : IStrategy
{
    public StrategyKey Key => StrategyKey.TypeC;
    public string Execute(string input) => $"[C Strategy] Input: {input}";
}


// デフォルトストラテジー
public class DefaultStrategy : IDefaultStrategy
{
    public StrategyKey Key => StrategyKey.Unknown;
    public string Execute(string input) => $"[Default Strategy] Input: {input}";
}
```

## 案1 : FirstOrDefault パターン

**ContentHandler の実装**  
``` c#
public interface IContentHandler
{
    string HandleContent(List<(StrategyKey key, string input)> inputs);
}

public class ContentHandler(
    IEnumerable<IStrategy> _strategies,
    IDefaultStrategy _defaultStrategy) : IContentHandler
{
    public string HandleContent(List<(StrategyKey key, string input)> inputs)
    {
        var result = new List<string>();
        
        foreach (var (key, input) in inputs)
        {
            var strategy = _strategies.FirstOrDefault(s => s.Key == key)
                ?? _defaultStrategy;

            result.Add(strategy.Execute(input));
        }
        
        return string.Join("\n", result);
    }
}
```

**DIコンテナへの登録と処理の実行**  
``` c#
using Microsoft.Extensions.DependencyInjection;

// Program.cs などの DI コンテナ設定
var services = new ServiceCollection();

// Strategy を複数登録
services.AddSingleton<IStrategy, StrategyA>();
services.AddSingleton<IStrategy, StrategyB>();
services.AddSingleton<IStrategy, StrategyC>();
services.AddSingleton<IDefaultStrategy, DefaultStrategy>();

// ContentGenerator を登録
services.AddScoped<IContentHandler, ContentHandler>();

var inputs = new List<(StrategyKey, string)>
{
    (StrategyKey.TypeA, "1. First input"),
    (StrategyKey.TypeB, "2. Second input"),
    (StrategyKey.TypeC, "3. Third input"),
    (StrategyKey.TypeA, "4. Fourth input"),
    (StrategyKey.TypeC, "5. Fifth input"),
    ((StrategyKey)999, "6. Unknown input") // 存在しないキー
};

// サービスプロバイダーの構築
var serviceProvider = services.BuildServiceProvider();
var handler = serviceProvider .GetRequiredService<IContentHandler>();
// 実行
var output = handler.HandleContent(inputs);

Console.WriteLine("=== 1. FirstOrDefault Pattern Strategy Output ===");
Console.WriteLine(output);
```

**実行結果**  
``` plain text
=== 1. FirstOrDefault Pattern Strategy Output ===
[A Strategy] Input: 1. First input
[B Strategy] Input: 2. Second input
[C Strategy] Input: 3. Third input
[A Strategy] Input: 4. Fourth input
[C Strategy] Input: 5. Fifth input
[Default Strategy] Input: 6. Unknown input
```

## 案2 : Dictionaryパターン

**ContentHandler の実装**  
``` c#
public interface IContentHandler
{
    string HandleContent(List<(StrategyKey key, string input)> inputs);
}

public class ContentHandler(
    IDictionary<StrategyKey, IStrategy> _strategyMap,
    IDefaultStrategy _defaultStrategy) : IContentHandler
{
    public string HandleContent(List<(StrategyKey key, string input)> inputs)
    {
        var result = new List<string>();
        
        foreach (var (key, input) in inputs)
        {
            var strategy = _strategyMap.TryGetValue(key, out var foundStrategy)
                ? foundStrategy
                : _defaultStrategy;

            result.Add(strategy.Execute(input));
        }
        
        return string.Join("\n", result);
    }
}
```

**DIコンテナへの登録と処理の実行**  
``` c#
using Microsoft.Extensions.DependencyInjection;

// Program.cs などの DI コンテナ設定
var services = new ServiceCollection();

// Strategy を複数登録
services.AddSingleton<IStrategy, StrategyA>();
services.AddSingleton<IStrategy, StrategyB>();
services.AddSingleton<IStrategy, StrategyC>();
services.AddSingleton<IDefaultStrategy, DefaultStrategy>();

// Dictionary を生成して DI コンテナに登録
services.AddSingleton<IDictionary<StrategyKey, IStrategy>>(provider =>
{
    var strategies = provider.GetServices<IStrategy>();
    return strategies.ToDictionary(s => s.Key);
});

// ContentGenerator を登録
services.AddScoped<IContentHandler, ContentHandler>();

var inputs = new List<(StrategyKey, string)>
{
    (StrategyKey.TypeA, "1. First input"),
    (StrategyKey.TypeB, "2. Second input"),
    (StrategyKey.TypeC, "3. Third input"),
    (StrategyKey.TypeA, "4. Fourth input"),
    (StrategyKey.TypeC, "5. Fifth input"),
    ((StrategyKey)999, "6. Unknown input") // 存在しないキー
};

// サービスプロバイダーの構築
var serviceProvider = services.BuildServiceProvider();
var handler = serviceProvider .GetRequiredService<IContentHandler>();
// 実行
var output = handler.HandleContent(inputs);

Console.WriteLine("=== 2. Dictionary Pattern Strategy Output ===");
Console.WriteLine(output);
```

**実行結果**  
``` plain text
=== 2. Dictionary Pattern Strategy Output ===
[A Strategy] Input: 1. First input
[B Strategy] Input: 2. Second input
[C Strategy] Input: 3. Third input
[A Strategy] Input: 4. Fourth input
[C Strategy] Input: 5. Fifth input
[Default Strategy] Input: 6. Unknown input
```

## 案3 : IDictionary拡張 パターン

`IDictionary` では `GetValueOrDefault` メソッドが存在しないので `IDictionary` インターフェースに拡張メソッドを定義して `GetValueOrDefault` メソッドを実装して使用できるようにしたパターン。  
やっていることは案2と変わりないです。  
DIを `IDictionary` 型ではなく `Dictionary` 型にするとやらなくても良いことではあるが、割と感動したのでメモしておく。  

``` c#

public interface IContentHandler
{
    string HandleContent(List<(StrategyKey key, string input)> inputs);
}

public class ContentHandler(
    IDictionary<StrategyKey, IStrategy> _strategyMap,
    IDefaultStrategy _defaultStrategy) : IContentHandler
{
    public string HandleContent(List<(StrategyKey key, string input)> inputs)
    {
        var result = new List<string>();
        
        foreach (var (key, input) in inputs)
        {
		        // IDictionary型でGetValueOrDefaultメソッドを使用する
            var strategy = _strategyMap.GetValueOrDefault(key, _defaultStrategy);

            result.Add(strategy.Execute(input));
        }
        
        return string.Join("\n", result);
    }
}

// 拡張メソッドを定義
public static class DictionaryExtensions
{
    public static TValue GetValueOrDefault<TKey, TValue>(
        this IDictionary<TKey, TValue> dictionary,
        TKey key,
        TValue defaultValue = default!)
    {
        return dictionary.TryGetValue(key, out var value) ? value : defaultValue;
    }
}
```

## メリットとデメリット（一般論として）

**メリット**  
- **検索速度の向上**：O(n) → O(1)  
- **拡張性**：Strategy を追加・削除しやすい  
- **テストしやすい**：依存注入によりテスト可能性向上  

**デメリット**  
- **初期化コスト**：Dictionary 構築時に O(n)  
- **構成の複雑化**：Factory や Dictionary の管理が必要  

## 結論（適用判断の指針）

| 条件                  | 推奨手法                          |
| ------------------- | ----------------------------- |
| Strategy 数が少ない（<10）   | `FirstOrDefault` でも十分         |
| Strategy 数が多い（>100）   | `Dictionary` による高速化推奨         |
| 頻繁に検索が発生する処理        | `Dictionary` を推奨              |
| 将来的な拡張や可読性を重視する場合   | `Factory + Dictionary` の構成を採用   |

小規模なアプリや一時的な用途であれば `FirstOrDefault` の簡潔さがメリットですが、大規模・高頻度のケースでは `Dictionary` を用いた方式が有効です。設計の柔軟性を重視する場合は拡張メソッドや Factory を組み合わせた構成も検討すると良いでしょう。  
