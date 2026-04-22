# 第1章：WebAPIの基本

> 執筆者：易　コウカ
> 最終更新：2026-04-22

## この章で学ぶこと

この章では、インターネット上のサーバーからデータを取得する「WebAPI連携」の基礎を学びます。具体的には、iTunesの公開APIを使って音楽データを取得し、SwiftUIの List を使って検索結果を表示するアプリの作り方をマスターします。

## 模範コードの全体像

```swift
// ============================================
// 第1章（応用）：エラーハンドリングとMVVM構成
// ============================================
// 基本編のコードをMVVMパターンで書き直し、
// エラーハンドリングとローディング状態を
// より適切に管理するバージョンです。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let resultCount: Int
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}

// MARK: - ViewModel

@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?

    enum SearchError: LocalizedError {
        case invalidURL
        case networkError(Error)
        case decodingError(Error)
        case noResults

        var errorDescription: String? {
            switch self {
                case .invalidURL:
                    return "検索URLの作成に失敗しました"
                case .networkError(let error):
                    return "通信エラー: \(error.localizedDescription)"
                case .decodingError:
                    return "データの読み取りに失敗しました"
                case .noResults:
                    return "検索結果が見つかりませんでした"
            }
        }
    }

    func searchMusic() async {
        guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }
        print("searchText: \(searchText)")
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }
        print("encodedText: \(encodedText)")

        let urlString = "https://itunes.apple.com/search?term=\(searchText)&media=music&country=jp&limit=25"

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
        } catch let error as DecodingError {
            errorMessage = SearchError.decodingError(error).errorDescription
            songs = []
        } catch {
            errorMessage = SearchError.networkError(error).errorDescription
            songs = []
        }

        isLoading = false
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - 検索バー

    private var searchBar: some View {
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
        .padding()
    }

    // MARK: - コンテンツエリア

    @ViewBuilder
    private var contentArea: some View {
        if viewModel.isLoading {
            Spacer()
            ProgressView("検索中...")
            Spacer()
        } else if viewModel.songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(viewModel.songs) { song in
                NavigationLink(destination: SongDetailView(song: song)) {
                    SongRow(song: song)
                }
            }
        }
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
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
        .padding(.vertical, 4)
    }
}

// MARK: - 詳細ビュー

struct SongDetailView: View {
    let song: Song

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                    image.resizable().aspectRatio(contentMode: .fit)
                } placeholder: {
                    ProgressView()
                }
                .frame(width: 200, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 8)

                Text(song.trackName)
                    .font(.title2)
                    .bold()
                    .multilineTextAlignment(.center)

                Text(song.artistName)
                    .font(.title3)
                    .foregroundStyle(.secondary)

                if let albumName = song.collectionName {
                    Text(albumName)
                        .font(.subheadline)
                        .foregroundStyle(.tertiary)
                }

                Text(song.priceText)
                    .font(.headline)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                    .background(.blue.opacity(0.1))
                    .clipShape(Capsule())
            }
            .padding()
        }
        .navigationTitle("曲の詳細")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - エラーバナー

struct ErrorBanner: View {
    let message: String

    var body: some View {
        HStack {
            Image(systemName: "exclamationmark.triangle.fill")
                .foregroundStyle(.yellow)
            Text(message)
                .font(.caption)
        }
        .padding(10)
        .frame(maxWidth: .infinity)
        .background(.red.opacity(0.1))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

検索バーにアーティスト名などを入力すると、iTunesにある最新の楽曲情報を最大25件取得してリスト表示します。各行には曲名、歌手名、ジャケット写真、価格が表示され、タップすると詳細画面へ遷移します。

## コードの詳細解説

### データモデル（Codable構造体）

```swift

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}
```

**何をしているか：**
APIから送られてくるJSON形式のデータを受け取るための「器（うつわ）」を作っています。JSONのキー名と構造体の変数名を一致させることで、自動的にデータを流し込めるようにしています。

**なぜこう書くのか：**
Codable を使うのがSwiftで最も標準的で、少ないコード量でJSON解析（デコード）ができるからです。また、Identifiable を継承して id を用意することで、List で表示する際にSwiftUIが「どのデータがどの行か」を正確に判別できるようにしています。

**もしこう書かなかったら：**
JSONデータを辞書型（Dictionary）として一つずつ手動で取り出す必要があり、コードが非常に長く、読みづらくなります。また、Identifiable がないと、リストの表示が不安定になったり、アニメーションが正しく動かなかったりします。

---

### API通信の処理

```swift
guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }
        print("searchText: \(searchText)")
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }
        print("encodedText: \(encodedText)")

        let urlString = "https://itunes.apple.com/search?term=\(searchText)&media=music&country=jp&limit=25"

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
        } catch let error as DecodingError {
            errorMessage = SearchError.decodingError(error).errorDescription
            songs = []
        } catch {
            errorMessage = SearchError.networkError(error).errorDescription
            songs = []
        }

        isLoading = false
```

**何をしているか：**
入力された文字列をURLに組み込んで、インターネット経由でiTunesのサーバーにデータをリクエストしています。データが届いたら、先ほど作った SearchResponse という器に変換して、画面表示用の songs 配列に保存しています。

**なぜこう書くのか：**
async/await を使うことで、時間がかかる通信処理を「待機」させつつ、コード自体は上から順にスッキリ書けるからです。do-catch を使うことで、ネットが切れているなどの「失敗した時」の処理もセットで書くことができます。

**もしこう書かなかったら：**
通信中の数秒間、アプリの画面が固まって操作できなくなる（フリーズしたような状態）可能性があります。また、try? などでエラーを無視してしまうと、なぜ検索結果が出ないのか原因がわからず、ユーザーが困ってしまいます。

---

### ビューの構成

```swift
struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - 検索バー

    private var searchBar: some View {
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
        .padding()
    }

    // MARK: - コンテンツエリア

    @ViewBuilder
    private var contentArea: some View {
        if viewModel.isLoading {
            Spacer()
            ProgressView("検索中...")
            Spacer()
        } else if viewModel.songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(viewModel.songs) { song in
                NavigationLink(destination: SongDetailView(song: song)) {
                    SongRow(song: song)
                }
            }
        }
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
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
        .padding(.vertical, 4)
    }
}

// MARK: - 詳細ビュー

struct SongDetailView: View {
    let song: Song

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                    image.resizable().aspectRatio(contentMode: .fit)
                } placeholder: {
                    ProgressView()
                }
                .frame(width: 200, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 8)

                Text(song.trackName)
                    .font(.title2)
                    .bold()
                    .multilineTextAlignment(.center)

                Text(song.artistName)
                    .font(.title3)
                    .foregroundStyle(.secondary)

                if let albumName = song.collectionName {
                    Text(albumName)
                        .font(.subheadline)
                        .foregroundStyle(.tertiary)
                }

                Text(song.priceText)
                    .font(.headline)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                    .background(.blue.opacity(0.1))
                    .clipShape(Capsule())
            }
            .padding()
        }
        .navigationTitle("曲の詳細")
        .navigationBarTitleDisplayMode(.inline)
    }
}
```

**何をしているか：**
画像のURLを指定して、インターネットから画像をダウンロードして表示しています。ダウンロードが終わるまでは placeholder（ぐるぐる回るインジケータ）を表示させています。

**なぜこう書くのか：**
AsyncImage は、画像のダウンロードから表示、メモリ管理までをSwiftUIが自動でやってくれる非常に便利な標準機能だからです。

**もしこう書かなかったら：**
自分で通信処理を書いて、データを UIImage に変換して…という非常に複雑な処理が必要です。また、placeholder がないと、画像が表示されるまで画面に穴が空いたような違和感が出てしまいます。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `addingPercentEncoding` | URLに使えない文字（日本語など）を専用の形式に変換する。 | `searchText.addingPercentEncoding(...)` |
| `@Observable` | データの変化を検知して画面を自動更新する最新の仕組み。 | `@Observable class MusicSearchViewModel` |
| | | |

## 自分の実験メモ

**実験1：**　limit（検索件数）の数字を変えてみた
- やったこと：　urlString の末尾にある limit=25 を limit=1 や limit=200 に書き換えて実行した。
- 結果：ちゃんと表示される件数が変わった。
- わかったこと：APIにリクエストを送る際の「URLパラメータ」をいじるだけで、サーバーから返ってくるデータをコントロールできるのが面白かった。

**実験2：**　機内モードにして検索ボタンを押してみた
- やったこと：通信をオフにしてエラーハンドリングが動くか試した。
- 結果：自作の ErrorBanner が表示され、「通信エラー」という文字が出た。
- わかったこと：正常に動く時だけじゃなくて、失敗した時（異常系）の処理を catch でしっかり書かないと、アプリがフリーズしたりして不親切な設計になることがわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
Codable を使うとき、JSONのキー名と変数名をわざと変えることはできる？
   **得られた理解：**
APIから返ってくるデータ（JSON）のキー名は trackName だけど、自分のコードでは songTitle みたいに好きな名前を使いたい場合がある。その時は CodingKeys という列挙型（enum）を構造体の中に定義して、マッピングすればいいことがわかった。これをしないと、変数名を一文字でも間違えるだけでデコードエラーになってアプリが動かなくなる理由が納得できた。

3. **質問：**
async await を使わずに URLSession を使う方法（昔の書き方）との違いは？
   **得られた理解：**　
昔は「クロージャ（Completion Handler）」を使って、通信が終わった後の処理をネスト（入れ子）にして書いていたらしい。async await を使うと、上から下に順番に実行されるように見えるので、コードがすごく読みやすくなる。特にエラーが起きた時に do-catch でまとめて処理できるのが、専門学生レベルの自分にとってもバグを減らせる大きなメリットだと感じた。

4. **質問：**
AsyncImage の placeholder って絶対に書かないとダメ？
   **得られた理解：**　
書かなくても動くけど、インターネットが遅い時に画面が真っ白になって、ユーザーが「バグかな？」と不安になってしまう。placeholder に ProgressView（くるくる）やグレーの四角を表示しておくことで、アプリの「おもてなし（UX）」が向上することを学んだ。技術的な実装だけでなく、使う人の気持ちを考える大切さに気づけた。

## この章のまとめ

### 通信は「必ず失敗するもの」として書く
ネットが遅い、URLが間違っている、サーバーが落ちている……。WebAPIの世界では、自分のコードが正しくてもエラーが起きます。do-catch を使って、**「エラーが起きた時にユーザーに何を伝えるか」**までをセットで設計するのが、エンジニアとしての責任だと感じました。

### ユーザーを待たせない工夫（UX）
通信には時間がかかります。ボタンを押して何も起きないと、ユーザーは連打したりアプリを閉じたりします。isLoading フラグを使って ProgressView を出したり、AsyncImage の placeholder を使ったりすることで、「今頑張って処理しているよ」という状況を伝える大切さを実感しました。
