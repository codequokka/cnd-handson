# MEMO: chapter_setup / chapter_cluster-create の実施記録

作業ディレクトリ: `cnd-handson/chapter_cluster-create/`
実施日: 2026-11-24（コマンド出力のタイムスタンプは 2026-11-23 UTC）

## 結果サマリ

| 手順 | 内容 | 結果 |
|---|---|---|
| 5 | kind でクラスター作成 | 完了（3 ノードとも Ready） |
| 6-1 | Gateway API CRD (v1.6.1, experimental) | 完了 |
| 6-2 | Envoy Gateway 独自 CRD (v1.9.1) | 完了 |
| 6-3 | Cilium 1.20.4 / Envoy Gateway v1.9.1 を helmfile で導入 | 完了 |
| 7 | 共通 Gateway `handson-gateway` 作成 | 完了（PROGRAMMED=True） |
| 8 | デモアプリ (`handson` ns) デプロイ | 完了（curl で 200 OK） |

未実施・未確認:
- ローカル PC のブラウザから `http://app.example.com` を開く確認
- kubectl のシェル補完設定（任意）
- `cilium connectivity test`（任意、約 10 分）

## 全体の流れ

```
[chapter_setup]  VM 作成 → VM セットアップ(Docker 等) → hosts 設定 / リポジトリ clone
[chapter_cluster-create]
  ツール導入 → kind クラスター作成 → CRD 導入 → Cilium/Envoy Gateway 導入
  → Gateway 作成 → デモアプリ確認
```

通信経路:

```
ブラウザ/curl ─▶ VM:80 ─(kind ポート転送)─▶ Control Plane:30080 (NodePort)
  ─▶ Envoy (Envoy Gateway) ─▶ HTTPRoute ─▶ Service handson ─▶ Pod
```

## kind-config.yaml のポイント

- ノード: Control Plane 1 台 + Worker 2 台（1 ノード = 1 Docker コンテナ）。
- イメージ: `kindest/node:v1.36.4@sha256:...`（ダイジェストまで固定）。Cilium 1.20 と Envoy Gateway v1.9 の対応範囲（K8s 1.33〜1.36）に合わせている。
- `networking.disableDefaultCNI: true` / `kubeProxyMode: none`: kindnet と kube-proxy を使わず、Cilium が CNI と kube-proxy 代替を担う。このためクラスター作成直後は NotReady で正常。
- `extraPortMappings`（VM → Control Plane コンテナ）。クラスター作成時にしか設定できない。

  | VM | ノード | 用途 | NodePort の固定箇所 |
  |---|---|---|---|
  | 80 | 30080 | Envoy Gateway | `manifest/gateway/gateway.yaml` |
  | 8080 / 8443 | 31080 / 31443 | Cilium Ingress | `helm/values/cilium.values.yaml` |
  | 18080 / 18443 | 32080 / 32443 | Istio | chapter_istio 側 |

- `node-labels: ingress-ready=true` は Ingress NGINX 時代の名残り。リポジトリ内に参照箇所はなく、現構成では不要。

## 手順 6 の要点

- **6-1**: `kubectl apply --server-side -f .../v1.6.1/experimental-install.yaml`
  - Gateway API は K8s 本体に含まれないため CRD を追加。`--server-side` は CRD が大きく通常の apply だとアノテーション上限を超えるため。
  - experimental 版は standard 版を包含する。standard を先に入れると `ValidatingAdmissionPolicy`（`safe-upgrades.gateway.networking.k8s.io`）に拒否されるので、experimental のみ適用する。
  - experimental を選んだ理由は README に明記なし。experimental 限定のリソースも現状のマニフェストでは未使用（推測: 後の章や将来のための余地）。
- **6-2**: `helm template ... | kubectl apply --server-side -f -`
  - CRD が大きく `helm install` だとリリース Secret の上限 1MiB を超えるため。
  - values で Gateway API CRD を無効化（6-1 で入れ済みのため、バージョン/チャンネル不一致や管理元の二重化を避ける）。
- **6-3**: `helmfile sync -f helm/helmfile.yaml`
  - `needs` で Cilium → Envoy Gateway の順。Envoy Gateway は証明書生成 Job で Pod を起動するため、CNI が先に必要。
  - Cilium は Hubble、Cilium Ingress、Gateway API サポートも有効（後の章で使用）。

## 手順 7: gateway.yaml の構成

- `Namespace gateway`: Gateway の置き場。
- `EnvoyProxy handson-proxy`: Envoy の Service を `NodePort`（kind に LB がないため）、`nodePort: 30080` 固定、`externalTrafficPolicy: Cluster`（デフォルトの `Local` だと Envoy Pod がないノードで通信が破棄される。ポート転送先は Control Plane のみ）。
- `GatewayClass eg`: `controllerName: gateway.envoyproxy.io/gatewayclass-controller`、`parametersRef` で EnvoyProxy を紐付け。
- `Gateway handson-gateway`: HTTP 80、`allowedRoutes.namespaces.from: All`（全 chapter から HTTPRoute を紐付け可能。本番では `Same`/`Selector` で制限する）。

## 実行結果（確認できたこと）

- ノード: `kind-control-plane` / `kind-worker` / `kind-worker2` が Ready（v1.36.4）。
- `cilium status`: Cilium / Operator / Envoy DaemonSet / Hubble Relay すべて OK、Helm chart 1.20.4、Cluster Pods 6/6 managed。
- `envoy-gateway-system`: `envoy-gateway-...` が 1/1 Running。
- GatewayClass: `eg` と `cilium` が ACCEPTED=True。
- Gateway: `handson-gateway`（class `eg`）が PROGRAMMED=True、ADDRESS `172.18.0.2`（環境依存）。
- Service: `envoy-gateway-handson-gateway-e8a6ab28` が NodePort `80:30080/TCP`。
- デモアプリ: `service/handson`（ClusterIP 8080）、`deployment/handson-blue`（1/1）、`httproute/handson`（`app.example.com`）。
- HTTPRoute の status: `Accepted` / `ResolvedRefs` ともに True。
- `curl -I http://app.example.com` → `HTTP/1.1 200 OK`。

## つまずきポイント（参考）

- docker グループ追加後は再ログインが必要。
- inotify 上限不足で Pod が `too many open files` になる（`sysctl` で引き上げ）。
- `extraPortMappings` を変えるにはクラスターの作り直しが必要。
- 画面が出ないときは、ローカル PC の hosts、Gateway の PROGRAMMED、Security Group の 80 番を確認する。

## 次にやること

- ローカル PC のブラウザで `http://app.example.com` を確認
- 次の chapter へ（例: `chapter_docker`、または `chapter_prometheus` / `chapter_argocd`）
