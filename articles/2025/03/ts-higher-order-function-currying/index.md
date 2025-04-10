---
type: "Tech"
title: "【TS】今さら聞けない高階関数・カリー化"
tags: ["TypeScript","FP"]
date: "2025-03-31T00:00:00"
---


第一級関数、高階関数、カリー化を復習するために写経する。  

[https://zenn.dev/nekoniki/articles/5b7980fac91048775931](https://zenn.dev/nekoniki/articles/5b7980fac91048775931)  

## 第一級関数
第一級オブジェクトとは「生成・代入・演算・(引数・戻り値としての)受け渡し」と言った基本的な操作を制限なしに使用できるオブジェクトを指す。  
つまり、「第一級関数」は「関数」を「引数や戻り値」として扱うことができる関数、と読み解くことが出来る。  

## 高階関数
> 高階関数とは、第一級関数をサポートしているプログラミング言語において少なくとも以下のうち1つを満たす関数である。  
> ・関数(手続き)を引数にとる  
> ・関数を返す  
高階関数とは「関数」を「引数もしくは戻り値、あるいは両方」として扱う関数、と読み解ける。  

## カリー化
``` typescript
const fetch = (method: 'POST' | 'GET') => (url: string) => (param: {}) => {
    return () => {
      // 3種類の引数を使って非同期処理を実行...
    }
}

const post = fetch('POST');
const get = fetch('GET');

const postHoge = post('http://hogehogehogehoge');
const postFuga = post('http://fugafugafugafuga');
const getFuga = get('http://fugafugafugafuga');

const postHogeXXXX = postHoge({value: 'xxxx'});
const postFugaXXXX = postFuga({value: 'xxxx'});
const getFugaNoParam = getFuga({});

postHogeXXXX();
postFugaXXXX();
getFugaNoParam();
```

1. `fetch` は `(method: 'POST' | 'GET')` を受け取り、その `method` を使って `(url: string) => (param: {}) => () => {}` という多段階の関数を返します。  
   - `fetch('POST')` や `fetch('GET')` を呼ぶと `(url: string) => (param: {}) => () => {}` の関数が返る。  
2. `post = fetch('POST')`、`get = fetch('GET')` により、それぞれのメソッドに固定された関数を作成。  
3. `post('http://hogehogehogehoge')` のようにURLを渡すと `(param: {}) => () => {}` という関数が返る。  
4. さらに `postHoge({ value: 'xxxx' })` のように `param` を渡すと `() => {}` という関数が返る。  
5. 最後に `postHogeXXXX()` を実行すると、非同期処理（実際のHTTPリクエストなど）が実行される（コード上では未実装）。  
このように、**関数を段階的に適用することで、メソッド → URL → パラメータの順に細かく関数を分割している** のが特徴ですね。  

[https://chatgpt.com/share/67ea474d-1ff0-8013-b5f6-4eed85d8f483](https://chatgpt.com/share/67ea474d-1ff0-8013-b5f6-4eed85d8f483)  
