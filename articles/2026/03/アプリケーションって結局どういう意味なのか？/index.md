---
title: "アプリケーションって結局どういう意味なのか？"
type: "Idea"
description: "本来「適用・応用」という行為を指す抽象名詞であった "Application" が、いかにして現代の「ソフトウェア」という実体を指す言葉に変遷したのかを整理したメモ。

・https://gemini.google.com/app/6d971f79a0670178"
tags: ["Tips"]
date: "2026-03-27T00:00:00"
---


## 1. 語源としての「適用・応用」
もともと英語の **"Apply"** は、ラテン語の *applicare*（〜に合わせる、貼り付ける）を語源としています。そこから派生した **"Application"** は、本来「何かを特定の目的に向けて使うこと（適用・応用）」という**行為**を指す言葉でした。  
- **Application of medicine:** 薬を塗ること（患部への適用）  
- **Application of theory:** 理論の応用  

## 2. コンピュータの「使い道」としての誕生
初期のコンピュータは、特定の作業（計算など）を行うための巨大な機械でした。1940年代〜50年代、コンピュータを特定の業務（給与計算、軌道計算など）に使う際、それは **「コンピュータの特定の課題への適用（Application of computers to specific tasks）」** と呼ばれていました。  
つまり、最初は「ソフト」という実体を指すのではなく、 **「その機械を使って何を成し遂げるか」という目的（用途）** を指していたのです。  

## 3. 「システム」と「アプリケーション」の分離
1960年代に入り、コンピュータの仕組みが複雑になると、役割が2つに分かれました。  
1. **System Software:** 機械を動かし、管理するための土台（OSの原型）  
2. **Application Software:** その土台の上で、ユーザーが具体的な「目的（＝適用例）」を果たすための道具  
この「特定の目的のために応用されたソフトウェア」を略して、次第に **"Application"** と呼ぶようになりました。  

## 4. 「App」というアイコンへの変遷
1990年代までは、まだ「応用ソフト」というニュアンスが残る少し硬い言葉でした。しかし、2008年にAppleが **"App Store"** を公開したことで、その呼び名は決定的なものになります。  
「特定の目的のための道具」だったアプリケーションは、スマートフォンの普及とともに、誰もが気軽にタッチして起動する **"App（アプリ）"** という非常に身近な存在へと姿を変えたのです。  

## 5. なぜAppleは「App（アプリ）」と呼んだのか？
ここからが面白いところです。「Application」を「App」という極めて身近な言葉に変えたのはAppleですが、そこにはいくつかのドラマがありました。  
### ① NeXTSTEPから続く「.app」の伝統
技術的な背景として、AppleのOSの基礎となった **NeXTSTEP**（ジョブズがAppleを離れていた時に作ったOS）時代からのこだわりがあります。  
- **詳細:** NeXTSTEPでは、プログラム一式をまとめたフォルダを **「.app」** という拡張子で管理していました。これが後のmacOSやiOSに引き継がれています。ジョブズはWindowsが好んだ「Program」という言葉を嫌い、一貫して「Application」という言葉を推奨していました。  
### ② Salesforce CEOからの「贈り物」という実話
意外なことに、「App Store」という名前の権利とドメインを最初に持っていたのはAppleではなく、SalesforceのCEOマーク・ベニオフ氏でした。  
- **経緯:** 2003年、ベニオフ氏がスティーブ・ジョブズにアドバイスを求めた際、ジョブズは「アプリケーションのエコシステムを作るべきだ」と助言。これに触発されたベニオフ氏は「AppStore.com」というドメインと商標を登録しました。  
- **ギフト:** 2008年にAppleが「App Store」を発表した際、ベニオフ氏は恩師であるジョブズへの感謝として、その**ドメインと商標を無償で譲渡**しました。  
### ③  「今年の言葉（2010）」に選ばれた社会現象
Appleが「There's an app for that（それ、アプリでできるよ）」というキャッチコピーに巨額の広告費を投じたことで、「App」は専門用語から日常語へ昇華しました。  
- **事実:** 2010年、アメリカ方言協会（ADS）は **"App" を「Word of the Year」** に選出。選考理由として「Appleのマーケティングの力により、12ヶ月で爆発的に普及した」ことが挙げられています。  

## 6. 補足：プラットフォーム間の哲学（Program vs Application）
Windows（IBM/Microsoft）とMac（Apple/NeXT）では、長年採用する用語が異なっていた。これはコンピュータに対する捉え方の違いに起因しています。  
### ①  Windows（IBM系）：マシン中心の「Program」
初期のMicrosoft（およびIBM）は、コンピュータを **「命令を実行する機械」** として捉えていました。  
- **視点:** プログラマ・エンジニア視点。  
- **定義:** コンピュータに対する「一連の指示（指令）」がProgram。  
- **象徴:** Windows 3.xの頃、ソフトを管理する中心機能は **「プログラム・マネージャ（Program Manager）」** という名称でした。  
- **ニュアンス:** 「機械に何をさせるか」という実行プロセスに重きが置かれています。  
### ② Mac（Apple/NeXT系）：ユーザー中心の「Application」
スティーブ・ジョブズや初期のAppleは、コンピュータを **「人間の能力を拡張する知の自転車（Bicycle for the Mind）」** と定義しました。  
- **視点:** ユーザー・人間中心視点。  
- **定義:** 機械の万能性を、人間の特定の活動に「適用」したものがApplication。  
- **象徴:** Macintoshのガイドライン（HIG）では、初期から「Program」という言葉を避け、「Application」と呼ぶよう開発者に徹底させていました。  
- **ニュアンス:** 「人間が何を成し遂げるか」という目的・用途に重きが置かれています。  

## まとめ：言葉に込められた「万能性」への敬意
「アプリケーション」という名前は、コンピュータが **何にでもなれる万能な機械」** であることの裏返しです。  
計算機に「文章作成」という目的を**適用**すればワープロになり、「通信」を**適用**すればチャットツールになる。  
私たちが普段「アプリ」と呼んでいるものは、先人たちが「この万能な機械を、どうやって人間の役に立てようか？」と試行錯誤して生み出してきた「適用の軌跡」そのものだと言えるでしょう。  

## 参考リンク

### A. 語源・コンピュータ史全般
| 内容                                    | URL                                                                                                                                                    |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Applicationの語源（Etymonline）            | [https://www.etymonline.com/word/application](https://www.etymonline.com/word/application)                                                             |
| Application softwareの定義と歴史（Wikipedia）   | [https://en.wikipedia.org/wiki/Application_software](https://en.wikipedia.org/wiki/Application_software)                                               |
| 初期メインフレームの用途（IBM Archives）            | [https://www.ibm.com/ibm/history/exhibits/mainframes/mainframes_intro.html](https://www.ibm.com/ibm/history/exhibits/mainframes/mainframes_intro.html)   |
### B. Apple・NeXT・App Storeの歴史
| 内容                               | URL                                                                                                                                                                                                                                                    |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| NeXTSTEPとAppの深い歴史（CHM）           | [https://computerhistory.org/blog/the-deep-history-of-your-apps-steve-jobs-nextstep-and-early-object-oriented-programming/](https://computerhistory.org/blog/the-deep-history-of-your-apps-steve-jobs-nextstep-and-early-object-oriented-programming/)   |
| App Store命名の裏話（9to5Mac）          | [https://9to5mac.com/2020/01/02/marc-benioff-app-store-before-apple-steve-jobs-help/](https://9to5mac.com/2020/01/02/marc-benioff-app-store-before-apple-steve-jobs-help/)                                                                             |
| ジョブズのアドバイスと商標譲渡（Salesforce Blog）   | [https://www.salesforce.com/blog/steve-jobs-inspired-appexchange/](https://www.salesforce.com/blog/steve-jobs-inspired-appexchange/)                                                                                                                   |
### C. 言葉の普及と社会現象
| 内容                          | URL                                                                                                                                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "App" が2010年「今年の言葉」に選出（ADS）   | [https://americandialect.org/American-Dialect-Society-2010-Word-of-the-Year-PRESS-RELEASE.pdf](https://americandialect.org/American-Dialect-Society-2010-Word-of-the-Year-PRESS-RELEASE.pdf)   |
### d. プラットフォーム間の哲学
| 内容                                            | URL                                                                                                                                                    |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Apple Human Interface Guidelines (Historical)   | [https://developer.apple.com/design/human-interface-guidelines/](https://developer.apple.com/design/human-interface-guidelines/)                       |
| Windows Program Managerの歴史                    | [https://en.wikipedia.org/wiki/Program_Manager](https://en.wikipedia.org/wiki/Program_Manager)                                                         |
| ジョブズの「知の自転車」インタビュー                            | [https://www.computerhistory.org/revolution/personal-computers/17/297/1253](https://www.computerhistory.org/revolution/personal-computers/17/297/1253)   |