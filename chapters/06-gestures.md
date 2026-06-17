# 第6章：ジェスチャー操作

> 執筆者：易　コウカ
> 最終更新：2026年6月17日

## この章で学ぶこと

この章では、SwiftUIでユーザーの指の操作を検出するジェスチャー機能について学ぶ。タップ、ロングプレス、ドラッグ、拡大縮小、回転などの基本的なジェスチャーの使い方を理解し、それぞれの状態管理方法も学ぶ。また、複数のジェスチャーを組み合わせて、より実践的なUIを実装する方法についても体験する。

## 模範コードの全体像

```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、SwiftUIの代表的なジェスチャーを実際に試しながら学習するためのサンプルアプリである。ユーザーはタップやロングプレスで色や状態を変更したり、ドラッグでカードを移動したり、ピンチで拡大縮小したり、回転ジェスチャーでオブジェクトを回転させたりできる。また、最後の画面では複数のジェスチャーを組み合わせて同時に操作できる。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
.onTapGesture {
    tapCount += 1
    backgroundColor = Color(
    hue: Double.random(in: 0...1),
    saturation: 0.7,
    brightness: 0.9
    )
}

.onLongPressGesture(minimumDuration: 0.5) {
    isPressed = true
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        isPressed = false
    }
}
```

**何をしているか：**
`onTapGesture` はタップされた回数をカウントし、背景色をランダムに変更している。`onLongPressGesture` は一定時間押し続けたことを検出し、円の色やサイズを変更して長押し成功を表現している。

**なぜこう書くのか：**
SwiftUIではタップや長押しを簡単に扱うための専用Modifierが用意されている。ボタンを使わなくてもビューに直接ジェスチャーを追加できるため、コードをシンプルに書くことができる。

**もしこう書かなかったら：**
`onTapGesture` を削除するとタップしても何も起こらなくなる。`onLongPressGesture` を削除すると長押しを検出できず、色やサイズの変化も発生しない。

---

### ドラッグジェスチャーとオフセット管理

```swift
.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
```

**何をしているか：**
`DragGesture`を利用してユーザーの指の移動量を取得し、その値を `offset` に反映してカードを移動している。ドラッグ終了時には現在位置を `lastOffset` に保存し、次回のドラッグ開始位置として利用している。

**なぜこう書くのか：**
`value.translation` はドラッグ開始地点からの移動量しか保持していない。そのため、前回までの移動結果を保持する `lastOffset` が必要になる。これによって何回ドラッグしても現在位置を維持できる。

**もしこう書かなかったら：**
`lastOffset` を使わない場合、毎回ドラッグ開始時に位置がずれたり、ビューが元の位置へ戻るような挙動になる。ドラッグ終了後の位置を保持できなくなる。

---

### 拡大縮小と回転

```swift
.gesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)

.gesture(
    RotateGesture()
        .onChanged { value in
            angle = lastAngle + value.rotation
        }
        .onEnded { _ in
            lastAngle = angle
        }
)
```

**何をしているか：**
`MagnifyGesture` はピンチ操作による拡大縮小を検出し、`scaleEffect` に反映している。`RotateGesture` は二本指による回転操作を検出し、`rotationEffect` に反映している。

**なぜこう書くのか：**
ジェスチャー中の変化量だけでなく、前回までの状態を保持する必要があるため `lastScale` と `lastAngle` を使用している。これにより何度も連続して拡大や回転を行うことができる。

**もしこう書かなかったら：**
`lastScale` や `lastAngle` がない場合、毎回ジェスチャー開始時に初期状態へ戻ったような挙動になり、連続した操作ができなくなる。

---

### ジェスチャーの組み合わせとアニメーション

```swift
.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
)
.gesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)
.gesture(
    RotateGesture()
        .onChanged { value in
            angle = lastAngle + value.rotation
        }
        .onEnded { _ in
            lastAngle = angle
        }
)
```

**何をしているか：**
ドラッグ、拡大縮小、回転の3種類のジェスチャーを同じビューに適用し、ユーザーが複数の操作を行えるようにしている。

**なぜこう書くのか：**
実際のアプリでは単一のジェスチャーだけではなく、複数のジェスチャーを組み合わせる場面が多い。そのため、基本ジェスチャーを組み合わせる方法を学ぶことは重要である。また、実験の結果から、複数の継続ジェスチャーを扱う場合は `simultaneousGesture()` を利用した方が安定して動作することが分かった。

**もしこう書かなかったら：**
ジェスチャーを組み合わせない場合、拡大だけ、回転だけといった単純な操作しかできなくなる。また、複数の `.gesture()` を使用するとジェスチャー同士が競合し、一部の操作が正しく認識されないことがある。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `DragGesture` | ドラッグジェスチャーを認識するジェスチャーレコグナイザー | `.gesture(DragGesture().onChanged { ... })` |
| `MagnifyGesture` | ピンチジェスチャーで拡大・縮小を認識 | `.gesture(MagnificationGesture().onChanged { scale in ... })` |
| `onLongPressGesture` | ロングプレスの書き方 | .onLongPressGesture(minimumDuration: 0.5) {} |
| `RotateGesture` |　回転のジェスチャー | RotateGesture().onChanged { value in} .onEnded { _ in} |
| `simultaneousGesture` |　gestureの新しい書き方 | .simultaneousGesture() |

## 自分の実験メモ

**実験1：**
- やったこと：`DragGesture`の`.onEnded`で`value.velocity`を取得して、カードに慣性移動を追加した。
- 結果：指を速く離すと、カードが少し先まで移動するようになった。
- わかったこと：`velocity`を使うと物理的な動きを表現できる。

**実験2：**
- やったこと：`.gesture()`を`.simultaneousGesture()`に変更した。
- 結果：`.gesture()`の場合、拡大縮小・回転を同時にうまく動作しないことがあった。
- わかったこと：複数の継続ジェスチャーを同時に使いたい場合は、`simultaneousGesture`が適している。SwiftUIの`gesture`はジェスチャー同士が競合する場合がある。

**実験3：**
- やったこと：`DispatchQueue.main.asyncAfter(deadline: .now() + 1) {}`を`Task { try? await Task.sleep(for: .seconds(1))}` に変更した

**実験4：**
- やったこと：`withAnimation(.spring)`を`withAnimation(.snappy)`に変更した

**実験5：**
- やったこと：`gesture`の`.onChanged { value in }`を`.updating($dragOffset) { value, state, _ in state = value.translation}`に変更した

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
iOS 26・27でドラッグ処理の新しい書き方はありますか

   **得られた理解：**
`DragGesture`の基本的な考え方は変わっていないことが分かった。でも新しい書き方ができたら、必ず新しいの方法を使います。

2. **質問：**
複数のジェスチャーをを同時に使う場合、新しい書き方はありますか

   **得られた理解：**
実際に試したところ、`.gesture()` では回転がうまく動かなかったが、`.simultaneousGesture()` に変更すると正常に動作した。複数ジェスチャーではこちらを使うべきだと理解した。

3. **質問：**
   **得られた理解：**

## この章のまとめ

この章ではSwiftUIの基本的なジェスチャー機能について学んだ。タップやロングプレスだけでなく、ドラッグ、拡大縮小、回転などの継続的なジェスチャーでは状態管理が重要であることを理解した。また、複数のジェスチャーを組み合わせる場合にはジェスチャー同士の競合が発生することがあり、`simultaneousGesture()` を利用すると改善できることも確認した。実際にコードを書き換えて実験したことで、ジェスチャーの仕組みをより深く理解できた。
