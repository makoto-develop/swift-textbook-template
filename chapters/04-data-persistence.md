# 第4章：データの永続化

> 執筆者：易　コウカ
> 最終更新：2026年6月3日

## この章で学ぶこと

この章では、SwiftUIアプリに「データの永続化」を組み込み、アプリを閉じてもデータや設定が消えない仕組みを学ぶ。構造化された本格的なデータを扱うための「SwiftData」と、ユーザー名やトグル状態などの簡易的な設定を保持するための「@AppStorage」という2つの手法を、シンプルなメモアプリの実装を通して体系的に理解する。

## 模範コードの全体像

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

```

**このアプリは何をするものか：**

作成日時、お気に入りフラグを持つ「メモデータ」を保存・編集・削除できるアプリ。
トップ画面ではメモが新しい順にリスト表示され、設定画面から「ユーザー名の変更」や「お気に入りを最上部に固定するソート条件」を切り替えることができる。これらのデータと設定は、アプリを完全に終了したり端末を再起動したりしても、すべて自動で引き継がれる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**

`SwiftData`がデータベースの「構造」として認識できるように、ただのSwiftのクラスに `@Model` マクロを付与してデータモデルを定義している。

**なぜこう書くのか：**

SwiftUIと相性いい`SwiftData`を利用するため。従来の`Core Data`のように複雑なGUI上でのモデリングファイルを必要とせず、ピュアなSwiftコードだけでプロパティの種類や初期化のルールを直感的に記述できるからである。

**もしこう書かなかったら：**

`@Model`を付与せず普通のクラスとして定義した場合、`.modelContainer` や `@Query` に渡した際にコンパイルエラーとなる。また、プロパティが変更された際、`SwiftUI`の画面（View）へ自動的に変更を通知するリアクティブな仕組み（`Observable`）も機能しなくなる。

---

### データの追加・削除（modelContext）

```swift
.onDelete(perform: deleteMemos)
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        let memo = displayedMemos[index]
        modelContext.delete(memo)
    }
}

let memo = Memo(title: title, content: content)
modelContext.insert(memo)
```

**何をしているか：**

データベースを操作するための`modelContext`に対して、生成したメモインスタンスを挿入したり、リストから選択された既存のインスタンスを削除したりしている。

**なぜこう書くのか：**

SwiftDataでは`modelContext`というメモリ上の作業スペースに命令を出すことで、安全かつ間接的に裏側のファイルを更新するアーキテクチャが採用されているため。

**もしこう書かなかったら：**

`modelContext.insert(memo)` や `.delete(memo)` を呼び出さない、あるいは実行を忘れた場合、メモリ上のUI状態（State など）だけが一時的に変わるか何も起きず、アプリを再起動した瞬間にすべての変更が消え去って元の状態に戻ってしまう。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
```

**何をしているか：**

データベースに保存されているすべての Memo データを全自動で取得し、作成日時の新しい順に並び替えた上で、配列`memos`に常に最新の状態で格納し続けている。

**なぜこう書くのか：**

手動で「データ読み込み関数」を実行する手間を一切省き、データベースの更新をSwiftUIのView側がリアルタイムで検知して画面を自動再描画（リアクティブ・レンダリング）できるようにするため。

**もしこう書かなかったら：**

単なる配列`var memos: [Memo]`として定義した場合、アプリ起動時に自前でデータベースからフェッチ（ロード）する処理を記述しなければ画面が空っぽのままになる。また、メモを追加・編集した後に画面を最新にするための「再読み込み処理」もすべて手動で同期をとらなければならず、コードが肥大化しバグの原因になる。

---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""
```

**何をしているか：**

iOSの軽量な保存領域である「UserDefaults」に直接アクセスし、ユーザー名やソート順のON/OFF（sortByFavorite）というシステム設定を、変数への代入・参照だけで永続化している。

**なぜこう書くのか：**

SwiftData（データベース）を構築するほどではない「たった1つの文字列や真偽値」のような軽量な環境設定を、@State と全く同じ感覚で最もシンプルに、かつバインディング（$）による双方向データ連携を維持したまま保存・運用するため。

**もしこう書かなかったら：**

アプリが起動するたびにユーザー名やソート設定が初期値（空欄、またはfalse）にリセットされてしまい、ユーザーはアプリを開くたびに毎回設定をやり直さなければならなくなる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| `@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
| `ContentUnavailableView` | データが無い時に表示される内容 | `ContentUnavailableView("", systemImage: "", description: Text(""))` |
| `onDelete` | リストから内容を削除 | `List { ForEach() {}.onDelete(perform: deleteMemos)` |
| `.cancellationAction / .confirmationAction` | ナビゲーションバーのボタンの表示される場所を配置 | `ToolbarItem(placement: .cancellationAction)` |

## 自分の実験メモ

**実験1：**
- やったこと： `TextEditor`を`TextField`に変わって　```TextField("内容", text: $content, axis: .vertical).lineLimit(5...)```
- 結果： 同じように複数行の入力欄だけど、`TextField`はヒントが表示される、リミットもできるし内容に応じて表示領域の縦に広げていける。
- わかったこと：入力欄なら`TextEditor`より`TextField`の方が適切だと思う

**実験2：**
- やったこと：```ToolbarItem(placement: .cancellationAction)```の`cancellationAction`を`topBarTrailing`に変わった
- 結果：見た目は全然同じ
- わかったこと：「左側に置きたいなら `.topBarLeading`、右側なら `.topBarTrailing` で十分では？」と思うかもしれませんが、Appleがわざわざ `.cancellationAction / .confirmationAction` を用意しているのは、アクセシビリティや多言語対応、複数デバイス展開（マルチプラットフォーム）を綺麗に行うためです。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
    toolbarのボタンToolbarItemのplacementにcancellationActionとtopBarTrailingの区別は何
   
   **得られた理解：**

   ToolbarItem で使用する placement の .cancellationAction と .topBarTrailing は、画面上の「配置」だけでなく、**「そのボタンが持つ意味（セマンティクス）」** において明確な違いがあります。.cancellationAction と .topBarTrailingを使ってアクセシビリティや多言語対応、複数デバイス展開（マルチプラットフォーム）を綺麗に行うためです。

3. **質問：**
    modelContext 何これ
   
   **得られた理解：**

   Swiftのデータ保存フレームワークCore Dataの進化版データを操作する為。
5. **質問：**
    配列はどうして@Queryを使うの、役割は
   
   **得られた理解：**

   データベースを常に監視し、変化があったらすぐに最新のデータを綺麗に並び替えて、画面に反映する役割。
## この章のまとめ

- **データの性質に合わせた棲み分け：** アプリの設定やユーザーの環境設定のような単一でシンプルなデータは`@AppStorage`へ、IDや日付など複数のプロパティが紐付いた構造化データは `SwiftData`へと適切に使い分けることが設計の基本である。
 - **SwiftDataのゴールデンペア：** データの変更命令は`modelContext`、画面へのリアルタイム反映は`@Query`という2つの役割分担を理解しておけば、画面の同期バグは原理的に発生しなくなる。
 - **セマンティクスを意識したUI設計：** テキスト入力（TextEditor / TextField）のヒント表示、表示領域の縦に文字に応じて広げると行数制限の挙動特性や、ツールバーボタン（ToolbarItem）への役割の付与など、見た目の固定配置だけでなく「システムやユーザーがどう認識するか」を考慮することが、モダンで洗練されたiOSアプリ開発には不可欠である。
