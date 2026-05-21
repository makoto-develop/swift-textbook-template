# 第3章：カメラの利用

> 執筆者：易　コウカ
> 最終更新：2026/05/15

## この章で学ぶこと

この章では、SwiftUIでフォトライブラリから画像を選択する最新の PhotosPicker の使い方と、UIViewControllerRepresentable を用いて標準のカメラ機能を統合する方法を学びます。

## 模範コードの全体像

```swift
// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、
// 画面に表示します。シミュレータでも動作します。
// ============================================

import SwiftUI
import PhotosUI

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、「iPhoneに保存されている写真」または「その場で撮影した写真」をアプリ内に取り込んで表示する、シンプルなフォトビューアーです。
主な動作の流れは以下の通りです：

	1.	起動時: 画面中央には「写真を選択または撮影してください」というプレースホルダー（仮の表示）が表示されています。
	2.	ライブラリ選択: 「ライブラリ」ボタンをタップすると、iPhoneの標準フォトライブラリが立ち上がり、保存されている画像の中から1枚選ぶことができます。
	3.	カメラ撮影: 「カメラ」ボタンをタップすると、カメラ機能が起動します。写真を撮影して「写真を使用」を選択すると、その画像がアプリのメイン画面に反映されます。
	4.	表示: 選択または撮影された画像は、画面に合わせてリサイズされ、角が丸くなった状態で綺麗に表示されます。
	
<img width="1179" height="2556" alt="image" src="https://github.com/user-attachments/assets/7fb734cc-773d-4418-b5e8-845df6bfde4a" />

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)
```

**何をしているか：**
iOSの写真ライブラリを開いて、写真に選択ができる

**なぜこう書くのか：**
SwiftUIのフレームワークに提供しているから、こうしか書けない

**もしこう書かなかったら：**
こう書かなかったら、写真を選択できないかな

---

### 画像の非同期読み込み

```swift
    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
```

**何をしているか：**

`PhotosPicker` で選択されたアイテム（PhotosPickerItem）から、実際の画像データを取り出し、画面に表示できる形式（Image）に変換しています。

**なぜこう書くのか：**
1.	「待機」が必要だから (await): 高画質な写真はデータ容量が大きく、読み込みに時間がかかります。`await` を使うことで、読み込みが終わるまでプログラムの実行をスマートに待機させることができます。
2.	アプリを固まらせないため (async): この処理を「非同期（async）」にすることで、重い画像の読み込み中であっても、ユーザーが画面をスクロールしたり他の操作をしたりすることを妨げないようにしています。
   
**もしこう書かなかったら：**

 - フリーズ（画面の固まり）: メインスレッド（画面描画を担当する場所）でこの重い処理を行うと、画像を表示するまでの数秒間、ボタンが反応しなくなったり画面が全く動かなくなったりします。
 - アプリの強制終了: 読み込みに失敗した際の処理（catch）を書かないと、予期せぬデータが入ってきた時にアプリがクラッシュする原因になります。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
}
```

**何をしているか：**

SwiftUIには「カメラ画面」という部品がまだ用意されていないため、**昔からあるUIKitのカメラ画面（`UIImagePickerController`）を、SwiftUIで使えるように「箱詰め」**しています。

**なぜこう書くのか：**

SwiftUIとUIKitは仕組みが違うからです。そのままでは会話できないため、この UIViewControllerRepresentable という**共通のプロトコル**に従って書くことで、SwiftUIの画面の一部としてカメラを表示できるようになります。

**もしこう書かなかったら：**

SwiftUIはカメラの標準UIを数行で呼び出す機能がないため、アプリにカメラ機能を組み込むことができません。

---

### Coordinatorパターン

```swift
class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
	let parent: CameraView

	init(_ parent: CameraView) {
		self.parent = parent
	}

	func imagePickerController(
		_ picker: UIImagePickerController,
		didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
	) {
		if let image = info[.originalImage] as? UIImage {
			parent.capturedImage = image
		}
		parent.dismiss()
	}

	func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
		parent.dismiss()
	}
}
```

**何をしているか：**

カメラ側から届く「写真が撮れたよ！」という通知を受け取って、SwiftUI側に中継する「連絡係」です。

**なぜこう書くのか：**

SwiftUI（構造体）は、UIKitからの「撮影完了」という通知を直接受け取ることができないというルールがあるからです。通知を受け取れる専用の「クラス」としてこの `Coordinator` を用意します。

**もしこう書かなかったら：**

シャッターボタンを押しても「写真データがアプリに届かない」、かつ「撮影後にカメラ画面が閉じない」という、動かないボタンになってしまいます。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `CIFilter` | 画像処理にとても便利なフィルタ一 | `CIFilter.sepiaTone() / CIFilter.photoEffectMono()` |
| `loadTransferable` | ファイルや画像を読み込む処理を行うときに便利なの | `await item.loadTransferable(type: Data.self)` |
| `createCGImage` | Core Image用の画像情報(CIImage)から表示用の画像(CGImage)を生成する | context.createCGImage(output, from: ciImage.extent) |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：反転のフィルターを追加
 
  ```
	case .invert:
	let filter = CIFilter.colorInvert()
	filter.inputImage = inputImage
	return filter.outputImage
  ```
- 結果：予想通り動作した
- わかったこと：何も

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

カメラはまだUIKitを使わなきゃいけないのか

   **得られた理解：**

写真選択（PhotosPicker）はSwiftUIで一瞬だけど、生カメラビューはまだ裏でUIKitの力を借りる必要がある。

2. **質問：**

4つのImage型の違いについて

   **得られた理解：**

「画面表示用の Image」「加工用の CIImage」「保存用の UIImage」のように、適材適所で型を変換しながらデータがリレーのバトンのように渡されているイメージが掴めた。

3. **質問：**

サムネイル処理の負荷について

   **得られた理解：**

負荷がデカいの可能性があるなら、動作遅くなるに注意しなきゃ

## この章のまとめ

SwiftUIで画像加工（CoreImage）を扱うときは、「データフローの整理」と「画像のサイズ（解像度）」に命をかけること！
画面に表示するだけならSwiftUIの Image でいいけれど、裏でエフェクトをかけるときは CIImage に変換し、それを CIContext でレンダリングするという一連の儀式が必要。
性能の問題は大事に注意を払わなきゃ

