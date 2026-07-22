# 第8章：ウィジェット

> 執筆者：易　コウカ
> 最終更新：2026年7月8日

## この章で学ぶこと

この章では、アプリ内に名言をリストとして表示して、WidgetKitを使ってホーム画面やロック画面に表示できるウィジェットを実装する方法を学ぶ。具体的には毎日異なる名言を表示するウィジェットを題材にして、ウィジェットビューの構成、複数サイズへの対応、そしてメインアプリとの連携方法を学ぶ。

## 模範コードの全体像

```swift
import SwiftUI

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

import Foundation

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}

```

**このアプリは何をするものか：**

いくつかの名言をアプリ内でリストに表示して、携帯のホーム画面にWidgetを提供してます。

## コードの詳細解説

### TimelineProviderの仕組み

```swift
func placeholder(in context: Context) -> QuoteEntry {
    QuoteEntry(
        date: Date(),
        quote: Quote(id: 0, text: "読み込み中...", author: "")
    )
}

// スナップショット（ウィジェットギャラリーでのプレビュー）
func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
    let entry = QuoteEntry(
        date: Date(),
        quote: QuoteStore.todaysQuote()
    )
    completion(entry)
}

// タイムライン（実際のウィジェット更新スケジュール）
func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
    let currentDate = Date()
    let quote = QuoteStore.todaysQuote()
    let entry = QuoteEntry(date: currentDate, quote: quote)

    // 次の日の0時にウィジェットを更新
    let tomorrow = Calendar.current.startOfDay(
        for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
    )

    let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
    completion(timeline)
}
```

**何をしているか：**
ウィジェットの「仮の読み込み画面」「設定画面でのプレビュー」「実際の更新スケジュール」の3つの表示タイミングを管理しています。

**なぜこう書くのか：**
ウィジェットはアプリのように常に裏で動き続けるとバッテリーを激しく消費するため、あらかじめ「次の日の0時に新しくして」とOSに予定表（タイムライン）を渡す仕組みになっているからです。

**もしこう書かなかったら：**
名言が毎日自動で日替わり更新されず、ずっと同じ名言が表示されっぱなしになってしまいます。

---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}
```

**何をしているか：**
何日の何時にどのquoteを表示するかという、スケジュール帳の1ページ分を作っています。

**なぜこう書くのか：**
OSが指定された時間になった瞬間に、このデータをウィジェットの画面に自動で描き直させるためです。

**もしこう書かなかったら：**
時間のデータと表示したい名言データがバラバラになり、OSがいつ、何を画面に出せばいいのか分からなくなってエラーになります。

---

### ウィジェットサイズごとのレイアウト

```swift
var smallWidget: some View {
      VStack(spacing: 4) {
          Image(systemName: "quote.opening")
              .font(.caption)
              .foregroundStyle(.blue)

          Text(entry.quote.text)
              .font(.caption)
              .bold()
              .multilineTextAlignment(.center)
              .lineLimit(3)

          Text(entry.quote.author)
              .font(.caption2)
              .foregroundStyle(.secondary)
      }
      .padding(12)
  }

  // 中サイズ
  var mediumWidget: some View {
      HStack(spacing: 16) {
          Image(systemName: "quote.opening")
              .font(.title)
              .foregroundStyle(.blue)

          VStack(alignment: .leading, spacing: 4) {
              Text("今日の名言")
                  .font(.caption2)
                  .foregroundStyle(.secondary)

              Text(entry.quote.text)
                  .font(.subheadline)
                  .bold()

              Text("— \(entry.quote.author)")
                  .font(.caption)
                  .foregroundStyle(.secondary)
          }

          Spacer()
      }
      .padding()
  }
```

**何をしているか：**
ホーム画面に置くウィジェットのサイズがsmallかmediumかによって、画面のデザインを自動で切り替えています。

**なぜこう書くのか：**
小さいサイズのデザインのまま横長サイズに表示すると、左右に不自然な余白が空いて不恰好になるため、横長用にデザインを用意しています。

**もしこう書かなかったら：**
どのサイズでホーム画面に置いても同じ見た目になってしまい、文字がはみ出したり、余白がスカスカになったりしてデザインが崩れます。

---

### メインアプリとの連携

```swift
// セットアップ手順4より
// QuoteStore.swift を選び、右側インスペクタの Target Membership で
// 「メインアプリ」と「QuoteWidget Extension」の両方にチェックを入れる
```

**何をしているか：**
名言のQuoteStoreを、メインアプリとウィジェット側の両方のファイルから同時に使えるようにシェアしています。

**なぜこう書くのか：**
メインアプリとウィジェットは見た目は1つのアプリですが、内部的には別々のプログラムとして動いているため、データのファイルを共有設定にしてあげる必要があるからです。

**もしこう書かなかったら：**
ウィジェット側のコードでQuoteやQuoteStoreのデータ型は見つかりませんカラビルドできなくなります。

---


## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `TimelineProvider` | ウィジェットを更新するタイミングとコンテンツを定義 | `struct QuoteProvider: TimelineProvider { ... }` |
| `@main` + `WidgetConfiguration` | ウィジェットのエントリーポイント | `@main struct QuoteWidget: Widget { ... }` |
| `\.widgetFamily` | 現在ユーザーがホーム画面に置いているウィジェットのサイズを取得する機能 | `@Environment(\.widgetFamily) var family` |
| `containerBackground` | ウィジェットで背景の色や質感を安全に指定するための新しい壁紙設定 | `.containerBackground(.fill.tertiary, for: .widget)` |
| `Calendar.current.startOfDay` | 指定した日付の夜中の0時0分0秒の時間を正確に計算してくれる機能 | `Calendar.current.startOfDay(for: tomorrow)` |

## 自分の実験メモ

**実験1：**
- やったこと：
`QuoteWidgetEntryView の .supportedFamilies([.systemSmall, .systemMedium]) から .systemSmall を削り、大サイズ（.systemLarge）`を追加してみた。

- 結果：
ウィジェット追加画面で小さいサイズが選べなくなり、大きなサイズの領域が選べるようになった、ただ、デザインを記述していなかったため `mediumWidget` のレイアウトがそのまま拡大表示された。

- わかったこと：
`supportedFamilies` で指定したサイズしかユーザーのホーム画面には置けないこと、サイズを増やす場合は `switch family` に専用のデザインを準備する必要があることが分かった。

**実験2：**
- やったこと：
ウィジェット側から `QuoteStore.quotes.append(...)` を実行して、動的に新しい名言を配列に追加しようと試みた。

- 結果：
ウィジェットが表示される一瞬だけは追加されたように見えたが、次回更新時や別サイズの表示に切り替えた瞬間に元の7件に戻ってしまい、追加したデータが保持されなかった。

- わかったこと：
ウィジェット拡張は更新のたびにプロセスが生成・破棄される使い捨ての構造になっているため、QuoteStore のようなメモリ上の静的データ（static var）を変更しても永続化されない。メインアプリとウィジェット間でユーザーが追加したデータなどを動的に同期させたい場合は、メモリ上の変更ではなく `App Group` を経由した `UserDefaults` や `Shared File` への書き込みが必要不可欠だと理解できた。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

このウィジェット開発は「未来の表示スケジュールを事前に組み立ててOSに委ねる」こと。アプリのようにリアルタイムで画面を書き換えるのではなく、`TimelineProvider` で「何時にどのデータを出すか」のコマ送り図鑑を作ってOSに渡すのが基本ルール。
また、メインアプリとウィジェットは別々のプログラムとして扱われるため、共通で使いたいデータ構造`Quote` などは `Target Membership` の設定を忘れないこと。
