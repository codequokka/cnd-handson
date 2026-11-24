# MEMO: chapter_docker の実施記録

作業ディレクトリ: `cnd-handson/chapter_docker/`
実施日: 2026-11-24（コマンド出力のタイムスタンプは 2026-11-24 UTC）
環境: Docker 29.10.0

## 結果サマリ

| セクション | 内容 | 結果 |
|---|---|---|
| 1 | サンプルイメージ (`docker/getting-started`) の pull・起動・停止 | 完了（curl で 200 OK） |
| 2 | Dockerfile からオリジナルイメージを作成 | 完了（自作 `index.html` が表示された） |
| 3 | マルチステージビルド（before / after の比較） | 完了（両方 `Hello, World!` を返却） |

## セクション 1: コンテナ作成

- `docker pull docker/getting-started` → `docker run -d -p 8888:80 docker/getting-started`。
- `curl -I localhost:8888` → `HTTP/1.1 200 OK`（nginx/1.23.3）。
- イメージサイズ: DISK USAGE 73.9MB / CONTENT SIZE 21.4MB。
- README は `docker ps` でコンテナ ID を調べて `docker stop <id>` するが、今回は `--name gs` を付けて名前で停止できるようにした。

## セクション 2: Dockerfile からオリジナルコンテナを作成

`hands-on/Dockerfile` は `nginx:latest` に `index.html` を差し替えるだけの構成。

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
RUN service nginx start
```

- タグは README のプレースホルダーに `handson/myapp:v1` を使用。
- ビルド結果: DISK USAGE 241MB / CONTENT SIZE 63.9MB。
- 起動して `curl localhost:8888` → `<title>HelloWorld</title>` と `<h1>Hello World!!🙂</h1>` を確認。
- `RUN service nginx start` はビルド時に一時的に nginx を起動するだけで、実行時の動作には影響しない（実行時は nginx イメージの既定 CMD が使われる）。

## セクション 3: マルチステージビルド

`before/app/server.go` と `after/app/server.go` は同一。違いは Dockerfile のみ。

| | before | after |
|---|---|---|
| ベース | `golang:latest` のみ | ビルド: `golang:latest` / 実行: `gcr.io/distroless/base-debian12` |
| 実行ユーザー | root | `nonroot:nonroot` |
| 含まれるもの | Go ツールチェーン一式 | ビルド済みバイナリのみ |

イメージサイズの比較（`docker images multistage`）:

| イメージ | DISK USAGE | CONTENT SIZE |
|---|---|---|
| `multistage:before` | 1.57GB | 378MB |
| `multistage:after` | 49.5MB | 14.1MB |

- DISK USAGE は約 1.5GB 縮小。README の「1GB ほど縮小」より大きく縮んだ（推測: README 執筆時と `golang:latest` のサイズが変わっている。未検証）。
- 応答確認: `1111`（before）/ `2222`（after）ともに `curl` で `Hello, World!`。
- コンテナは `--rm` 付きで起動しており、`docker stop` で自動削除される。

## つまずきポイント（参考）

- after のビルドで `cd ../after` のパスを誤り失敗した。before のサブシェルを抜けた後は `multistage/` 直下にいるため、`cd after` が正しい。README の通り `cd ..` → `cd after` と順に移動すれば起きない。
- `--rm` 付きコンテナを停止した直後に `docker ps -a` を実行すると、`Dead` と表示されることがある。削除処理の途中で、数秒後に再確認すると消えている。
- ポート 8888 は セクション 1 と 2 で共用しているため、セクション 1 のコンテナを止めてから進む。

## 後始末の状態

- 実行したコンテナはすべて停止・削除済み（残っているのは kind の 3 ノードのみ）。
- ビルドしたイメージ（`handson/myapp:v1`、`multistage:before`、`multistage:after`、`docker/getting-started`）はローカルに残している。

## 次にやること

- 次の chapter へ（例: `chapter_prometheus` / `chapter_argocd`）
