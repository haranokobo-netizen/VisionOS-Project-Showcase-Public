# VisionOS-Project-Showcase-Public

🎵 空間マルチメディア・プレーヤー (Spatial Multi-Media Player)

このプロジェクトは、Apple Vision Proの没入型（Immersive）空間において、高精細なVR映像とお気に入りの音楽を同時に楽しむための実装サンプルです。
🌟 プロジェクト概要 視界いっぱいに広がるVRの世界に身をゆだねながら、バックグラウンドで好きな音楽を流すことができるアプリケーションです。RealityKit と AVFoundation を組み合わせることで、視覚と聴覚が調和した、深い没入体験を提供します。

✨ 主な機能 没入型VR再生: RealityKitを使用し、360度の高解像度ビデオ環境を構築。
独立したオーディオ制御: 映像の音とバックグラウンドミュージックを個別に管理し、自分好みのサウンドバランスを実現。
空間オーディオ対応: Appleの空間オーディオ（Spatial Audio）をフル活用し、音がその場に存在するかのような臨場感を演出。
visionOS専用UI: 視線操作（Eye-tracking）とハンドジェスチャーに最適化した、直感的で美しいグラスモーフィズム・デザイン。

🚀 技術的なポイント 使用フレームワーク: RealityKit, SwiftUI, AVFoundation
ターゲット: Apple Vision Pro (visionOS 2.0以上)
実装ロジック: ビデオフレームとオーディオバッファを個別に制御するハイブリッド再生システムを構築し、高いパフォーマンスを維持しています。

###🛠️URLをJSONで読み込むコード

```swift

URLSession.shared.dataTask(with: url) { [weak self] data, _, error in
    // 1. エラーチェックとデータ取り出し
    if error != nil { return }
    guard let data = data else { return }
    
    do {
        // 2. バックグラウンドでJSON解析
        let decoded = try JSONDecoder().decode([MediaItem].self, from: data)
        
        // 3. バックグラウンドで翻訳 & 音楽と映像への振り分けをすべて済ませる
        let localizedList = decoded.map { item in
            MediaItem(
                id: item.id,
                title: NSLocalizedString(item.title, comment: ""),
                urlString: item.urlString,
                isVideo: item.isVideo
            )
        }
        
        // 音楽リスト（!$0.isVideo）の作成
        let audioItems = localizedList.filter { !$0.isVideo }.map { item in
            VideoItem(name: LocalizedStringKey(item.title), url: item.urlString)
        }
        
        // 映像リスト（$0.isVideo）の作成
        let videoItems = localizedList.filter { $0.isVideo }.map { item in
            VideoItem2(name: LocalizedStringKey(item.title), url: item.urlString)
        }
        
        // 4. 完成したリストをメインスレッドに渡して一気に反映
        DispatchQueue.main.async {
            guard let self = self else { return }
            
            self.audioList = audioItems
            self.videoList = videoItems
            
            self.selectedAudio = audioItems.first
            self.selectedVideo = videoItems.first
        }
        
    } catch {
        print("JSONの解析に失敗しました: \(error)")
    }
}.resume()


```

１、バックグラウンドでの非同期処理とデータ成形 (URLSession) ネットワーク経由でJSONデータを取得し、アプリ内で使える形にパース・分類する処理です。

ポイント①：メインスレッドの保護（パフォーマンス対策） Apple Vision Proでは、UIのレンダリングや手のトラッキングを高フレームレートで維持する必要があります。そのため、重い処理（JSONのデコード、filter、map による配列のループ処理など）はすべてバックグラウンドで実行し、最後の画面更新（self.videoList = ...）の瞬間だけ DispatchQueue.main.async を使ってメインスレッドに戻しています。

ポイント②：多言語対応（Localization） NSLocalizedString と LocalizedStringKey を用いることで、サーバーから取得したテキストを端末の言語設定（日本語・英語など）に合わせて動的にローカライズしています。

ポイント③：安全なメモリ管理 クロージャ（非同期処理の塊）の先頭に [weak self] を指定することで、通信中に画面が閉じられた場合などの循環参照（メモリリーク）を安全に防いでいます。

###🛠️VR動画プレーヤーと音楽プレーヤーのセットアップするコード

```swift

// MARK: - Music Player Setup (音楽用)
func setupMusicPlayer(url: URL) {
    let asset = AVURLAsset(url: url)
    let playerItem = AVPlayerItem(asset: asset)
    
    // 音楽の終了通知：終わったら最初からループ再生など
    NotificationCenter.default.addObserver(
        forName: .AVPlayerItemDidPlayToEndTime,
        object: playerItem,
        queue: .main
    ) { [weak self] _ in
        print("音楽が終わりました。ループ再生します。")
    }

    self.musicPlayer = AVPlayer(playerItem: playerItem)
}

// MARK: - Video Player Setup (映像用)
func setupVideoPlayer(url: URL, appModel: AppModel) {
    let asset = AVURLAsset(url: url)
    let playerItem = AVPlayerItem(asset: asset)
    
    // 映像の終了通知：終わったら次のビデオへ切り替え
    NotificationCenter.default.addObserver(
        forName: .AVPlayerItemDidPlayToEndTime,
        object: playerItem,
        queue: .main
    ) { _ in
        print("映像が終了しました。次のビデオへ切り替えます。")
        appModel.playNextVideo()
    }
    
    self.videoPlayer = AVPlayer(playerItem: playerItem)
}


```

２、独立したマルチプレイヤー制御 (AVFoundation) バックグラウンド音楽用の musicPlayer と、VR映像用の videoPlayer を個別にセットアップし、それぞれの「ライフサイクル（終了タイミング）」を制御しています。

ポイント①：NotificationCenter による個別イベント検知 AVPlayerItemDidPlayToEndTime（再生終了の通知）を、それぞれの playerItem にピンポイントで紐付けています。これにより、「音楽が終わったか」「映像が終わったか」を正確に識別できます。

###🛠️VR動画を流すスクリーを設定するコード

```swift

// 1. ビデオ素材（VideoMaterial）の作成
    let material = VideoMaterial(avPlayer: videoPlayer)
    
    // 2. 巨大な球体（360度スクリーン）の作成と素材の適用
    let sphereMesh = MeshResource.generateSphere(radius: 100)
    let entity = ModelEntity(mesh: sphereMesh, materials: [material])
    
    // 3. 反転処理：球体の「内側」に動画が映るようにする（ユーザーが球の中心に立つため）
    entity.scale *= .init(x: -1, y: 1, z: 1)
    
    // 4. RealityKitの空間（RealityView）にエンティティを追加
    content.add(entity)


```

３、RealityKitによる360度空間スクリーンの構築 (RealityKit) 平面の動画を、Vision Proの強みである「没入空間（Immersive Space）」へと拡張する、最もコアなグラフィックス処理です。

ポイント①：VideoMaterial による質感の定義 AVPlayer の映像フレームを、3D空間上のオブジェクトに貼り付けられる「マテリアル（素材）」へとリアルタイムに変換します。

ポイント②：巨大な360度全球スクリーンの生成 MeshResource.generateSphere(radius: 100) を使い、アプリの原点（ユーザーの立ち位置）を中心とした、半径100メートルの巨大な3Dの球体（ジオデシック・ドームのようなもの）をバーチャル空間に浮かび上がらせます。

ポイント③：空間を包み込む「反転（フリップ）」の魔法 通常、3Dの球体は「外側」から眺めるように作られているため、そのままでは内側にいるユーザーから動画が見えません。そこで、entity.scale *= .init(x: -1, y: 1, z: 1)を適用し、X軸（左右）のスケールをマイナスに反転させています。これにより、球体のメッシュ（裏表）が内側にひっくり返り、ユーザーを360度ぐるりと囲む巨大スクリーンが完成します。
後はそれぞれのプレイヤーを稼働させると映像と音楽が流れます。

🎬 アプリケーションの起動 これら3つの準備が整ったあと、それぞれのプレイヤーに対して .play() を実行します。

```swift

// 空間オーディオと共に、視界いっぱいのVR映像と心地よい音楽がシンクロして動き出します
self.videoPlayer?.play()
self.musicPlayer?.play()

```

視覚（RealityKitによる360度映像）と、聴覚（AVFoundationによる空間オーディオとBGM）を、完全に独立したシステムとしてハイブリッド稼働させることで、Vision Proならではの「最高に贅沢で居心地の良いマイ空間」を作り出すことができます。
