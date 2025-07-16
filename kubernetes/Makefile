# Variables
CLUSTER_NAME ?= k8s-local

# Colors
GREEN = \033[0;32m
BLUE = \033[0;34m
YELLOW = \033[0;33m
RED = \033[0;31m
NC = \033[0m

.PHONY: help
help: ## Show help
	@echo "$(BLUE)🚀 Kubernetes Platform Setup$(NC)"
	@echo ""
	@echo "$(BLUE)Phase 1: Cilium Installation$(NC)"
	@echo "make phase1        - Complete Phase 1"
	@echo "make cluster-create - Create k3d cluster"
	@echo "make cilium-install - Install Cilium"
	@echo "make fix-coredns   - Fix CoreDNS configuration"
	@echo "make status        - Check cluster status"
	@echo ""
	@echo "$(BLUE)Phase 2: FluxCD Installation$(NC)"
	@echo "make phase2        - Complete Phase 2"
	@echo "make flux-install  - Install FluxCD"
	@echo "make flux-status   - Check FluxCD status"
	@echo ""
	@echo "$(BLUE)Phase 3: Gitea Git Repository$(NC)"
	@echo "make phase3        - Complete Phase 3"
	@echo "- Deploys Gitea in gitea namespace"
	@echo "- Admin: giteaadmin/admin123"
	@echo "- Access: kubectl port-forward -n gitea svc/gitea-gitea-http 3000:3000"
	@echo ""
	@echo "$(BLUE)Phase 4: Application Deployment$(NC)"
	@echo "make phase4        - Complete Phase 4"
	@echo ""
	@echo "$(BLUE)Individual targets:$(NC)"
	@echo "make gitops-setup    - Setup FluxCD GitOps management"
	@echo "make gitops-status   - Check GitOps status"

.PHONY: cluster-create
cluster-create: ## Create k3d cluster
	@echo "$(BLUE)🚀 Creating k3d cluster...$(NC)"
	@if k3d cluster list 2>/dev/null | grep -q "^$(CLUSTER_NAME)"; then \
		echo "$(YELLOW)⚠️  Cluster already exists$(NC)"; \
	else \
		export K3D_FIX_DNS=1 && \
		export K3D_FIX_MOUNTS=1 && \
		k3d cluster create $(CLUSTER_NAME) \
			--port "80:80@loadbalancer" \
			--port "443:443@loadbalancer" \
			--k3s-arg "--disable=traefik,metrics-server@server:*" \
			--k3s-arg "--disable-network-policy@server:*" \
			--k3s-arg "--flannel-backend=none@server:*" \
			--k3s-arg "--disable-kube-proxy@server:*" \
			--k3s-arg "--resolv-conf=/etc/resolv.conf@server:*"; \
		echo "$(GREEN)✅ Cluster created$(NC)"; \
	fi

.PHONY: cilium-install
cilium-install: ## Install Cilium
	@echo "$(BLUE)📦 Installing Cilium...$(NC)"
	@helm repo add cilium https://helm.cilium.io/ >/dev/null 2>&1 || true
	@helm repo update >/dev/null 2>&1
	@if helm list -n kube-system | grep -q cilium; then \
		echo "$(YELLOW)⚠️  Cilium already installed$(NC)"; \
	else \
		helm install cilium cilium/cilium --version 1.17.6 \
			--namespace kube-system \
			--set kubeProxyReplacement=true \
			--set k8sServiceHost=k3d-$(CLUSTER_NAME)-server-0 \
			--set k8sServicePort=6443 \
			--set ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" \
			--set operator.replicas=1 \
			--set gatewayAPI.enabled=true \
			--set hostServices.enabled=false \
			--set dnsProxy.enabled=false \
			--set ipv6.enabled=false \
			--set enableIPv4Masquerade=true \
			--wait --timeout=300s; \
		echo "$(GREEN)✅ Cilium installed$(NC)"; \
	fi

.PHONY: fix-coredns
fix-coredns: ## Fix CoreDNS after k3s stabilization
	@echo "$(BLUE)⏳ Waiting for k3s initialization to complete...$(NC)"
	@kubectl wait --for=condition=ready node --all --timeout=300s >/dev/null
	@sleep 60
	@echo "$(BLUE)🔧 Applying CoreDNS patch (after k3s stabilization)...$(NC)"
	@kubectl apply -f infrastructures/kubernetes/overlays/k3d/coredns-patch.yaml >/dev/null
	@kubectl rollout restart deployment/coredns -n kube-system >/dev/null
	@kubectl rollout status deployment/coredns -n kube-system --timeout=300s >/dev/null
	@echo "$(BLUE)🧪 Testing DNS resolution...$(NC)"
	@if kubectl run dns-test --image=busybox --restart=Never --command -- sh -c "nslookup google.com" >/dev/null 2>&1; then \
		kubectl wait --for=condition=ready pod dns-test --timeout=30s >/dev/null 2>&1; \
		echo "$(GREEN)✅ DNS resolution working$(NC)"; \
		kubectl delete pod dns-test >/dev/null 2>&1 || true; \
	else \
		echo "$(RED)❌ DNS resolution failed$(NC)"; \
		kubectl delete pod dns-test >/dev/null 2>&1 || true; \
		exit 1; \
	fi

.PHONY: status
status: ## Check cluster status
	@echo "$(BLUE)📊 Cluster Status$(NC)"
	@kubectl get nodes
	@echo ""
	@kubectl get pods -n kube-system

.PHONY: phase1
phase1: cluster-create cilium-install fix-coredns status ## Complete Phase 1
	@echo "$(GREEN)🎉 Phase 1 Complete!$(NC)"

# Phase 2: FluxCD インストール
.PHONY: flux-install
flux-install: ## Install FluxCD
	@echo "$(BLUE)📦 Installing FluxCD...$(NC)"
	@if kubectl get namespace flux-system >/dev/null 2>&1; then \
		echo "$(YELLOW)⚠️  Already installed$(NC)"; \
	else \
		flux install --namespace=flux-system >/dev/null; \
		echo "$(GREEN)✅ FluxCD installed$(NC)"; \
	fi
	@echo "$(BLUE)⏳ Waiting for FluxCD to be ready...$(NC)"
	@kubectl wait --for=condition=ready pod -l app=source-controller -n flux-system --timeout=300s >/dev/null
	@echo "$(GREEN)✅ FluxCD is ready$(NC)"

.PHONY: flux-status
flux-status: ## Check FluxCD status
	@echo "$(BLUE)📊 FluxCD Status$(NC)"
	@kubectl get pods -n flux-system
	@echo ""
	@flux get all -A 2>/dev/null || echo "$(YELLOW)No resources found$(NC)"

.PHONY: phase2
phase2: flux-install flux-status ## Complete Phase 2
	@echo "$(GREEN)🎉 Phase 2 Complete!$(NC)"

# Phase 3: Gitea Deployment
# (Gitea will be deployed via FluxCD in infrastructures/gitea/)
.PHONY: phase3
phase3: ## Complete Phase 3 (Gitea Deployment)
	@echo "$(BLUE)💻 Deploying Gitea...$(NC)"
	@kubectl apply -f infrastructures/gitea/overlays/k3d >/dev/null
	@echo "$(BLUE)⏳ Waiting for Gitea to be ready...$(NC)"
	@kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=gitea -n gitea --timeout=300s >/dev/null
	@echo "$(GREEN)✅ Gitea is ready$(NC)"
	@echo "$(BLUE)Access: kubectl port-forward -n gitea svc/gitea-gitea-http 3000:3000$(NC)"
	@echo "$(BLUE)Admin: giteaadmin/admin123$(NC)"
	@echo "$(GREEN)🎉 Phase 3 Complete!$(NC)"

# Phase 4: Application Deployment
.PHONY: gitops-setup
gitops-setup: ## Setup FluxCD GitOps management
	@echo "$(BLUE)🔧 Setting up FluxCD GitOps...$(NC)"
	@echo "$(BLUE)Creating FluxCD resources...$(NC)"
	@kubectl apply -f infrastructures/helmrepositories.yaml >/dev/null
	@export GIT_REPO_URL="http://gitea-gitea-http.gitea.svc.cluster.local:3000/giteaadmin/platform.git"; \
	echo "$(BLUE)Git URL: $$GIT_REPO_URL$(NC)"; \
	envsubst < templates/gitrepository.yaml | kubectl apply -f - || echo "$(YELLOW)⚠️  Push code to repository first$(NC)"; \
	kubectl apply -f templates/kustomization.yaml || echo "$(YELLOW)⚠️  GitRepository must be ready first$(NC)"; \
	echo "$(GREEN)✅ GitOps setup completed$(NC)"

.PHONY: gitops-status
gitops-status: ## Check GitOps status
	@echo "$(BLUE)📊 GitOps Status$(NC)"
	@flux get sources git -A || echo "$(YELLOW)No Git sources found$(NC)"
	@flux get kustomizations -A || echo "$(YELLOW)No Kustomizations found$(NC)"

.PHONY: phase4
phase4: gitops-setup gitops-status ## Complete Phase 4 (Application Deployment)
	@echo "$(GREEN)🎉 Phase 4 Complete!$(NC)"
	@echo "$(BLUE)Application deployment ready$(NC)"
