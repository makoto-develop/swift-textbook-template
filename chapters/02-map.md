# 第2章：地図アプリの基本

> 執筆者：易　コウカ
> 最終更新：2026/05/08

## この章で学ぶこと

MapKitを使用して、アプリに地図を導入して、地図上に独自のマーカーを表示し、それをカテゴリごとに絞り込む機能を実装します。

## 模範コードの全体像

```swift
import SwiftUI
import MapKit

// MARK: - データモデル
struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ
extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー
struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター
struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード
struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

<img width="310" height="644" alt="image" src="https://github.com/user-attachments/assets/9b74af61-9b38-407f-a0a9-b8a35c5b91a2" />
東京の有名なランドマーク（寺院、タワー、公園）を地図上に表示するアプリです。画面下のボタンをタップして表示するカテゴリを切り替えたり、マーカーをタップしてその場所の詳細情報をカード形式で確認したりできます。

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ
extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(...)
    ]
}
```

**何をしているか：**

ランドマークの情報をひとまとめにするための「設計図」の役割を果たしています。
名前、説明文、地図上の座標（緯度・経度）、そして「寺社」や「公園」のカテゴリなど、一つの場所に必要な情報をセットにして定義しています。

**なぜこう書くのか：**

``Identifiable`` や ``Hashable`` というプロトコルに従って書くことで、SwiftUIがそれぞれのデータを見分けることができるようになるからです。

 - ``Identifiable``: 各ランドマークに固有のID（UUID）を持たせることで、同名の場所があっても別物として区別できます。

 - ``enum Category``: カテゴリを文字列ではなく列挙型で定義することで、打ち間違いを防ぎ、アイコンや色を自動的に割り当てやすくしています。

**もしこう書かなかったら：**

``
static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
``

 - 書かないとエラーが出る：Type 'Landmark' does not conform to protocol 'Equatable'

 - extension Landmark中のsampleDataをstruct Landmarkに移動して、何にも変わってない

---

### 地図の表示とカメラ制御

```swift
Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))
```

**何をしているか：**

Map を使って画面に地図を表示し、「今どこを映しているか」というカメラの視点を表示しています。
具体的には、``@State`` で定義した ``cameraPosition`` を使って、アプリ起動時に東京駅周辺を適切な拡大サイズで表示するように設定しています。

**なぜこう書くのか：**

``Map(position: $cameraPosition)`` のように、変数の前に ``$`` をつけて「バインディング」状態にしているからです。
こうすることで、コード側から表示場所を指定できるだけでなく、ユーザーが手で地図を動かした時に、その新しい位置情報が自動的に ``cameraPosition`` 変数に書き込まれ、プログラムと画面が常に同期されるようになります。

**もしこう書かなかったら：**

 - positionを書かないと、東京駅周辺じゃなくて東京周辺に表示します。
 - selectionを書かないと、浅草寺などの場所クリックしても反応ない
 
---

### マーカーの表示

```swift
Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
```

**何をしているか：**

地図上の指定した座標にマーカーを表示しています。
ただの点ではなく、ランドマークの名前を表示し、カテゴリに応じたアイコン（systemImage）と色（tint）を付けて、一目で何があるか分かるようにデコレーションしています。

**なぜこう書くのか：**

``Marker`` という SwiftUI 専用の部品を使うことで、地図のズームレベルに合わせて自動でサイズを調整してくれるからです。

``.tint``: カテゴリごとに色を変えることで、「赤は寺社」「青はタワー」といった視覚的な分類を直感的に伝えています。

``.tag(landmark)``: これを書くことで、ユーザーがマーカーをタップした時に「どの場所が選ばれたか」をプログラムが特定できるようになり、詳細カードの表示を可能にしています。

**もしこう書かなかったら：**

 - ``.tag(landmark)``を書かないと、マーカーをクリックしても反応ない
 - ``.tint(landmark.category.color)``を書かないと、マーカーの背景色は全部が赤くになる
 - ``systemImage: landmark.category.iconName,``を書かないと、アイコンは全部同じのアイコンにする

---

### フィルター機能

```swift
HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
```

**何をしているか：**

画面下のボタンを使って、地図に表示するランドマークをカテゴリ（寺社・タワー・公園）ごとにフィルターします。
ユーザーが選んだカテゴリだけを ``Landmark.sampleData`` から探し出し、地図上のマーカーを更新しています。

**なぜこう書くのか：**

``@State``（選択状態）と 計算プロパティ：filteredLandmarksを組み合わせることで、「データが変われば、画面も自動で変わる」という SwiftUI の強みを活かすためです。

``Set<Category>``: 複数のカテゴリを同時に選択したり解除したりする操作を効率よく行うために、重複がない ``Set`` 型を使っています。

``filter`` メソッド: 元のデータを壊さずに、条件に合うものだけを抽出して表示用のリストを作る、安全でスマートな書き方です。

**もしこう書かなかったら：**

もし計算プロパティを使わなかったら、同期が取れなくなる、コードが長く複雑になる

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`extension` | 既存のクラスにメソッドを追加するというのが目的であった | `struct Landmark、extension Landmark` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：extension Landmark中のstatic let sampleDataをstruct Landmarkに移動して
- 結果：何にも変わってない
- わかったこと：extension を使うのは、「関連するものをグループ化して、コードの見通しを良くする」という開発者の工夫のひとつです。

**実験2：**
- やったこと：
このコードを追加
```
extension CLLocationCoordinate2D: Equatable {
    public static func == (lhs: CLLocationCoordinate2D, rhs: CLLocationCoordinate2D) -> Bool {
        return lhs.latitude == rhs.latitude && lhs.longitude == rhs.longitude
    }
}
```
- 結果：エラーが消えた
- わかったこと：
  `SwiftUI` の `.onChange(of: value)` メソッドは、監視する対象のデータ型が「比較可能（Equatable）」であることを要求します。しかし、Appleの公式フレームワーク `CoreLocation` で定義されている `CLLocationCoordinate2D` 構造体は、標準では `Equatable` ではないため、値が変化したかどうかを `onChange` が判断できずコンパイルエラーになります。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

struct Landmarkの後のextensionは何の為に書いた、一つのメソッドでコードが長すぎならないように読みやすくなのか
extension Landmark {
    static let sampleData
    
   **得られた理解：**

   - 役割（データの定義と実際のデータ）を分けるため
   - コードの肥大化を防ぎ、読みやすくするため
   - 「型名.プロパティ名」で呼び出せるようにするため


2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
