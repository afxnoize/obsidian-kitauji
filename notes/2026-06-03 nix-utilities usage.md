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

整理するなら GC を nix-sweep か nh clean のどちらかに寄せるのが唯一の重複ポイント。今は両刀のまま。