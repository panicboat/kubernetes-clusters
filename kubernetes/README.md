# Kubernetes Platform with GitOps

## 概要

Phase-based approach でKubernetesプラットフォームを構築する自動化スクリプト集です。Cilium + FluxCD + OpenTelemetry + Prometheus のスタックを段階的にセットアップします。

## 🏗️ アーキテクチャ

```
Phase 1: Cilium (CNI + kube-proxy replacement)
    ↓
Phase 2: FluxCD (GitOps Controller)
    ↓
Phase 3: Gitea (Git Repository Server)
    ↓
Phase 4: Application Deployment (Monitoring Stack etc.)
```

## ✅ 現在の実装状況

### 完了済み

- ✅ **Phase 1**: Cilium CNI + DNS修正 + kube-proxy置換
- ✅ **Phase 2**: FluxCD GitOps基盤
- ✅ **Phase 3**: Gitea Gitリポジトリサーバー (Kubernetes管理)
- ⚠️ **Phase 4**: アプリケーション展開基盤

### 動作中のコンポーネント

| コンポーネント | 状態 | 管理方法 | 備考 |
|-------------|------|--------|------|
| Cilium | ✅ Running | 自動インストール | DNS修正含む |
| FluxCD | ✅ Running | 自動インストール | GitOps基盤 |
| Gitea | ✅ Running | Kubernetes管理 | gitea namespace |
| Prometheus Stack | ⚠️ Pending | FluxCD管理 | CRD依存性問題 |
| OpenTelemetry | ⚠️ Pending | FluxCD管理 | CRD依存性問題 |

## 🚀 クイックスタート

### 前提条件

- Docker
- kubectl
- k3d
- flux CLI
- helm CLI

### Phase 1: Cilium基盤構築

```bash
make phase1
```

**実行内容:**
- k3d クラスター作成
- Cilium CNI インストール (kube-proxy置換)
- CoreDNS 修正
- ネットワーク接続性確認

### Phase 2: FluxCD導入

```bash
make phase2
```

**実行内容:**
- FluxCD インストール
- GitOps基盤構築

### Phase 3: Giteaセットアップ

```bash
make phase3
```

**実行内容:**
- Giteaサーバーをgitea namespaceにデプロイ
- 管理者アカウント自動作成 (giteaadmin/admin123)
- ClusterIPサービスで内部アクセス提供

**Giteaアクセス方法:**

```bash
# 1. port-forwardでアクセスを有効化
kubectl port-forward -n gitea svc/gitea-gitea-http 3000:3000 &

# 2. ブラウザでGiteaアクセス
# http://localhost:3000 にアクセス
# ユーザー: giteaadmin / パスワード: admin123 でログイン

# 3. リポジトリ作成
# "platform" という名前でリポジトリを作成

# 4. ローカルGitリポジトリ初期化（新規プロジェクトの場合）
git init
git add .
git commit -m "Initial commit"
git branch -M main

# 5. リモート追加とプッシュ（認証情報付き）
git remote add gitea http://giteaadmin:admin123@localhost:3000/giteaadmin/platform.git
git push -u gitea main
```

### Phase 4: アプリケーション展開

```bash
make phase4
```

**実行内容:**
- GitOpsリポジトリ接続
- HelmRepositories適用
- 監視スタック展開 (Prometheus + OpenTelemetry)

## 📁 プロジェクト構造

```
├── Makefile                     # 自動化スクリプト
├── templates/                   # GitOps テンプレート
│   ├── gitrepository.yaml       # FluxCD Git接続
│   └── kustomization.yaml       # インフラ管理
└── infrastructures/             # Kubernetes マニフェスト
    ├── helmrepositories.yaml    # Helm リポジトリ定義
    ├── gitea/                   # Gitea設定
    ├── prometheus-operator/     # Prometheus設定
    └── opentelemetry/          # OpenTelemetry設定
```

## 🛠️ Makefile targets

### Phase実行

```bash
make phase1                    # Cilium基盤構築
make phase2                    # FluxCD導入
make phase3                    # Giteaデプロイ
make phase4                    # アプリケーション展開
```

### 個別操作

```bash
# GitOps管理
make gitops-setup             # FluxCD設定作成
make gitops-status            # GitOps状態確認

# クラスター確認
make status                   # クラスター状態
make flux-status              # FluxCD状態
```

## 💡 Tips

### Git操作

```bash
# リモート確認
git remote -v

# 複数リモートへのpush
git add . && git commit -s -m "your commit message"
git push gitea main    # ローカルGiteaへ（検証用）
git push origin main   # GitHubへ（本番用）

# 全リモートに一括push
git push --all
```

### デバッグ

```bash
# Gitea Pod状態確認
kubectl get pods -n gitea
kubectl logs -n gitea -l app.kubernetes.io/name=gitea
kubectl exec -n gitea -it deployment/gitea-gitea -- sh

# FluxCD同期状況
flux get all -A
flux reconcile source git platform-source

# Cilium状態確認
cilium status
cilium connectivity test
```

### クリーンアップ

```bash
# クラスター完全削除
k3d cluster delete k8s-local

# 全体リセット（Giteaデータも含む）
k3d cluster delete k8s-local
```

### 開発ワークフロー

```bash
# 1. 実験用ブランチ作成
git checkout -b experiment-feature

# 2. ローカルGiteaで検証
git push gitea experiment-feature
make phase4  # 検証環境で確認

# 3. 問題なければmainにマージしてGitHubへ
git checkout main
git merge experiment-feature
git push origin main
```

## ⚠️ 現在の課題・制限事項

### 既知の問題

1. **Prometheus Operator CRD依存性**
   - `templates/kustomization.yaml`のhealthChecksでkube-prometheus-stackを参照
   - 実際のHelmReleaseがprometheus-operator-systemネームスペースに存在しない可能性
   - GitOps展開時にCRDインストール順序の問題が発生する場合がある

2. **Kustomizationパス設定**
   - `templates/kustomization.yaml`のpath設定が `"./infrastructures"` に戻された
   - GitOpsリポジトリ構造との整合性要確認

3. **GitOps展開の安定性**
   - FluxCDによる監視スタック展開で依存関係エラーが発生する可能性
   - CRDとHelmReleaseのインストール順序調整が必要

### 対応予定

- [ ] Prometheus Operator CRDの事前インストール自動化
- [ ] healthChecks設定の見直し
- [ ] GitOpsリポジトリパス設定の最適化
- [ ] 監視スタック展開の完全自動化

## 🎯 今後の改善案

### 🔮 Phase 5以降の計画

#### 完全GitOps化
```bash
# 長期目標
- Cilium も FluxCD管理に移行
- クラスター全体のGitOps管理
- マルチクラスター対応
```

#### 本番環境対応
```bash
# エンタープライズ機能
- RBAC/NetworkPolicy
- TLS証明書管理
- 永続化Storage設定
- Backup/Restore
```

## 📊 監視・オブザーバビリティ

### 現在利用可能なツール

- **Prometheus**: メトリクス収集・アラート
- **Grafana**: ダッシュボード (prometheus-operator-system namespace)
- **OpenTelemetry**: トレース・メトリクス統合
- **Cilium Hubble**: ネットワーク観測

### アクセス方法

```bash
# Grafana (port-forward)
kubectl port-forward -n prometheus-operator-system svc/prometheus-operator-system-kube-prometheus-stack-grafana 3000:80

# Prometheus (port-forward)
kubectl port-forward -n prometheus-operator-system svc/prometheus-prometheus-operator-system-prometheus 9090:9090
```

## 🔧 トラブルシューティング

### よくある問題

1. **CoreDNS解決失敗**
   ```bash
   kubectl apply -f infrastructures/kubernetes/overlays/k3d/coredns-patch.yaml
   kubectl rollout restart deployment/coredns -n kube-system
   ```

2. **FluxCD同期失敗**
   ```bash
   flux reconcile source git platform-source
   flux get all -A
   ```

3. **Gitea接続確認**
   ```bash
   kubectl port-forward -n gitea svc/gitea-gitea-http 3000:3000 &
   curl http://localhost:3000/api/v1/version
   kubectl logs -n gitea -l app.kubernetes.io/name=gitea
   ```

### ログ確認

```bash
# FluxCD
flux logs

# Cilium
cilium status
kubectl logs -n kube-system -l k8s-app=cilium

# OpenTelemetry
kubectl logs -n opentelemetry-system -l app.kubernetes.io/name=opentelemetry-collector
```

## 🤝 開発・運用

### ローカル開発

1. Phase 1-3でプラットフォーム構築
2. `kubectl port-forward`でサービスアクセス
3. Gitea使用でGitOpsワークフロー体験

### 本番運用への移行

1. **外部Git**: GitHubなど外部GitプロバイダーへFluxCD接続
2. **Ingress/Gateway**: 実際のドメインでサービス公開
3. **永続化**: 適切なStorageClass使用
4. **セキュリティ**: TLS、RBAC、NetworkPolicy設定

## 📚 参考

- [Cilium Documentation](https://docs.cilium.io/)
- [FluxCD Documentation](https://fluxcd.io/docs/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [OpenTelemetry](https://opentelemetry.io/)

---

**Status**: 全Phase完了、完全自動化済み
**Next**: 本番環境対応、完全GitOps化
