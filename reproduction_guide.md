# 常磐アジャイル登壇スライド 再現・修正指示書

本ドキュメントは、[code_artifact.html](file:///Users/haradashoukyuu/git/speaker_slide/tohoku-it-bussanten2026/code_artifact.html) が万が一破損したり、一から作り直す必要が生じた際に、これまでのすべての修正指示（レイアウト修復、100名会場向け視認性向上、フルスクリーンスケーリング、ホバー強調など）を完全に再現するための手順とコード設計を記録したものです。

---

## 1. プレゼンテーション用スケーリング＆フルスクリーンシステム

プロジェクターや液晶画面の解像度（4:3, 16:9, 16:10など）を問わず、スライドのアスペクト比を16:9で完全に維持しながら画面に最大化フィットさせるため、**CSS Transform Scale** による相似リサイズを導入しています。

### A. CSSスタイル設計
`<style>` タグ内に以下を定義し、スライドステージを `1280px × 720px` に固定します。

```css
body {
    font-family: 'Noto Sans JP', sans-serif;
    background-color: #0b0f19;
    color: #1f2937;
    overflow: hidden;
    margin: 0;
    padding: 0;
    width: 100vw;
    height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
}

/* 16:9 の論理サイズで完全固定 */
.slide-container {
    width: 1280px;
    height: 720px;
    background-color: #f8f9fa;
    flex-shrink: 0;
    transform-origin: center center;
    transition: transform 0.1s cubic-bezier(0.4, 0, 0.2, 1);
}

.slide {
    display: none;
    opacity: 0;
    transition: opacity 0.2s ease-in-out;
    height: 100%;
}

.slide.active {
    display: flex;
    opacity: 1;
}

/* ホバー時の拡大（マイクロアニメーション） */
.diagram-node {
    transition: transform 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

.diagram-node:hover {
    transform: scale(1.03);
}

/* フルスクリーン時の丸角・枠線のリセット */
.slide-container:fullscreen {
    border: none;
    border-radius: 0;
    box-shadow: none;
}
```

### B. フッターのフルスクリーンボタン
フッター（`<footer>`）の右端に「全画面」ボタンを追加します。

```html
<div class="flex items-center space-x-2">
    <button onclick="toggleSlideListModal()"
        class="px-3.5 py-1.5 rounded-lg bg-gray-100 text-gray-700 hover:bg-gray-200 transition-colors text-sm font-bold flex items-center shadow-sm">
        <i class="fa-solid fa-list mr-1.5"></i> 一覧
    </button>
    <button onclick="toggleFullscreen()"
        class="px-3.5 py-1.5 rounded-lg bg-brand-orange text-white hover:bg-orange-700 transition-colors text-sm font-bold flex items-center shadow-sm">
        <i id="fs-icon" class="fa-solid fa-expand mr-1.5"></i> 全画面
    </button>
</div>
```

### C. JavaScript 制御ロジック
スクリプトの末尾に、リサイズ計算、フルスクリーン API 連携、および **Fキー / Enterキー** での全画面トグル、左右矢印・スペースキーでのスライド送りを実装します。

```javascript
// スライドのリサイズ・フィット計算
function resizeSlides() {
    const container = document.querySelector('.slide-container');
    if (!container) return;

    const baseW = 1280;
    const baseH = 720;

    const isFullscreen = !!(document.fullscreenElement || document.webkitFullscreenElement || document.mozFullScreenElement || document.msFullscreenElement);
    
    // macOS等のフルスクリーン切替ラグに対応するため、全画面時はscreenの絶対サイズを使用
    let winW = window.innerWidth;
    let winH = window.innerHeight;
    
    if (isFullscreen) {
        winW = screen.width;
        winH = screen.height;
    }

    const scaleX = winW / baseW;
    const scaleY = winH / baseH;
    
    const margin = isFullscreen ? 1.0 : 0.96;
    const scale = Math.min(scaleX, scaleY) * margin;
    
    container.style.transform = `scale(${scale})`;
}

// フルスクリーン切り替え
function toggleFullscreen() {
    const container = document.querySelector('.slide-container');
    if (!document.fullscreenElement) {
        container.requestFullscreen().then(() => {
            document.getElementById('fs-icon').className = "fa-solid fa-compress mr-1.5";
            // 段階的にリサイズを実行し、OSのアニメーション完了後に最大サイズを確保する
            setTimeout(resizeSlides, 50);
            setTimeout(resizeSlides, 200);
            setTimeout(resizeSlides, 500);
            setTimeout(resizeSlides, 800);
        }).catch(err => {
            console.error(`Error: ${err.message}`);
        });
    } else {
        document.exitFullscreen().then(() => {
            document.getElementById('fs-icon').className = "fa-solid fa-expand mr-1.5";
            setTimeout(resizeSlides, 50);
            setTimeout(resizeSlides, 200);
            setTimeout(resizeSlides, 500);
        });
    }
}

// Escキー等によるフルスクリーン解除の監視
document.addEventListener('fullscreenchange', () => {
    const isFullscreen = !!document.fullscreenElement;
    const fsIcon = document.getElementById('fs-icon');
    if (fsIcon) {
        fsIcon.className = isFullscreen ? "fa-solid fa-compress mr-1.5" : "fa-solid fa-expand mr-1.5";
    }
    resizeSlides();
    setTimeout(resizeSlides, 200);
    setTimeout(resizeSlides, 500);
});

// キーボードナビゲーション
document.addEventListener('keydown', (e) => {
    if (e.key === 'ArrowRight' || e.key === ' ' || e.key === 'PageDown') {
        nextSlide();
    } else if (e.key === 'ArrowLeft' || e.key === 'PageUp') {
        prevSlide();
    } else if (e.key === 'f' || e.key === 'F') {
        toggleFullscreen();
    }
});

// 初期化とリサイズ登録
updateSlide();
resizeSlides();
window.addEventListener('resize', resizeSlides);
window.addEventListener('load', resizeSlides);
```

---

## 2. 全20枚のスライドインデックスと整合性の連番化

スライドめくり時のカテゴリ番号を完全に一致させ、整合性を保つための構成テーブルです。

*   **総ページ数**: `20`
*   **JS titles配列定義**:
    ```javascript
    const titles = [
        "タイトル", "自己紹介", "本セッションのテーマ", "本日のアジェンダ",
        "前提①「関係人口」", "前提②「アジャイルコミュニティ」", "常磐アジャイルの誕生",
        "立ち上げストーリー①出会い", "立ち上げストーリー②発足", "立ち上げメンバーの構成",
        "立ち上げ後に起きた「変化」", "理論① エフェクチュエーション", "理論② 5つの基本原則",
        "理論③ 常磐アジャイルでの体現", "理論④ 弱い結びつきの強み", "大切にしていること",
        "これからの課題とNext Step", "まとめ", "Call to Action", "Q&A / 最後に"
    ];
    ```

### スライドヘッダーの連番マッピング：

| No. | コメントタグ | 左上カテゴリ表記 (`<span>` 内) | タイトル |
| :--- | :--- | :--- | :--- |
| 1 | `SLIDE 1` | (なし) | タイトルスライド |
| 2 | `SLIDE 2` | `02. INTRODUCTION` | 自己紹介 |
| 3 | `SLIDE 3` | `03. MISSION` | 本セッションのテーマ |
| 4 | `SLIDE 4` | `04. AGENDA` | 本日のアジェンダ |
| 5 | `SLIDE 5` | `05. PREMISE 1` | 前提整理①「関係人口」とは？ |
| 6 | `SLIDE 6` | `06. PREMISE 2` | 前提整理②「アジャイルコミュニティ」とは？ |
| 7 | `SLIDE 7` | `07. ORIGIN & SCOPE` | 常磐アジャイルの誕生 |
| 8 | `SLIDE 8` | `08. ORIGIN STORY 1a` | 常磐アジャイルの立ち上げストーリー① |
| 9 | `SLIDE 9` | `09. ORIGIN STORY 1b` | 常磐アジャイルの立ち上げストーリー② |
| 10 | `SLIDE 10` | `10. ORIGIN STORY 2` | 神輿を担ぎに集まった仲間たち |
| 11 | `SLIDE 11` | `11. CHANGES` | 立ち上げ後に起きた「変化」 |
| 12 | `SLIDE 12` | `12. THEORY 1` | 立ち上がり＝「エフェクチュエーション」の体現 |
| 13 | `SLIDE 13` | `13. THEORY 2` | エフェクチュエーションの5つの基本原則 |
| 14 | `SLIDE 14` | `14. THEORY 3` | エフェクチュエーションの原則と常磐アジャイル |
| 15 | `SLIDE 15` | `15. THEORY 4` | 地域コミュニティにおける「弱い結びつきの強み」 |
| 16 | `SLIDE 16` | `16. VALUES` | コミュニティ運営で「大切にしていること」 |
| 17 | `SLIDE 17` | `17. CHALLENGES` | コミュニティの課題とこれから |
| 18 | `SLIDE 18` | `18. SUMMARY` | 本日のまとめ |
| 19 | `SLIDE 19` | `19. CALL TO ACTION` | 関係人口として、コミュニティへ |
| 20 | `SLIDE 20` | `20. OUTRO` | Q & A / 最後に |

---

## 3. 主要スライドのデザイン・レイアウト変更

### A. タイトルスライド (Slide 1) - 登壇者情報追加
本番でそのまま使えるようにするため、サブタイトルの整理と発表者クレジットを追加。
```html
<div class="slide active flex-col justify-center items-center text-center h-full space-y-6">
    <div class="inline-block px-4 py-1.5 rounded-full bg-brand-yellowBg text-brand-amber font-bold text-sm md:text-base border border-amber-200 shadow-sm">
        東北IT物産展2026 登壇資料
    </div>
    <div class="space-y-3">
        <h1 class="text-4xl md:text-5xl font-black text-brand-green leading-tight max-w-4xl tracking-tight">
            ITコミュニティが紡ぐ関係人口
        </h1>
        <p class="text-xl md:text-2xl font-extrabold text-brand-orange">
            〜アジャイルコミュニティがいわきで立ち上がった話〜
        </p>
    </div>
    <div class="my-3 py-2 flex items-center justify-center">
        <div class="relative p-2 bg-white rounded-2xl shadow-md border-2 border-brand-green/20">
            <img src="jouban.webp" alt="常磐アジャイル" class="h-28 md:h-36 object-contain">
        </div>
    </div>
    <div class="text-gray-700 space-y-1 mt-2">
        <p class="text-xs md:text-sm text-gray-500 font-bold tracking-widest uppercase">Presenter</p>
        <p class="text-lg md:text-xl font-black text-brand-green">原田 頌久 <span class="text-sm font-bold text-gray-600">/ 常磐アジャイル</span></p>
    </div>
</div>
```

### B. 自己紹介スライド (Slide 2) - 100人会場向け視認性向上
遠くの席（100名規模の会場）からでも完全に文字が読めるよう、カードを3枚から2枚（本業 ⇄ 東北・福島のご縁）へ統合し、文字サイズを **`text-lg` (18px) 〜 `text-xl` (20px)** にまで大幅に拡大。またアバターサイズも `w-72` に拡張し、プレースホルダーテキストを排除。
*   **名前**: `原田 頌久` (Harada Nobuhisa / Shoukyuu) (※システムのRealNameおよびニックネーム `nobu-harady` に合わせ正しい漢字表記を徹底)

### C. 前提整理①「関係人口」 (Slide 5)
*   **文字のバックカラー（背景色）の削除**: サブタイトルテキストの黄色背景（`bg-brand-yellowBg px-2 py-0.5 rounded text-brand-amber`）を完全に消去し、シンプルな `text-brand-orange underline` に変更。
*   **改行余白の確保**: タイトルとの密着を防ぐため、段落に `mt-8` を追加して改行余白を挿入。

### D. そもそも「アジャイル」とは？ (Slide 6)
*   **直感的なアプローチ対比＆インラインSVGイラスト**: アジャイル未経験者向けに、従来型の「計画主導（ウォーターフォール）」と「適応・対話主導（アジャイル）」の２つのアプローチの違いを比較カードとして並置。各カード内に、階段状の一方通行プロセスを示すグレーのSVG（ウォーターフォール）と、回転ループから段階的成果物（Ver1, Ver2, 完成版）が飛び出すオレンジ・グリーンのSVG（アジャイル）をインライン配置し、視覚的な直感性を大幅に向上。
*   **アジャイルの広がりに関する解説**: 下部に、アジャイルが現在「ソフトウェア開発手法」の枠を超え、チームビルディングや組織づくりのマインドセットへと拡張されている旨の解説カードを追加。

### E. 常磐アジャイルの誕生 (Slide 8)
*   **「常磐道沿線・交流」の中継カード化**: 左右のカードと一貫性を持たせるため、オレンジ色の点線ボーダー（`border-2 border-dashed border-brand-orange/50 bg-orange-50/40 rounded-xl`）で囲まれた「中継カード」に変更。
*   **Connpassコミュニティプレビューの追加＆クリックリンク化**: 下部フッターに、コアメッセージカードと横並びで、実際のConnpassページプレビュー画像（`jouban1.png`）を白枠・シャドウ付きのカードとして配置。カード全体をクリックすると別タブで `https://jouban-agile.connpass.com/` にジャンプするリンク（`<a>`タグ）として実装。アイコンも付与。

### F. 立ち上げ後に起きた「変化」 (Slide 12)
*   **「架け橋」の視覚的強調**: 右側「地域愛と関わり方の壁」の図において、灰色の障害壁の上に「アジャイルコミュニティという架け橋」が架かっている様子を表現するため、**点線のアーチブリッジ**（`border-t-4 border-dashed border-brand-orange/60 rounded-t-full`）を CSS で絶対配置。
*   **名前ブレスト候補リストの追加**: 左側カラム「名前決定前の噂の拡散」の最下部に、当時ブレストされた個性豊かなコミュニティ名候補（ままどおるアジャイル、ハワイアンジャイル、いなわすくらむ等）をカラフルなバッジ風タグとして可愛く散りばめる。
*   **残骸のクリーンアップ**: 以前のマージ時に生じていた Slide 10 のマークアップの重複残骸を完全に削除し、DOM構造をクリーンアップ。

### G. エフェクチュエーションの原則体現 (Slide 15)
*   **5原則すべての完全カバー＆デザイン統一**: これまで4つしか掲載されていなかった原則に「**5. 飛行機の中のパイロットの原則**」を追加。さらに、Slide 13と完全に同じ2段グリッドレイアウト（原則1〜3は上段上部極太ボーダー、原則4〜5は下段左側極太ボーダー）と配色（緑、オレンジ、アンバー）を移植し、視覚的な一貫性を完璧に担保。

### H. メンバーの構成 (Slide 11)
*   **文字被りの解消と3カラムフレックス化**: 以前は左右の担ぎ手カードが絶対配置によるマイナスマージンで中央と被っていたバグを排除。通常の横並びフレックスボックス（`flex-row justify-center gap-10`）へ移行し、絶対に重ならない構造にリファクタリング。

### I. 「弱い結びつきの強み」理論 (Slide 16)
*   **理論概要テキストの追加**: スライド中央部に「弱い結びつきの強み」の社会学的概要を説明するカードを新規挿入。
*   **アーチ線と文字の被り解消**: 「アジャイル学びの橋」SVGにおいて、放物線とテキストが重なっていた座標バグを修正（放物線を下へ、文字をその上へ配置）。
*   **中央寄せの改善**: 2枚のカードが横に広がりすぎないよう最大幅を絞り、画面の横方向中央に美しく整列。

### J. 仲間募集 (Slide 20)
*   **本物のQRコード画像とリンクの挿入**: プレースホルダーを廃止し、実際のConnpassページへアクセス可能なQRコード（`qr_code.png`）を配置。ホバーアニメーションやクリック時の別タブ起動リンクも実装。

### K. Q&A / 最後に (Slide 21)
*   **登壇者X（Twitter）アカウント＆QRコードの挿入**: `your_account` となっていたプレースホルダーを廃止し、本名 `原田 頌久 (nobu-harady)` とXのハンドルネーム `@nb_rady` を挿入。さらに、`https://x.com/nb_rady` に直接アクセス・遷移できるQRコード（`x_qr.png`）を白枠カードの右側に美しく配置し、ホバー・クリック連動を実装。

---

## 4. ホバー時のインタラクティブ強調 (`diagram-node` の全域適用)

スライド内のほぼすべての主要オブジェクト（自己紹介、前提比較、タイムラインステップ、メンバー図、理論の対比、5つの原則、3つのバリュー、課題ロードマップなど）に `diagram-node` クラスを追加しました。
これにより、マウスカーソルが重なった際に**全てのカードが心地よく滑らかにズーム（`scale(1.03)`）され、視覚的に生き生きとしたダイナミックデザイン**を体験できます。

---

## 5. 登壇後のスライド公開・PDF化手順

登壇終了後、スライド資料の外部公開およびPDF配布（SpeakerDeckへの投稿など）を行う手順です。

### A. GitHub PagesでのWebスライドショー公開
リポジトリ直下の `index.html`（`code_artifact.html` の複製）をGitHub Pagesにデプロイすることで、誰でもブラウザ上で直接インタラクティブなスライドショーを閲覧できます。

1. **GitHubリポジトリにプッシュ**:
   `index.html` が含まれる最新の状態でリモートの `main` ブランチにプッシュされていることを確認します。
2. **GitHubリポジトリ設定を開く**:
   GitHub上の対象リポジトリ（`tohoku-it-bussanten-2026`）のメニューから **Settings（設定）** ➔ **Pages** に移動します。
3. **Build and deployment 設定**:
   - **Source**: `Deploy from a branch` を選択します。
   - **Branch**: `main` ブランチ、および対象ディレクトリを `/ (root)` に指定し、**Save** をクリックします。
4. **公開の確認**:
   数分後に `https://nobu-harady.github.io/tohoku-it-bussanten-2026/` のURLでスライドショーが世界に一般公開されます。

### B. PDFの書き出し手順 (SpeakerDeck投稿用)
CSSの印刷用レイアウト（`@media print`）が実装済みのため、ブラウザの印刷機能を使うだけで崩れのない高品質な16:9比率のPDFが作成されます。

1. **スライドをブラウザで開く**:
   ローカルまたはGitHub Pagesの公開スライドを Chrome 等のブラウザで開きます。
2. **印刷ダイアログを起動**:
   キーボードショートカット `Cmd + P` (Mac) または `Ctrl + P` (Windows) を押して印刷画面を表示します。
3. **印刷パラメータの設定**:
   - **送信先（プリンター）**: `PDFに保存` または `PDFとして保存` を選択します。
   - **レイアウト**: `横`（Landscape）に設定します。
   - **用紙サイズ**: `A4`（またはカスタム等）を選択します。
   - **マージン（余白）**: `なし`（None）に設定します（※超重要）。
   - **オプション（背景のグラフィック）**: `背景のグラフィック` に必ず**チェックを入れる**（ONにする）ことで、美しいダークブルー背景（`#0b0f19`）や配色カラーが維持されます。
4. **PDFの保存**:
   **保存**ボタンを押し、ファイル名（例: `tohoku_it_bussanten_slide.pdf`）を指定してローカルに保存します。これでSpeakerDeckにそのままアップロード可能なPDFが手に入ります。
