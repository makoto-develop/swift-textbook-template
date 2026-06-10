# 第5章：機能統合の実践

> 執筆者：易　コウカ
> 最終更新：2026年6月10日

## この章で学ぶこと

この章では、これまでに学んだカメラ（写真選択）、地図（MapKit）、**データ保存（SwiftData）**の3つの独立した機能を1つに組み合わせ、「フォトマップ」アプリを実装する方法を学びます。

## 模範コードの全体像

```swift
// ============================================
// 第5章：カメラ + 地図 + データ保存の統合アプリ
// ============================================
// 写真を撮影し、撮影場所を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
//   - NSCameraUsageDescription（実機の場合）
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}
```

**このアプリは何をするものか：**

自分がどこで写真を撮ったのかが地図上の丸いアイコンで一目で分かり、タップするとその写真の詳細やメモを確認できます。また、一覧画面では過去の記録が日付順にリスト表示され、不要になったデータはスワイプで簡単に削除できます。

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**
SwiftDataを使ってアプリ内に保存する「写真の記録データ」の構造を定義しています。

**なぜこう書くのか：**
@Modelを付与することで、複雑なデータベースの構築コードを一切書かずに、クラスをそのままSwiftDataのテーブルとして利用できるためです。

**もしこう書かなかったら：**
もし `imageData` を `Data` 型ではなく直接 `UIImage` で保存しようとすると、SwiftDataがサポートしていない型であるためコンパイルエラーになります。また、`coordinate` の変換プロパティがない場合、地図にピンを立てるたびに毎回 `CLLocationCoordinate2D(latitude: record.latitude, longitude: record.longitude)` という長いコードを各ビューに書く必要があります。

---

### タブ構成の設計

```swift
TabView {
    MapTab()
        .tabItem {
            Label("マップ", systemImage: "map")
        }

    ListTab()
        .tabItem {
            Label("一覧", systemImage: "list.bullet")
        }
}
```

**何をしているか：**
SwiftUIの TabView を使って、画面下部に「マップ」と「一覧」の2つの切り替えタブを配置し、アプリのメインナビゲーションを構築しています。

**なぜこう書くのか：**
ユーザーが「地図から直感的に写真を探したい時」と「リストから日付順にしっかりとデータを確認・削除したい時」の2つの目的を、画面をワンタップで行き来できるようにするため、モバイルアプリで最も一般的な UI 設計パターンである下部タブを採用しています。

**もしこう書かなかったら：**
画面遷移がナビゲーションリンクだけになってしまい、マップから一覧に切り替えるために一度画面を戻って別のボタンを押す、といった非常に面倒な操作の手間が発生してしまいます。

---

### カメラと位置情報の連携

```swift
@State private var locationManager = LocationManager()
AddRecordView(locationManager: locationManager)
if let location = locationManager.currentLocation {
    Text("緯度: \(location.latitude, specifier: "%.4f")")
    Text("経度: \(location.longitude, specifier: "%.4f")")
}
```

**何をしているか：**
`CLLocationManager` をバックグラウンドで動かして常にスマホの現在地を追跡し、ユーザーが「新しい記録」画面を開いた時に、その瞬間の最新の緯度・経度を自動的にキャッチして写真データに埋め込める状態を作っています。

**なぜこう書くのか：**
`@Observable`を使った `LocationManager` クラスを作成することで、GPSの位置情報が更新されるたびに SwiftUI の画面が自動的に再描画され、ユーザーに「今位置情報が取得できているか」をリアルタイムで分かりやすく提示できるからです。

**もしこう書かなかったら：**
位置情報の取得タイミングと画面の更新が同期せず、写真の保存ボタンを押したときに位置情報が空（nil）のままになって保存ができなかったり、最悪の場合、アプリが現在地を取得しようとした瞬間に画面がフリーズしてクラッシュする原因になります。

---

### SwiftDataでの画像保存

```swift
func saveRecord() {
    guard let location = locationManager.currentLocation else { return }

    let record = PhotoRecord(
        title: title,
        memo: memo,
        latitude: location.latitude,
        longitude: location.longitude,
        imageData: selectedImageData
    )
    modelContext.insert(record)
```

**何をしているか：**
ユーザーが入力した情報と写真、そして取得した位置情報をまとめて `PhotoRecord` インスタンスを生成し、`modelContext.insert(record)` によって`SwiftDataの管理環境にデータを投げ込んで永続化`（保存）しています。

**なぜこう書くのか：**
SwiftDataでは、`modelContext.insert()` を呼び出すだけで、バックグラウンドで自動的にディスクへの書き込み（コミット）が行われるため、明示的に「保存処理」の重い非同期コードを自分で書く必要がなく、安全かつ高速にデータを保存できるからです。

**もしこう書かなかったら：**
`modelContext.insert(record)` を忘れると、データはメモリ上（画面上）だけで一時的に作られた状態になり、アプリを一度終了して再起動した瞬間に、せっかく撮影して入力した写真データがすべて消えて空っぽになってしまいます。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TabView` | 複数のビューをタブで切り替えるコンポーネント | `TabView { ... }.tabViewStyle(.page)` |
| 例：`CLLocationManager` | GPS位置情報を取得するAPIManager | `let location = manager.location?.coordinate` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
