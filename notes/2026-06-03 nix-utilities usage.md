## Nix 周りパッケージ usage まとめ

現在 `home.nix` / modules に入ってる Nix 系ツール。

### nh（今回追加）
home-manager の操作ラッパー。
```bash
nh home switch ./nix -c $USER -- --impure   # ビルド+差分プレビュー+適用
nh search ripgrep                            # 高速パッケージ検索
nh clean all                                 # 世代/時間ベース GC（要確認: nix-sweep と用途重複）
```
`NH_FLAKE` を設定すればパス指定を省ける。`--impure` 必須なのは既存事情のとおり。

### nix-output-monitor（nom）
`nix build` の出力をツリーで可視化。nh が内部利用するので単体で叩く機会は減る。
```bash
nom build --impure .#homeConfigurations.$USER.activationPackage
nix build ... |& nom    # パイプでも使える
```

### nix-sweep（unstable）
世代の掃除 / GC。nh clean と役割は被るが残置中。
```bash
nix-sweep            # 古い世代をプレビュー
nix-sweep --keep 5   # 直近5世代だけ残す
```
ローカルの実バイナリ（unstable 版）から正確な仕様を取った。README より新しくサブコマンド構成になってる。

世代の掃除と gc-root 整理に特化したツール。`nh clean` と GC 用途は被るが、こっちは**プロファイル世代の細かい保持ポリシー**と**store 分析**が強み。

#### tldr
対応リスト（使ってる store / 消したいもの / コマンド）：

- **home-manager 世代** → nix-sweep の `home`
  - 実体: `~/.local/state/nix/profiles/home-manager`（6.34 GiB / 10 gens）
  - dotfiles 由来: `hms`（`nh home switch`）を打つたびに増える
  - 消す: `nix-sweep cleanout home --keep-min 5 --remove-older 14 --gc`
- **命令的 nix profile** → nix-sweep の `user`
  - 実体: `~/.local/state/nix/profiles/profile`（15.0 GiB / 33 gens）← 一番でかい
  - dotfiles 由来: home-manager じゃなく `nix profile install` 系で入れたもの。Determinate Nix インストーラや手動 install の蓄積。33世代はそれの履歴
  - 消す: `nix-sweep cleanout user --keep-min 5 --remove-older 14 --gc`
  - （home とまとめて `cleanout user home ...` で一発でいい）
- **direnv の dev shell** → profile じゃなく **gc-root**
  - 実体: 各リポの `.direnv/flake-profile-*`（合計 8.12 GiB の大半）
  - dotfiles 由来: `xeno-nz/*` のリポで `nix develop` / direnv 使ってるぶん
  - 消す: `nix-sweep tidyup-gc-roots --older 30d`（cleanout では消えない）
  - 消しても次に direnv allow / nix develop で作り直されるだけ
- **NixOS system profile** → 君の環境には無い
  - 理由: Manjaro + Determinate Nix なので NixOS の `/nix/var/nix/profiles/system` は存在しない。analyze に出た `per-user/root/profile`（356 MiB）は root 用で、いじるなら sudo 案件。基本放置
- **store 全体の重複** → profile/root とは別軸
  - 実体: `/nix/store` 20.1 GiB（未最適化）
  - 消す: `nix store optimise`（削除じゃなく dedup でハードリンク化）

順番でいうと: `cleanout user home --gc` → `tidyup-gc-roots --older 30d` → worktree の result を rm → `nix store optimise`。profile 側（21 GiB）と root 側（8 GiB）の両方に効くのはこの並び。

#### サブコマンド一覧

| コマンド                 | 役割                                           |
| ------------------------ | ---------------------------------------------- |
| `cleanout`               | 古い世代の削除（メイン機能）                   |
| `generations`            | 世代の一覧表示                                 |
| `analyze`                | store サイズ・最適化状態・closure 占有率の分析 |
| `gc`                     | `nix-store --gc` のラッパー                    |
| `gc-roots`               | gc-root の一覧                                 |
| `tidyup-gc-roots`        | 不要 gc-root の選択削除                        |
| `presets`                | cleanout 用プリセットの確認                    |
| `add-root` / `path-info` | root 追加 / パス情報                           |

`<PROFILES>` は `system` / `user` / `home` / `<プロファイルパス>` を指定。あなたは home-manager なので主に **`home`**。

#### cleanout（中核）

保持基準は**正基準（keep 系）が負基準（remove 系）より優先**。最新世代と現役世代は基準に合致しても消えない。

```bash
# まず dry-run で確認（必ずこれから）
nix-sweep cleanout home --dry-run

# 最低5世代は残しつつ、14日より古いものを削除
nix-sweep cleanout home --keep-min 5 --remove-older 14

# 確認なしで実行 + 後続 GC
nix-sweep cleanout home --keep-min 3 --remove-older 30 --non-interactive --gc
```

主なフラグ：

| フラグ                           | 意味                                         |
| -------------------------------- | -------------------------------------------- |
| `--keep-min N`                   | 最低 N 世代は残す                            |
| `--keep-max N`                   | 最大 N 世代まで                              |
| `--keep-newer D`                 | D 日より新しいものは全て残す                 |
| `--remove-older D`               | D 日より古いものを削除                       |
| `-g, --generation N`             | 特定世代を指名削除（複数可）                 |
| `-d, --dry-run`                  | 削除せず一覧のみ                             |
| `-i / -n`                        | 対話確認する / しない                        |
| `--gc`                           | 削除後に GC 実行                             |
| `--gc-bigger G` / `--gc-quota %` | store が G GiB / device の %超 のときだけ GC |
| `--gc-modest`                    | 上記閾値に達する分だけ控えめに回収           |
| `--no-size`                      | サイズ計算を省略（旧機種で高速化）           |
| `-p, --preset` / `-C, --config`  | プリセット / 別 config                       |

#### analyze（地味に便利）

```bash
nix-sweep analyze              # store サイズ + profile/root の closure 占有率 top5
nix-sweep analyze -a -d        # 全 root 表示 + dead path 情報（重い）
```
どの世代/root が store を食ってるかを把握してから cleanout、が定石。

#### gc-root の掃除

```bash
nix-sweep gc-roots --older 30d           # 30日より古い root を一覧
nix-sweep tidyup-gc-roots --older 30d     # それらを対話削除（-f で即時）
```
direnv や nix-shell が残す古い root が溜まりがちなので、ここが nh clean では拾いにくい差分。

#### プリセット（TOML）

`cleanout` の基準を名前付きで保存。場所は `$XDG_CONFIG_HOME/nix-sweep/presets.toml` または `/etc/nix-sweep/presets.toml`。

```bash
nix-sweep presets --list                  # 利用可能プリセット一覧
nix-sweep cleanout home -p housekeeping    # プリセット適用
```

#### 環境変数
`NIX_SWEEP_NUM_THREADS` でワーカースレッド数を調整可能。

#### nh clean との棲み分け（再掲・具体化）

| やりたいこと                                   | 向いてる方                    |
| ---------------------------------------------- | ----------------------------- |
| home 世代を keep-min/remove-older で細かく整理 | **nix-sweep cleanout**        |
| store の占有内訳を見る                         | **nix-sweep analyze**         |
| 古い direnv/shell の gc-root 掃除              | **nix-sweep tidyup-gc-roots** |
| 全プロファイル横断でざっくり GC                | nh clean all                  |

### nix-index + comma（`programs.nix-index`, `home.nix:59-60`）
コマンド名 → パッケージ逆引き。DB は `nix-index-database` で事前ビルド済み。
```bash
nix-locate bin/fd      # ファイルがどの pkg に属するか
, ripgrep              # 未インストールの pkg を一時実行（comma）
```

### 比較・棲み分け

| 用途           | 第一選択          | 重複/代替                            |
| -------------- | ----------------- | ------------------------------------ |
| 設定の適用     | `nh home switch`  | 素の `nix build --impure` + activate |
| ビルド可視化   | nom（nh が内包）  | —                                    |
| GC/世代整理    | nix-sweep         | `nh clean`（被り）                   |
| パッケージ検索 | `nh search`       | `nix search`                         |
| コマンド逆引き | nix-index / comma | —                                    |
