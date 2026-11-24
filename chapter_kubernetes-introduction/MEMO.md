# MEMO: chapter_kubernetes-introduction の実施記録

作業ディレクトリ: `cnd-handson/chapter_kubernetes-introduction/`
実施日: 2026-11-24（コマンド出力のタイムスタンプは 2026-11-24 UTC）
環境: kubectl v1.37.1 / クラスター v1.36.4（kind、3 ノード）

## 結果サマリ

| セクション | 内容 | 結果 |
|---|---|---|
| 1 | kubectl とクラスターの疎通確認 | 完了 |
| 2 | Deployment の適用、ポートフォワード、Pod 削除、スケーリング | 完了 |
| 3 | Service と Gateway API（HTTPRoute）での公開 | 完了（ブラウザ確認 OK） |
| 4 | Rolling Update / Blue-Green Deployment | 完了 |
| 5 | ConfigMap / Secret | 完了 |
| 6 | トラブルシュート（`cnd-web.yaml` の修正） | 完了（ブラウザ確認 OK） |

各セクションの最後にリソースを削除しており、クラスターには `kubernetes` Service のみが残っている。

## セクション 1: 事前準備

- `kubectl version --client` → Client v1.37.1。クラスター（v1.36.4）とは 1 マイナーバージョン差で、許容範囲。
- `kubectl get nodes` → 3 ノードとも Ready。コンテキストは README と同じ `kind-kind`。
- README の `cd ~/cnd-handson/...` は、このリポジトリの配置（ghq 配下）とは異なる。実際のパスに移動して実施した。

## セクション 2: アプリケーションのデプロイ

- 2.1: `test-deployment.yaml` を適用。`test` Deployment と Pod 1 つが作成された。
- 2.2: `kubectl port-forward <Pod名> 8888:80` で `curl -I` が 200 OK。README の `fg` + Ctrl+C は対話シェル向けなので、バックグラウンド実行したプロセスを `kill` で止めた。
- 2.3: Pod を削除すると、名前と AGE が新しい Pod がすぐ作られる（ReplicaSet による復元）。完全に消すには Deployment ごと削除する。
- 2.4: `replicas` を 1 → 2 に変更して再適用すると Pod が 2 つになった。マニフェストの変更は確認後に `git checkout` で戻した。
- 削除した Pod が 30 秒近く `Terminating` のまま残る様子が見られた（nginx の停止待ちと思われるが未検証）。

## セクション 3: Pod の外部公開

- 3.1: `test-service`（ClusterIP）を作成。エンドポイントは `replicas: 2` の分の Pod 2 つ。
- 3.2: `test-httproute` を共通 Gateway `handson-gateway` に紐付け。status は `Accepted=True` / `ResolvedRefs=True`。
- 3.3: `Host: hello-world.example.com` を付けた `curl` で 200 OK。別ホスト名では 404 となり、HTTPRoute のホスト名で振り分けられることを確認した。
- 返るのは nginx の標準ページ（`Welcome to nginx!`）。README の「Hello World の文字が表示されたら成功」とは表示が合わない。使用イメージが `nginx:latest` のためで、README の記述が古い可能性がある。
- VM の `/etc/hosts` には `hello-world.example.com` がない。ブラウザからは、ローカル PC の hosts 設定が必要。

## セクション 4: アプリケーションの更新

### 4.1 Rolling Update

- `rollout.yaml`（`replicas: 4`、`maxSurge: 25%`、`maxUnavailable: 0`、`minReadySeconds: 10`）を適用すると `This App is Blue`。
- `kubectl set image deployment/rollout rollout-app=ghcr.io/cloudnativedaysjp/green-app:1.0` で、新 Pod 1 つ + 旧 Pod 4 つの状態から 1 つずつ入れ替わり、最終的に `This App is Green`。
- `kubectl rollout history` は revision 1 / 2。`kubectl rollout undo` で Blue（`ryuichitakei/blue-app:1.1`）に戻り、履歴は revision 2 / 3 になった。
- `rollout undo` 時に「`kubectl apply` で作ったリソースは `last-applied-configuration` が更新されない」という警告が出る。この章の手順では問題にならない。

### 4.2 Blue-Green Deployment

- `weight` が 100:0 のときは Blue のみ。0:100 に変更して適用すると Green のみ。元のファイルを再適用すると Blue に切り戻せた。
- マニフェストの変更は確認後に `git checkout` で戻した。

### 注意点

- イメージの取得元が混在している。`rollout.yaml` と `blue-green.yaml` は `ryuichitakei/*`（個人の Docker Hub）、README の `set image` は `ghcr.io/cloudnativedaysjp/*`。いずれも pull はできた。

## セクション 5: ConfigMap / Secret

- 5.1: ConfigMap を `subPath` で nginx の `index.html` にマウント。一時 Pod（`nginx:alpine`）から `curl <PodのIP>` すると `CND ConfigMap Handson` の HTML が返った。
- 5.2: Secret を環境変数として渡し、Pod 内で `USERNAME=user01` / `PASSWORD=passwd` を確認。Secret の保存形式は base64 で、暗号化ではない。
- README は `kubectl exec -it ... -- bash` で対話ログインする。今回は TTY がないため `sh -c 'echo ...'` で同じ内容を非対話で確認した。

## セクション 6: トラブルシュート

`cnd-web.yaml` を適用すると、HTTPRoute の先に「dummy App」が表示され、問題は 2 層に分かれていた。リポジトリのファイルは課題用の壊れた状態のまま残し、修正は別ファイルにコピーして行った。

| # | 箇所 | 元 | 修正後 | 症状 |
|---|---|---|---|---|
| 1 | `cnd-web-svc` の `selector` | `name: dummy-app` | `name: cnd-web-app` | 「アプリケーションの接続先が異なります」 |
| 2 | `cnd-web-svc` の `targetPort` | `80` | `8888` | アプリが待ち受ける 8888 に届かない |
| 3 | `mysql-svc` の `targetPort` | `8888` | `3306` | `Connection refused`（エンドポイントも `<unset>`） |
| 4 | `mysql` Pod の `MYSQL_ROOT_PASSWORD` | `password` | `cndw2024` | `1045 Access denied`（Secret の `DB_Password` は `cndw2024`） |

進め方と判断:

- 1・2 を直すとアプリ本体のページが出る。3 を直すと DB 接続エラーが `Connection refused` から認証失敗に変わり、4 が見えた。
- 4 は MySQL 側を Secret に合わせた。Secret を `password` に変える直し方でも成立する。Pod の `env` は後から変更できないため、`mysql` Pod は削除して作り直した。
- 修正後に「Databaseへの接続に成功しました！」が表示された。
- `mysql` イメージの初回 pull に約 1 分かかり、その間 `ContainerCreating` が続く。故障ではない。
- `dummy-app` は修正後はどこからも参照されなくなる。
- 切り分けのために、アプリ本体（`cnd-web-app`）のログと、Pod IP への直接アクセスを使った。

## 後始末の状態

- 実行したリソースはすべて削除済み。
- ローカルの作業ツリーに変更はない（実施中に書き換えたマニフェストはすべて元に戻した）。

## 次にやること

- 次の chapter へ（例: `chapter_kubernetes-overview`、`chapter_troubleshoot`）
