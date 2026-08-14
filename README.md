# Git / GitHub 超実践リファレンス
> 超初級者が基礎から理解でき、中級・上級者にとっても即戦力になる最強の手引き

---

## 目次
1. [Gitの本質を理解する（概念マップ）](#1-gitの本質を理解する概念マップ)
2. [環境セットアップ](#2-環境セットアップ)
3. [GitHubへの接続方式完全解説（HTTPS・SSH）](#3-githubへの接続方式完全解説httpsssh)
4. [リポジトリの作成と接続](#4-リポジトリの作成と接続)
5. [毎日使う基本ワークフロー](#5-毎日使う基本ワークフロー)
6. [ブランチ戦略](#6-ブランチ戦略)
7. [マージ・リベース・コンフリクト解消](#7-マージリベースコンフリクト解消)
8. [リモート操作](#8-リモート操作)
9. [やり直し・取り消し（最重要）](#9-やり直し取り消し最重要)
10. [スタッシュ・チェリーピック](#10-スタッシュチェリーピック)
11. [GitHub固有の操作（PR・Issue・Fork）](#11-github固有の操作prissueforк)
12. [GitHub Actions（CI/CD入門）](#12-github-actionscicd入門)
13. [状況別トラブルシューティング](#13-状況別トラブルシューティング)
14. [チーム開発のベストプラクティス](#14-チーム開発のベストプラクティス)
15. [コマンド逆引き辞典](#15-コマンド逆引き辞典)

---

## 1. Gitの本質を理解する（概念マップ）

### Gitは「スナップショット型」のバージョン管理システム

```
作業ディレクトリ  →  ステージング(Index)  →  ローカルリポジトリ  →  リモートリポジトリ
  (Working Tree)        (git add)             (git commit)           (git push)

ファイルを編集する場所   コミット前の準備場所     履歴が積まれる場所       チームと共有する場所
```

### 4つのエリアを意識するだけで9割の操作が理解できる

| エリア | 保存場所 | 操作 |
|---|---|---|
| Working Tree | PC上のファイル | 普通にエディタで編集 |
| Staging Area | `.git/index` | `git add` |
| Local Repository | `.git/objects` | `git commit` |
| Remote Repository | GitHub等のサーバー | `git push` / `git pull` |

### HEAD・ブランチ・コミットの関係

```
      HEAD
       ↓
    master ブランチ
       ↓
[コミットC] ← [コミットB] ← [コミットA]
  (最新)                      (最古)
```

- **コミット**：ファイル群のスナップショット（変更差分ではなく全体像）
- **ブランチ**：特定コミットを指すポインタ（軽量で作成コストゼロ）
- **HEAD**：「今自分がいるブランチ」を指すポインタ

---

## 2. 環境セットアップ

### インストール
```bash
# Windows: https://git-scm.com/download/win からインストーラーを実行
# Mac
brew install git
# Linux (Ubuntu/Debian)
sudo apt install git
```

### 初期設定（最初に必ず実施）
```bash
# 名前とメールアドレスを登録（コミット履歴に刻まれる）
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# デフォルトブランチ名を master に統一
git config --global init.defaultBranch master

# よく使うエイリアス（省略コマンド）を登録
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --all"

# 設定確認
git config --list
```

---

## 3. GitHubへの接続方式完全解説（HTTPS・SSH）

### はじめに：「接続方式」がなぜ重要か

`git push` や `git clone` の裏側では「あなたのPC ↔ GitHubサーバー」間で通信が発生します。この通信に使う「プロトコル（通信規則）」が **HTTPS** と **SSH** の2種類あります。どちらを選ぶかで認証の仕組み・日常の快適さ・セキュリティが大きく変わります。

---

### そもそも「プロトコル」とは何か

コンピュータ同士の通信は「プロトコル」という決め事（ルール）に従っています。

```
人間の世界：  日本語 + 英語 + ...  （言語 = コミュニケーションのルール）
PC の世界：   HTTPS + SSH + ...    （プロトコル = 通信のルール）
```

Git が使う代表的なプロトコル：

| プロトコル | 特徴 | 認証方式 | ポート |
|---|---|---|---|
| **HTTPS** | Webと同じ仕組み。企業FWを通りやすい | ユーザー名 + PAT | 443 |
| **SSH** | セキュアシェル。鍵認証で完全自動化 | 公開鍵暗号 | 22 |
| `git://` | 読み取り専用・非推奨 | なし | 9418 |

---

### HTTPS とは何か（基礎から徹底解説）

#### HTTP と HTTPS の違い

**HTTP（HyperText Transfer Protocol）** はウェブページのデータを送受信する通信規則です。しかし HTTP は**平文（暗号化なし）**のため、通信を傍受されると内容が丸見えになります。

```
HTTP（危険）：
  あなた ──"Password: ghp_xxx"──▶ 中継ルーター（傍受できる！）──▶ GitHub

HTTPS（安全）：
  あなた ──"Kx9#fQ..."（TLS暗号化済み）──▶ 中継ルーター（解読不可）──▶ GitHub
```

**HTTPS（HTTP Secure）** は HTTP に **TLS（Transport Layer Security）** という暗号化レイヤーを被せたものです。

#### TLS とは何か

TLS（旧称 SSL）は「通信相手が本物であることを確認し、通信内容を暗号化する」仕組みです。

```
TLS の2つの役割：
  1. 認証  ─ 「接続先は本物の github.com か？」（サーバー証明書で検証）
  2. 暗号化 ─ 「通信内容を第三者が読めないようにする」
```

- **ポート：443**（HTTP は 80）
- ブラウザの 🔒 マーク = TLS が有効な証拠
- `https://github.com/...` という URL が HTTPS 接続のサイン

#### GitHubがパスワード認証を廃止した背景（2021年8月）

2021年8月13日以降、GitHubは Git 操作でのパスワード認証を無効化しました。

```
❌ 廃止（現在は使用不可）
git push  →  Password: ••••••••  （GitHubアカウントのパスワード）

✅ 現在の正しい方法
git push  →  Password: ghp_xxxxxxxxxxxxxxxxxxxxxx  （Personal Access Token）
```

**廃止理由：**
- パスワードは他サービスとの使い回しによる漏洩リスクが高い
- 盗まれたパスワードは GitHub 以外でも悪用される
- PAT は権限・有効期限を細かく設定でき、不要になったらいつでも無効化できる

---

### Personal Access Token（PAT） — HTTPSの認証キー

#### PAT とは何か

PAT（Personal Access Token = 個人アクセストークン）は「GitHubが発行する、パスワードの代わりに使う認証文字列」です。

```
形式例： ghp_AbCdEfGhIjKlMnOpQrStUvWxYz012345
         ↑ ghp_ プレフィックスで GitHub Personal Token だとわかる
```

PAT には2種類あります：

| 種類 | 特徴 | 推奨度 |
|---|---|---|
| **Fine-grained PAT** | リポジトリ・権限を細かく設定可。有効期限必須 | ◎ セキュリティ重視 |
| **Classic PAT** | 従来形式。設定がシンプル | ○ 手軽に使いたい場合 |

#### Classic PAT の作成手順（GUI）

```
1. GitHub 右上アイコン → Settings
2. 左メニュー最下部 → Developer settings
3. Personal access tokens → Tokens (classic)
4. Generate new token (classic) をクリック
5. Note: 用途メモ（例: "MacBook - daily git operations"）
6. Expiration: 有効期限（90日推奨。"No expiration" は非推奨）
7. スコープ（権限）を選択：
     ✅ repo      → プライベートリポジトリの読み書き（通常はこれだけで十分）
     ✅ workflow  → GitHub Actions の操作が必要な場合
     □  delete_repo → 削除権限（不要なら絶対に付けない）
8. Generate token ボタン
   ⚠️ トークンは一度しか表示されない。必ずコピーして安全な場所に保存！
```

#### PAT を実際に使う

```bash
# HTTPS でクローン（認証を求められる）
git clone https://github.com/USERNAME/REPO.git
# Username: yourname
# Password: ghp_xxxxxxxxxxxxxxxxxxxx  ← ここに PAT を貼り付け

# URL にトークンを埋め込む（履歴に残るので非推奨）
git clone https://yourname:ghp_xxx@github.com/USERNAME/REPO.git
```

---

### Credential Helper — PAT を毎回入力しないために

**Credential Helper** は認証情報をOSのセキュアストレージに保存し、次回から自動入力してくれます。

```bash
# ── Windows ──
# Git for Windows 標準搭載（Windows 資格情報マネージャーに保存）
git config --global credential.helper manager

# ── Mac ──
# macOS キーチェーンに保存
git config --global credential.helper osxkeychain

# ── Linux ──
# メモリに一時保存（デフォルト15分。再起動で消える）
git config --global credential.helper cache
git config --global credential.helper "cache --timeout=86400"  # 24時間に延長

# ディスクに永続保存（~/.git-credentials にプレーンテキスト保存）
git config --global credential.helper store
# ⚠️ セキュリティリスクあり。個人の信頼できるPCのみで使用すること
```

**GitHub CLI を使った認証（最も簡単・推奨）：**

```bash
# インストール後、対話的に認証設定（https://cli.github.com/）
gh auth login
# ? What account do you want to log into? → GitHub.com
# ? What is your preferred protocol?       → HTTPS  (or SSH)
# ? Authenticate Git with your GitHub credentials? → Yes
# → ブラウザが開いてOAuth認証 → 以降 git push/pull が自動認証
```

---

### SSH とは何か（基礎から徹底解説）

**SSH（Secure Shell）** は「遠隔地のサーバーに安全に接続するための通信プロトコル」です。

```
あなたのPC  ──── 暗号化されたトンネル ────  GitHubのサーバー
            （SSH プロトコル、ポート22）
```

元々はサーバー管理者がリモートサーバーをコマンドラインで操作するためのツールでしたが、Git/GitHub の文脈では「コードのやり取りを安全に行う認証・通信方式」として広く使われています。

---

### HTTPS vs SSH — 徹底比較

| 比較項目 | HTTPS | SSH |
|---|---|---|
| URL形式 | `https://github.com/user/repo.git` | `git@github.com:user/repo.git` |
| 認証方法 | ユーザー名 + PAT | 秘密鍵ファイル |
| 初期設定の手間 | 少ない（PATをコピーするだけ） | やや多い（鍵生成・登録が必要） |
| 毎回の認証 | Credential Helper で自動化可 | 鍵登録後は完全自動 |
| PAT / 鍵の有効期限 | PATは定期更新が必要 | 鍵は削除しない限り永続 |
| セキュリティ | PAT漏洩リスクあり（管理が必要） | 秘密鍵が手元にある限り安全 |
| ファイアウォール | 通りやすい（ポート443） | 稀にブロックされる（ポート22） |
| 他人のPCでの作業 | ✅ 適している | ❌ 鍵を持ち込みたくない |
| **日常開発の快適さ** | ○（Credential Helper設定後） | **◎（設定後は完全ノーストレス）** |

> **結論：日常開発なら SSH が快適。一時的な利用・企業ネット環境は HTTPS が適切。**

---

### SSH 公開鍵認証の仕組み（なぜパスワード不要で安全なのか）

SSH は「**公開鍵暗号**」という数学的仕組みを使います。

```
【鍵ペアの生成】
ssh-keygen を実行すると2つのファイルが作られる：

  ~/.ssh/id_ed25519      ← 秘密鍵（絶対に他人に渡してはいけない。金庫の鍵）
  ~/.ssh/id_ed25519.pub  ← 公開鍵（GitHubに登録する。南京錠そのもの）
```

**認証の流れ（チャレンジ＆レスポンス方式）：**

```
1. あなたが git push しようとする
         ↓
2. GitHubが「乱数（チャレンジ）」を送ってくる
         ↓
3. あなたのPCが「秘密鍵」でその乱数に署名して返す
         ↓
4. GitHubが「公開鍵」で署名を検証
         ↓
5. 「本人確認OK！」→ 接続許可
```

ポイント：**秘密鍵はあなたのPCから一切外に出ない**。パスワードのように「送信して照合」しないので傍受されても意味がない。

---

### なぜ Ed25519 アルゴリズムを使うのか

SSH 鍵には複数の暗号アルゴリズムがあります：

| アルゴリズム | 鍵長 | 速度 | セキュリティ | 推奨度 |
|---|---|---|---|---|
| RSA | 4096bit必要 | 遅い | 古い設計 | △（古い環境との互換用） |
| ECDSA | 256bit | 速い | 良い | ○ |
| **Ed25519** | **256bit** | **最速** | **最高水準** | **◎ 現在の標準** |

`Ed25519` は「Edwards-curve Digital Signature Algorithm (Curve25519)」の略。短い鍵で RSA-4096 より高いセキュリティを実現します。**特別な理由がない限り Ed25519 一択**です。

---

### SSH 鍵の設定（GitHubへの認証）— 完全手順

#### Step 1：鍵ペアを生成する

```bash
# -t: アルゴリズム指定  -C: 鍵のコメント（識別用。メールアドレスが一般的）
ssh-keygen -t ed25519 -C "you@example.com"

# 実行すると以下のプロンプトが出る：
# Enter file in which to save the key (~/.ssh/id_ed25519):  → Enter で既定パスに保存
# Enter passphrase (empty for no passphrase):               → 空でもOK（後述）
# Enter same passphrase again:                              → 同じものを入力
```

> **パスフレーズについて：**
> - 設定すると秘密鍵自体が暗号化され、PCが盗まれても鍵が使えない
> - 設定しないと push のたびにパスフレーズ入力が不要で快適
> - **個人PCなら空で問題なし。会社のPCや共有環境なら設定推奨**

#### Step 2：生成されたファイルを確認する

```bash
# Mac / Linux
ls -la ~/.ssh/
# id_ed25519      ← 秘密鍵（パーミッション 600 であること）
# id_ed25519.pub  ← 公開鍵（GitHubに登録する）

# Windows (PowerShell)
dir $env:USERPROFILE\.ssh\
```

#### Step 3：公開鍵を表示してコピーする

```bash
# Mac / Linux
cat ~/.ssh/id_ed25519.pub

# Windows (PowerShell)
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub

# 出力例（これを丸ごとコピー）：
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIxxxxxxxxxxxxxxxxxxxxxxxx you@example.com
```

#### Step 4：GitHubに公開鍵を登録する（GUI）

```
1. GitHub にログイン
2. 右上のアイコン → Settings
3. 左メニュー → SSH and GPG keys
4. 「New SSH key」ボタン
5. Title: 鍵の識別名（例: "MacBook Pro 2025"、"会社PC" など）
6. Key type: Authentication Key（デフォルト）
7. Key: コピーした公開鍵を貼り付け
8. 「Add SSH key」ボタン
```

#### Step 5：SSH エージェントに秘密鍵を登録する（パスフレーズ設定時）

```bash
# ssh-agent を起動
eval "$(ssh-agent -s)"

# 秘密鍵を登録（以降パスフレーズ入力が不要になる）
ssh-add ~/.ssh/id_ed25519

# Windows (PowerShell)
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

#### Step 6：接続確認

```bash
ssh -T git@github.com
# 初回: "Are you sure you want to continue connecting?" → yes
# 成功: "Hi username! You've successfully authenticated, but GitHub does not provide shell access."
```

---

### 接続方式を後から切り替える

既存リポジトリの接続方式は `git remote set-url` でいつでも変更できます。

```bash
# 現在の接続方式を確認
git remote -v
# origin  https://github.com/USERNAME/REPO.git (fetch)  ← HTTPS 接続中
# origin  git@github.com:USERNAME/REPO.git (fetch)       ← SSH 接続中

# HTTPS → SSH に変更
git remote set-url origin git@github.com:USERNAME/REPO.git

# SSH → HTTPS に変更
git remote set-url origin https://github.com/USERNAME/REPO.git
```

---

### 使う場面・使わない場面

| 場面 | SSH | HTTPS | 理由 |
|---|---|---|---|
| 毎日 push/pull する | ✅ 推奨 | △ | 認証が完全自動で快適 |
| CI/CD（GitHub Actions等） | ○ | ✅ 推奨 | PAT をシークレットとして安全管理 |
| 他人のPCで一時作業 | ❌ | ✅ 推奨 | SSH鍵を持ち込みたくない |
| 企業のプロキシ・厳格なFW環境 | △ | ✅ 推奨 | ポート22がブロックされることがある |
| パブリックOSSのクローンのみ | ❌ | ✅ 推奨 | 書き込みしないなら認証不要 |
| 本番サーバーへのデプロイ | ✅ 必須 | △ | SSH でサーバーに直接ログイン |

---

### 複数の SSH 鍵を使い分ける（上級）

仕事用と個人用で GitHub アカウントが異なる場合など：

```bash
# 鍵を別名で生成
ssh-keygen -t ed25519 -C "work@company.com"    -f ~/.ssh/id_ed25519_work
ssh-keygen -t ed25519 -C "personal@gmail.com" -f ~/.ssh/id_ed25519_personal
```

```
# ~/.ssh/config の内容（ファイルが存在しない場合は新規作成）
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
```

```bash
# リポジトリの remote URL を Host 名に合わせる
git remote set-url origin git@github-work:company/repo.git      # 仕事用
git remote set-url origin git@github-personal:yourname/repo.git # 個人用

# 接続確認
ssh -T git@github-work      # → Hi work_username!
ssh -T git@github-personal  # → Hi personal_username!
```

---

### よくあるエラーと対処法

```bash
# ❌ remote: Support for password authentication was removed on August 13, 2021
# → PAT を使っていない。Credential Helper を設定して PAT を入力する
git config --global credential.helper manager  # Windows
# 次の git push/pull で PAT を入力 → Credential Helper が自動保存

# ❌ Permission denied (publickey)
# → 公開鍵がGitHubに登録されていない、または秘密鍵が見つからない
ssh -vT git@github.com   # -v でデバッグ情報を表示して原因を特定
ssh-add -l               # 登録済みの秘密鍵一覧を確認

# ❌ Host key verification failed
# → ~/.ssh/known_hosts の古いエントリが邪魔している
ssh-keygen -R github.com  # known_hosts から GitHub のエントリを削除
ssh -T git@github.com     # 再接続（yes で新しいフィンガープリントを登録）

# ❌ port 22: Connection timed out（企業FWでポート22がブロック）
# → ~/.ssh/config に以下を追加（HTTPS ポートで SSH を使う奥の手）
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
```

---

## 4. リポジトリの作成と接続

### パターンA：ローカルで新規作成してGitHubへ

```bash
mkdir my-project && cd my-project
git init                          # .git ディレクトリが作成される
git add .
git commit -m "first commit"
git branch -M master              # ブランチ名を master に統一
git remote add origin git@github.com:USERNAME/REPO.git
git push -u origin master         # -u で上流を設定（以降 git push だけでOK）
```

### パターンB：GitHubから既存リポジトリをクローン

```bash
git clone git@github.com:USERNAME/REPO.git
cd REPO                           # 自動でリモートが設定済み
```

### リモートの確認・変更

```bash
git remote -v                     # 現在のリモートURL確認
git remote set-url origin git@github.com:USERNAME/NEW-REPO.git  # URL変更
git remote add upstream git@github.com:ORIGINAL/REPO.git        # Fork元を追加
```

---

## 5. 毎日使う基本ワークフロー

```
[編集] → git add → git commit → git push → （PRを作成 or そのまま完成）
```

### ファイルのステータスを確認する

```bash
git status                        # 変更ファイルの一覧
git status -s                     # 短縮表示
git diff                          # Working Tree と Staging の差分
git diff --staged                 # Staging と 最新コミットの差分
```

### ステージングに追加する（git add）

```bash
git add ファイル名                 # 特定ファイルを追加
git add .                         # 全変更を追加
git add -p                        # 変更を塊ごとに選択して追加（強力）
git add src/                      # ディレクトリ以下を全追加
```

### コミットする（git commit）

```bash
git commit -m "コミットメッセージ"
git commit -am "メッセージ"       # add と commit を同時に（追跡済みファイルのみ）
git commit --amend                # 直前のコミットを修正（push前限定）
git commit --amend --no-edit      # メッセージそのままで直前コミットに追記
```

### コミットメッセージの書き方（Conventional Commits）

```
feat: ユーザー登録機能を追加
fix: ログイン時のNullPointerExceptionを修正
docs: README にセットアップ手順を追記
refactor: 認証ロジックを別クラスに分離
test: ユーザーサービスのユニットテストを追加
chore: package.json の依存バージョンを更新
```

### ログを確認する（git log）

```bash
git log                           # 詳細ログ
git log --oneline                 # 1行表示
git log --oneline --graph --all   # ブランチ含む全体をグラフ表示（最強）
git log -p                        # 各コミットの差分も表示
git log --author="名前"           # 特定人物のコミットのみ
git log --since="2024-01-01"      # 日付フィルター
git log --grep="キーワード"       # メッセージ検索
git log ファイル名                # 特定ファイルの変更履歴
```

---

## 6. ブランチ戦略

### ブランチの基本操作

```bash
git branch                        # ローカルブランチ一覧
git branch -a                     # リモート含む全ブランチ一覧
git branch feature/login          # ブランチ作成
git checkout feature/login        # ブランチ切り替え
git checkout -b feature/login     # 作成と切り替えを同時に（よく使う）
git switch -c feature/login       # 上と同じ（新しい書き方）
git branch -d feature/login       # ブランチ削除（マージ済みのみ）
git branch -D feature/login       # 強制削除（未マージでも削除）
git branch -M master              # 現在のブランチを強制リネーム
```

### Git Flow（チーム開発の王道）

```
main (master)  ←─── 本番リリース専用。直接コミット禁止
   ↑
develop        ←─── 開発の統合ブランチ
   ↑
feature/xxx    ←─── 機能開発ブランチ（develop から分岐）
hotfix/xxx     ←─── 本番バグ修正ブランチ（main から分岐）
release/x.x.x  ←─── リリース準備ブランチ（develop から分岐）
```

### シンプルなフィーチャーブランチ戦略（小〜中規模向け）

```bash
# 1. main から作業ブランチを作成
git checkout main
git pull origin main
git checkout -b feature/新機能名

# 2. 作業してコミット
git add .
git commit -m "feat: 〇〇を実装"

# 3. main の最新を取り込んでからプッシュ
git fetch origin
git rebase origin/main            # rebase で履歴をきれいに保つ
git push origin feature/新機能名

# 4. GitHubでPull Requestを作成 → レビュー → マージ
```

---

## 7. マージ・リベース・コンフリクト解消

### マージ（Merge）：2つのブランチを統合する

```bash
git checkout main
git merge feature/login           # fast-forward マージ（履歴が直線になる）
git merge --no-ff feature/login   # マージコミットを必ず作成（履歴に記録が残る）
git merge --squash feature/login  # 相手ブランチのコミットを1つにまとめてマージ
```

| マージ方法 | 特徴 | 使いどき |
|---|---|---|
| fast-forward | 履歴が直線になる | 個人作業・小さな修正 |
| --no-ff | マージコミットが残る | チーム開発・feature ブランチ |
| --squash | 相手の履歴を圧縮 | 細かいコミットを整理してマージ |

### リベース（Rebase）：コミット履歴を整理する

```bash
git checkout feature/login
git rebase main                   # mainブランチの先頭に付け替える

# インタラクティブリベース（コミット履歴の編集）
git rebase -i HEAD~3              # 直近3コミットを対話的に編集
```

**インタラクティブリベースで使うコマンド：**

```
pick   → そのまま使う
reword → メッセージを編集
edit   → コミット内容を編集
squash → 前のコミットに統合（メッセージを結合）
fixup  → 前のコミットに統合（メッセージは破棄）
drop   → コミットを削除
```

### コンフリクト（競合）の解消

```bash
# マージ/リベース中にコンフリクトが発生した場合

git status                        # コンフリクトしているファイルを確認
# ファイルを開くと以下のようなマーカーが表示される：
# <<<<<<< HEAD
# 自分の変更
# =======
# 相手の変更
# >>>>>>> feature/login

# エディタで手動修正してマーカーを削除
git add 修正したファイル
git commit                        # マージの場合
git rebase --continue             # リベースの場合

# やり直す場合
git merge --abort
git rebase --abort
```

### VS Code でのコンフリクト解消（GUI）

VS Code はコンフリクト箇所を自動ハイライトし、以下のボタンで解消できます：
- **Accept Current Change**（自分の変更を採用）
- **Accept Incoming Change**（相手の変更を採用）
- **Accept Both Changes**（両方残す）
- **Compare Changes**（差分比較）

---

## 8. リモート操作

### fetch・pull・push の違い

```
git fetch  → リモートの変更をローカルに「ダウンロード」するだけ（作業ツリーは変わらない）
git pull   → fetch + merge を同時に実行（作業ツリーが変わる）
git push   → ローカルのコミットをリモートへ送信
```

```bash
git fetch origin                  # 全ブランチの最新情報を取得
git fetch origin master           # masterブランチのみ取得

git pull origin master            # fetch + merge
git pull --rebase origin master   # fetch + rebase（履歴をきれいに保つ）

git push origin master            # masterをプッシュ
git push -u origin feature/login  # 上流設定しながらプッシュ
git push origin --delete old-branch # リモートブランチを削除
git push --force-with-lease       # 強制プッシュ（他人の変更を上書きしない安全版）
```

> ⚠️ `git push --force` は共有ブランチでは絶対使用禁止。`--force-with-lease` を使うこと。

### リモートブランチをローカルで追跡する

```bash
git checkout -b feature/login origin/feature/login  # リモートブランチをローカルに作成
git branch --set-upstream-to=origin/master master   # 上流ブランチを手動設定
```

---

## 9. やり直し・取り消し（最重要）

### 状況別の取り消しコマンド早見表

```
変更をまだ add していない → git restore ファイル名
add 済みで commit 前      → git restore --staged ファイル名
commit 済みで push 前     → git reset HEAD~1（または git commit --amend）
push 済み（共有前）       → git push --force-with-lease（慎重に）
push 済み（共有後）       → git revert COMMIT_HASH（履歴を消さずに打ち消す）
```

### git restore（ファイルの変更を元に戻す）

```bash
git restore ファイル名            # Working Tree の変更を破棄
git restore --staged ファイル名   # ステージングから外す（ファイルは変更のまま）
git restore --source=HEAD~2 ファイル名  # 2つ前のコミットの状態に戻す
```

### git reset（コミットを取り消す）

```bash
git reset --soft HEAD~1           # コミットだけ戻す（変更はstaged状態で残る）
git reset --mixed HEAD~1          # コミットとstageを戻す（変更はファイルに残る）← デフォルト
git reset --hard HEAD~1           # コミット・stage・ファイル全てを戻す（変更が消える）
git reset --hard COMMIT_HASH      # 特定コミットまで戻す
```

| オプション | コミット | Staging | Working Tree |
|---|---|---|---|
| --soft | ✅取り消し | そのまま | そのまま |
| --mixed | ✅取り消し | ✅取り消し | そのまま |
| --hard | ✅取り消し | ✅取り消し | ✅取り消し |

### git revert（コミットを「打ち消すコミット」で元に戻す）

```bash
git revert HEAD                   # 直前のコミットを打ち消す新コミットを作成
git revert COMMIT_HASH            # 特定コミットを打ち消す
git revert HEAD~3..HEAD           # 範囲指定で複数を打ち消す
```

> push済みの共有ブランチで間違えたら `git revert` が正解。`git reset` は使わない。

### 消したコミットを復元する（git reflog）

```bash
git reflog                        # HEADの移動履歴を全て表示
# 例: HEAD@{3}: commit: 消したいコミット が見つかったら
git checkout HEAD@{3}             # そのコミットの状態に移動
git reset --hard HEAD@{3}         # その状態に強制リセット
```

---

## 10. スタッシュ・チェリーピック

### git stash（作業を一時避難する）

```bash
# 状況：急なバグ修正が入ったが、現在の作業を中途半端に commit したくない
git stash                         # 現在の変更を退避（Working Tree とStagingが綺麗になる）
git stash push -m "ログイン機能の途中"  # メッセージ付きで退避

git stash list                    # 退避リスト確認
git stash pop                     # 最新の退避を復元して一覧から削除
git stash apply stash@{2}         # 特定の退避を復元（一覧には残る）
git stash drop stash@{0}          # 特定の退避を削除
git stash clear                   # 全退避を削除
git stash branch feature/test     # 退避から新ブランチを作成して復元
```

### git cherry-pick（特定コミットだけを取り込む）

```bash
# 状況：別ブランチの特定コミットだけ自分のブランチに欲しい
git cherry-pick COMMIT_HASH       # 特定コミットを現在のブランチに適用
git cherry-pick A..B              # A の次から B まで複数適用
git cherry-pick --no-commit HASH  # 適用するがコミットはしない（内容確認用）
```

---

## 11. GitHub固有の操作（PR・Issue・Fork）

### Pull Request（PR）の基本フロー

```bash
# 1. ブランチ作成・開発・プッシュ
git checkout -b feature/xxx
# ... 開発 ...
git push -u origin feature/xxx

# 2. GitHub上でPRを作成
#    base: main（マージ先）← compare: feature/xxx（自分のブランチ）
#    タイトル・説明を記入 → Create pull request

# 3. レビュー・修正対応
git add .
git commit -m "fix: レビュー指摘を修正"
git push origin feature/xxx      # PRに自動的に追加される

# 4. マージ（GitHubのボタン or CLI）
gh pr merge --squash             # GitHub CLI を使う場合
```

### Fork & Pull Request（OSS貢献の流れ）

```bash
# 1. GitHubでFork（UIでボタンを押す）

# 2. 自分のForkをクローン
git clone git@github.com:自分のUsername/REPO.git

# 3. 元リポジトリをupstreamとして登録
git remote add upstream git@github.com:元のUsername/REPO.git

# 4. upstreamの最新を定期的に取り込む
git fetch upstream
git merge upstream/main

# 5. 機能開発してForkにプッシュ → 元リポジトリへPRを出す
```

### GitHub CLI（gh）の便利コマンド

```bash
# インストール: https://cli.github.com/
gh auth login                     # 認証

gh repo create my-project --public  # リポジトリ作成
gh pr create                      # PRを対話式で作成
gh pr list                        # PR一覧
gh pr checkout 123                # PR番号でブランチに切り替え
gh pr merge 123 --squash          # PR をマージ
gh issue create                   # Issue作成
gh issue list                     # Issue一覧
gh repo clone USERNAME/REPO       # クローン
```

### .gitignore の設定

```bash
# .gitignore ファイルに管理不要なファイルを記述
node_modules/
.env
*.log
dist/
.DS_Store

# gitignore テンプレートを自動生成
# https://www.toptal.com/developers/gitignore にアクセス
# または
gh api /gitignore/templates/Node | jq -r '.source' > .gitignore
```

---

## 12. GitHub Actions（CI/CD入門）

### 基本構造（`.github/workflows/ci.yml`）

```yaml
name: CI

on:
  push:
    branches: [master, main]
  pull_request:
    branches: [master, main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4       # コードをチェックアウト
      - uses: actions/setup-node@v4     # Node.js をセットアップ
        with:
          node-version: '20'
      - run: npm ci                     # 依存インストール
      - run: npm test                   # テスト実行
      - run: npm run build              # ビルド
```

### よく使うトリガー

```yaml
on:
  push:                         # プッシュ時
  pull_request:                 # PR作成・更新時
  schedule:
    - cron: '0 9 * * 1'        # 毎週月曜9時
  workflow_dispatch:            # 手動実行
```

---

## 13. 状況別トラブルシューティング

### ❌ 間違ったファイルをコミットしてしまった

```bash
# 直前のコミットからファイルを取り除く（pushしていない場合）
git reset HEAD~1
git restore --staged 間違えたファイル
git commit -c ORIG_HEAD           # 元のメッセージを再利用してコミット
```

### ❌ コミットメッセージを間違えた

```bash
git commit --amend -m "正しいメッセージ"   # 直前のみ（push前）
git rebase -i HEAD~3                        # 過去のコミットを修正（push前）
```

### ❌ 間違ったブランチにコミットしてしまった

```bash
# 正しいブランチに移して、間違ったブランチから削除する
git log --oneline -3              # コミットハッシュを確認
git checkout 正しいブランチ
git cherry-pick COMMIT_HASH       # 正しいブランチに適用
git checkout 間違ったブランチ
git reset --hard HEAD~1           # 間違ったブランチから削除
```

### ❌ 大きなファイルをpushしてしまった（履歴ごと削除）

```bash
# git filter-repo を使う（git filter-branch は非推奨）
pip install git-filter-repo
git filter-repo --path 大きなファイル --invert-paths
git push --force-with-lease
```

### ❌ detached HEAD 状態になってしまった

```bash
git log --oneline -5              # 今のコミットハッシュを確認
git checkout -b rescue-branch     # ブランチを作って救出
# または
git switch master                 # ブランチに戻るだけでOKな場合
```

### ❌ マージしたくないブランチをマージしてしまった

```bash
git revert -m 1 MERGE_COMMIT_HASH   # マージコミットを打ち消す（push済みの場合）
git reset --hard ORIG_HEAD           # マージ直後ならこれが最速（push前のみ）
```

### ❌ git pull したら大量のコンフリクトが発生した

```bash
git merge --abort                 # マージをキャンセルして元の状態に戻す
git pull --rebase origin master   # rebaseで取り込んで1件ずつ解消する
```

### ❌ 誰がいつこのバグを入れたか調べたい

```bash
git blame ファイル名               # 各行の最終変更者とコミットを表示
git log -S "バグのある文字列"     # その文字列が追加/削除されたコミットを検索
git bisect start                  # 二分探索でバグを導入したコミットを特定
git bisect bad                    # 今がバグあり
git bisect good v1.0              # v1.0はバグなし
# Gitが自動で中間コミットをチェックアウトしてくるので bad/good を繰り返す
git bisect reset                  # 終了
```

---

## 14. チーム開発のベストプラクティス

### コミットの粒度

```
✅ 良い例：1コミット = 1つの論理的変更
  "feat: ユーザーモデルにemailフィールドを追加"
  "test: UserModel.email のバリデーションテストを追加"

❌ 悪い例：
  "いろいろ修正した"
  "wip"（push直前に squash しよう）
```

### ブランチ命名規則

```
feature/issue-123-user-authentication   # 機能追加
fix/issue-456-null-pointer-exception    # バグ修正
hotfix/critical-security-patch          # 緊急修正
refactor/extract-auth-service           # リファクタリング
docs/update-api-reference               # ドキュメント
chore/upgrade-dependencies              # 雑務・依存更新
```

### レビューを受けやすいPRの作り方

1. **1PR = 1つの目的**（複数の変更を混ぜない）
2. **差分は300行以内**を目安に（大きいと誰もレビューしない）
3. **説明に「なぜ」を書く**（「何を」はコードを見ればわかる）
4. **スクリーンショット**（UI変更があれば必須）
5. **テストを書く**（動作保証になる）

### .gitconfig エイリアス活用例

```ini
[alias]
    st = status
    co = checkout
    br = branch
    lg = log --oneline --graph --all --decorate
    undo = reset --soft HEAD~1
    unstage = restore --staged
    wip = !git add -A && git commit -m "wip"
    unwip = reset HEAD~1
    aliases = config --get-regexp alias
```

---

## 15. コマンド逆引き辞典

| やりたいこと | コマンド |
|---|---|
| 現在の状態を確認 | `git status` |
| 変更の差分を見る | `git diff` |
| ステージの差分を見る | `git diff --staged` |
| 全変更をステージに追加 | `git add .` |
| 変更を選択してステージに追加 | `git add -p` |
| コミットする | `git commit -m "メッセージ"` |
| 直前のコミットを修正 | `git commit --amend` |
| ログを見る | `git log --oneline --graph --all` |
| プッシュする | `git push origin ブランチ名` |
| プルする | `git pull --rebase origin ブランチ名` |
| ブランチを作って移動 | `git checkout -b ブランチ名` |
| ブランチを削除 | `git branch -d ブランチ名` |
| ブランチをリネーム | `git branch -M 新名前` |
| 変更を一時退避 | `git stash` |
| 退避を復元 | `git stash pop` |
| ファイルの変更を取り消し | `git restore ファイル名` |
| ステージを取り消し | `git restore --staged ファイル名` |
| 直前のコミットを取り消し（変更は残す） | `git reset --soft HEAD~1` |
| 直前のコミットを完全に取り消し | `git reset --hard HEAD~1` |
| push済みのコミットを安全に取り消し | `git revert HEAD` |
| 消えたコミットを復元 | `git reflog` |
| 特定コミットだけ取り込む | `git cherry-pick HASH` |
| バグ導入コミットを二分探索 | `git bisect start` |
| 誰がその行を書いたか調べる | `git blame ファイル名` |
| タグを作成 | `git tag v1.0.0` |
| タグをプッシュ | `git push origin --tags` |
| リモートブランチを削除 | `git push origin --delete ブランチ名` |
| 全ての変更を確認してから削除 | `git clean -n` → `git clean -fd` |

---

## 参考リンク

- [Pro Git（無料の公式書籍・日本語あり）](https://git-scm.com/book/ja/v2)
- [GitHub公式ドキュメント](https://docs.github.com/ja)
- [GitHub CLI ドキュメント](https://cli.github.com/manual/)
- [Conventional Commits 仕様](https://www.conventionalcommits.org/ja/)
- [Learn Git Branching（ブランチをビジュアルで学ぶ）](https://learngitbranching.js.org/?locale=ja)
- [gitignore テンプレート生成](https://www.toptal.com/developers/gitignore)

