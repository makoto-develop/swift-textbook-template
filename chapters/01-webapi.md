# 第1章：WebAPIの基本

> 執筆者：易　コウカ
> 最終更新：2026-04-15

## この章で学ぶこと
ネットから曲のデータをもらって、アプリに表示するやり方を学びます。iTunes APIを使って、好きなアーティストを検索できるアプリを作ります。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**
曲の曲名またはアーティストを検索して、結果をリストに表示して、リストのアイテムをクリックして曲の詳細が表示されます。

<img width="283" height="570" alt="image" src="https://github.com/user-attachments/assets/ff6f2b77-e36a-4904-a8af-58ac37a91bcb" />
<img width="270" height="572" alt="image" src="https://github.com/user-attachments/assets/43f538c3-d574-4ea1-9456-c3cc0ad7f432" />

## コードの詳細解説

### データモデル（Codable構造体）

```swift
let (data, _) = try await URLSession.shared.data(from: url)
let response = try JSONDecoder().decode(SearchResponse.self, from: data)
```

**何をしているか：**
インターネットから届いたJSON（ただの文字の集まり）を、Swiftで使える「SearchResponse」や「Song」という形に翻訳しています。

**なぜこう書くのか：**
```JSONDecoder``` を使うと、プログラムが自動で「名前」と「内容」をマッチングしてくれるからです。自分で一つずつ翻訳コードを書く必要がないので、間違いが少なくなります。

**もしこう書かなかったら：**
届いたデータがただの長い文字列のままなので、アプリの中で「曲名を表示する」といった操作ができなくなります。

---

### API通信の処理

```swift
let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        isLoading = true
        errorMessage = nil

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)

            if response.results.isEmpty {
                errorMessage = SearchError.noResults.errorDescription
                songs = []
            } else {
                songs = response.results
            }
        }
```

**何をしているか：** iTunesのサーバーに「この曲を教えて！」とリクエストを送って、データをもらってくる処理です。

**なぜこう書くのか：** ```URLSession.shared.data(from: url)``` を使うと、ネットからデータをダウンロードしている間もアプリが固まらず、スムーズに動くからです。また、do-catch を使うことで、ネットが繋がらないときなどの「失敗」にも対応できます。

**もしこう書かなかったら：**

---

### ビューの構成

```swift
HStack {
            TextField("アーティスト名を入力", text: $viewModel.searchText)
                .textFieldStyle(.roundedBorder)
                .onSubmit {
                    Task { await viewModel.searchMusic() }
                }

            Button("検索") {
                Task { await viewModel.searchMusic() }
            }
            .buttonStyle(.borderedProminent)
            .disabled(viewModel.searchText.isEmpty || viewModel.isLoading)
        }
HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                RoundedRectangle(cornerRadius: 8)
                    .fill(.gray.opacity(0.2))
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)
                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text(song.priceText)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
```

**何をしているか：** 「検索バー」でユーザーの入力を受け取り、「リスト」で取ってきた曲の情報をきれいに並べて表示しています。

**なぜこう書くのか：** ```AsyncImage``` を使うと、ネット上の画像を表示するのをSwiftUIが自動でやってくれるからです。```VStack``` や ```HStack``` を使うことで、文字や画像をきれいに整列させることができます。

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable, Identifiable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable, Identifiable { ... }` |
| 例：`入力テキストの処理` | 日本語やスペースなど、そのままではURLに使えない文字を、URL専用の形式（%記法）に変換する処理 | `guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：入力テキストを変換処理せずにリクエストして
```
print("searchText: \(searchText)")
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }
        print("encodedText: \(encodedText)")

        let urlString = "https://itunes.apple.com/search?term=\(searchText)&media=music&country=jp&limit=25"
        
```
- 結果：エラーに出なかった
出力：
```
searchText: アイ　＾ス
encodedText: %E3%82%A2%E3%82%A4%E3%80%80%EF%BC%BE%E3%82%B9
```

- わかったこと：Swift（Foundation）の URL 型や、通信ライブラリが気を利かせて、裏側で勝手にエンコードしてくれている可能性が高いです

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
 ```
guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }
検索のキーワード、何これ
 ```
   **得られた理解：**
実は、日本語（漢字・ひらがな）やスペースは、そのままではURLに含めることができません。
変換したらユーザーがどんな言葉（日本語や記号）を入れても、アプリが壊れずに正しく検索できるようにするため

2. **質問：** searchTextが処理しなくてもアプリ内に結果は予期のように動作してエラーが出なかった
   **得られた理解：**
   出力結果を見ると、実は**「バグの一歩手前」**の状態になっています。
      実は最近のSwift（Foundation）の URL 型や、通信ライブラリが気を利かせて、裏側で勝手にエンコードしてくれている可能性が高いです。
しかし、これは「たまたま動いている」状態に近く、以下のようなリスクがあります。
予期せぬ動作: 特殊な記号（& や + など）を検索ワードに入れたとき、URLの構造が壊れて検索に失敗する。
文字化け: サーバーの仕様によっては、自動変換がうまくいかず、検索結果が0件になる。
   
4. **質問：**
   **得られた理解：**

5. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
