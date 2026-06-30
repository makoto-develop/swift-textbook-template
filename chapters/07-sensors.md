# 第7章：センサーの活用

> 執筆者：易　コウカ
> 最終更新：2026年6月26日

## この章で学ぶこと

この章では、iPhoneに内蔵されているジャイロセンサーや加速度センサーを統合して管理する `CoreMotion` フレームワークの基礎を学ぶ。

## 模範コードの全体像

```swift
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

iPhoneの傾きなどの状態をリアルタイムに検知し、画面中央の丸いバブルが水準器のように動くアプリです。前後左右の傾きがほぼ水平になると、バブルが赤から緑色に変わり、画面に「水平！」と大きく表示されます。

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
private let motionManager = CMMotionManager()
var isAvailable: Bool
init() {
    isAvailable = motionManager.isDeviceMotionAvailable
}
```

**何をしているか：**
デバイスの各種モーションセンサーを統括する `CMMotionManager` のインスタンスを作成し、初期化の段階で、この端末に「デバイスモーション」が備わっているかを `isDeviceMotionAvailable` で確認して結果を保存しています。

**なぜこう書くのか：**
センサーが存在しない環境で無理やり計測を始めるとアプリが正常に動かなくなるため、事前に利用可能か、使えないかを判定し、画面の表示を安全に切り替える（`ContentUnavailableView` を出す）ためにこう書いています。

**もしこう書かなかったら：**
シミュレータでアプリを起動した際にも、センサーがある前提で画面（円や十字線のUI）が表示されてしまいます。しかし、いくらシミュレータを動かしても数値は「0」から一切動かないため、ユーザーや開発者自身が「バグで動かないのか、環境のせいで動かないのか」の区別がつかなくなります。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
```

**何をしているか：**
`CoreMotion` が計算してくれたデバイスの現在の傾き情報から、3次元の回転成分である前後の傾き、左右の傾き、水平方向の回転をそれぞれ取り出し、画面と連動している変数に代入しています。

**なぜこう書くのか：**
`deviceMotion` を使うことで、OS側がそれらの生データを高度な数理モデルで合成し、最初から綺麗な「傾き角度」として提供してくれるため、このプロパティを直接使うのがベストです。

**もしこう書かなかったら：**
自分で難しい数学（三角関数など）の計算コードをたくさん書く羽目になり、しかも動きがガタガタになります。

---

### 歩数計（CMPedometer）

```swift
pedometer.startUpdates(from: Date()) { [weak self] data, error in
    guard let self = self, let data = data else { return }

    DispatchQueue.main.async {
        self.stepCount = data.numberOfSteps.intValue
        if let dist = data.distance {
            self.distance = dist.doubleValue
        }
    }
}
```

**何をしているか：**
スタートボタンを押した瞬間から、ユーザーが歩いた歩数と距離をリアルタイムに数え始めます。

**なぜこう書くのか：**
歩数データはバックグラウンドで計算されるため、画面をスムーズに書き換えるために`DispatchQueue.main.async`を使ってメインの画面処理にデータを届ける必要があるからです。

**もしこう書かなかったら：**
何歩歩いても画面の数字が「0」のまま変わらなかったり、アプリが突然エラーを起こしてクラッシュしたりします。

---

### CoreLocationとの連携

```swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
    guard let location = newLocations.last else { return }
    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

**何をしているか：**
GPSが新しい位置をキャッチするたびに自動で呼び出され、「今の移動速度」と「通った場所の座標」を記録します。

**なぜこう書くのか：**
電波が悪いときに速度がマイナスになるのを防ぐため、`max(0, ...)`を使って「最低でも速度は0以下にならない」ように安全ガードをかけているからです。

**もしこう書かなかったら：**
外を走ったり歩いたりしても、現在の速度メーターや移動したルートの記録が全く反応せず、ずっと0 km/hのままになってしまいます。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `CMMotionManager` | iPhoneの傾き、加速、回転などのセンサーをまとめて管理するクラス。 | `let motionManager = CMMotionManager()` |
| `startDeviceMotionUpdates` | デバイスの傾きをリアルタイムに、`.main`へ届ける関数。 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| `CMDeviceMotion` | センサーから届くデータの塊。この中に傾き（attitude）などの情報が入っている。 | `motion.attitude.pitch` |
| `Path` | 直線や曲線を自分で自由に描いてカスタムの図形を作るための機能。 |　`Path { path in path.move(to: ...); path.addLine(to: ...) }` |

## 自分の実験メモ

**実験1：**
- やったこと：
水平判定の条件を abs(pitch) < 0.03 から abs(pitch) < 0.01 に狭めてみた。

- 結果：
バブルが緑色になって「水平！」と表示される判定が、前よりも厳しくなった。少しでも手が震えたり傾いたりすると、すぐに赤色に戻ってしまう。

- わかったこと：
何も

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
