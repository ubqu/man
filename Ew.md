East-West (Multi-Cluster)
	•	Service-to-service traffic across clusters → istio-eastwestgateway (sm-gateways):
	•	Purpose:
	•	Provides secure east-west connectivity between meshes in different clusters.
	•	Terminates/initiates mTLS for cross-cluster service traffic and routes it based on SNI / service discovery.
	•	Acts as the controlled entry/exit point for in-mesh traffic that must traverse DC/cluster boundaries.
	•	Ports (example / as deployed):
	•	15021/TCP — Envoy health checks (/healthz/ready) used by the external L4 LB.
	•	15443/TCP — Cross-cluster mTLS / SNI-based routing for east-west traffic (primary data plane port).
	•	15012/TCP — Secure xDS (control-plane → gateway) to receive config/certs from istiod.
	•	15017/TCP — Webhook / control-plane related port (used for specific Istio control-plane interactions depending on installation profile).
	•	Typical network path:
	1.	Workload sidecar in Cluster A → routes to istio-eastwestgateway Service in sm-gateways (when destination is remote).
	2.	istio-eastwestgateway (Cluster A) → external L4 routing (DC interconnect / VPN / WAN) → istio-eastwestgateway (Cluster B).
	3.	istio-eastwestgateway (Cluster B) → destination workload sidecar in Cluster B (mTLS inside the mesh).
	•	Encryption & identity:
	•	mTLS end-to-end for cross-cluster traffic (mesh trust model: shared root CA or federated trust domains).
	•	Workload identity remains SPIFFE-based; gateway enforces the same mesh security posture for remote peers.
	•	Policy / controls (recommended):
	•	NetworkPolicies limit who can reach the east-west gateway (only mesh workloads / trusted CIDRs from peer clusters).
	•	AuthorizationPolicies restrict which identities can use cross-cluster routes (especially for regulated/data-sensitive services).
	•	Notes:
	•	External connectivity must be explicitly designed (routing, firewall rules, health checks, and SNI passthrough where required).
	•	Observability should include per-cluster and cross-cluster metrics to detect asymmetric failures (Cluster A → B vs B → A).
