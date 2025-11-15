# Istio & Kubernetes 

## Agenda

1. Grundlagen ServiceMesh & Istio
   * [Einführung in Istio & Service Mesh-Architekturen](/istio/overview/introduction.md)
   * [Warum ein Service Mesh?](istio/overview/02-warum-servicemesh.md)
   * [Herausforderung & Vorteile](istio/overview/03-herausforderungen-vorteile.md)
   * [Architektur & Komponenten von Istio](istio/overview/04-architektur-komponenten.md)
   * [Vergleich mit Linkerd, Cilium, Consul](istio/overview/05-vergleich-linkerd-cilium-consul.md)

1. Setup Cluster
   * [Self-Service Cluster ausrollen](/istio/training-stack/install.md)
   * [Self-Service Cluster destroy](/istio/training-stack/uninstall.md)
    
1. Installation & Bereitstellung von Istio
   * [Systemanforderungen & Kubernetes-Cluster-Vorbereitung]
   * Istio-Installation mit istioctl
   * Istio-Installation mit Helm
   * Installation mit Istio-Operator
   * Einführung in Sidecar Injection & Proxy-Konzepte

1. Steuerung des Netzwerkverhaltens in Istio
   * Virtual Services, Destination Rules & Gateway-API
   * Load Balancing, Header- & Pfadbasiertes Routing
   * Erstellen von Routing- & Load-Balancing-Regeln

1. Sicherheit, Fehlertoleranz & Observability
   * Sicherheit & Zero Trust mit Istio
   * Service-zu-Service-Sicherheit mit mTLS (Mutual TLS)
   * Zugriffskontrolle mit RBAC & JWT-Authentifizierung
   * Istio Authorization Policies (ingress/egress Security)
   * Sichere Service-Kommunikation mit Istio konfigurieren

1. Service Resilience & Fehlertoleranz
   * Circuit Breaker, Retries, Rate Limiting
   * Fehlerinjektion & Chaos Engineering mit Istio
   * Belastungstests mit Fehlerinjektionen

1. Monitoring, Logging & Observability
   * Distributed Tracing mit Jaeger
   * Metriken & Dashboards mit Prometheus & Grafana
   * Service-Visualisierung mit Kiali
   * Analyse & Debugging von Service-Mesh-Daten

1. Skalierung, Erweiterbarkeit & Performance-Optimierung
   * Skalierung von Istio & Performance-Optimierung
   * Sidecar-Overhead & Ressourcenoptimierung
   * Ambient Mesh (sidecar-less Istio für Performance-Gewinn)
   * Multi-Cluster- & Hybrid-Umgebungen mit Istio
   * Istio Federation & Cross-Cluster Traffic

1. Erweiterte Routing-Techniken & Traffic-Optimierung
   * Canary Releases & Progressive Deployments
   * A/B-Tests & Traffic Mirroring
   * Blue-Green- und Canary-Deployments mit Istio

1. Erweiterbarkeit & Automatisierung mit Istio
   * WebAssembly (Wasm) für Istio-Erweiterungen
   * [Wo läuft WASM (WebAssembly) im Rahmen von istio ?](istio/wasm/plugins/where-is-the-wasm-runtime.md)
   * Automatisierung mit GitOps & ArgoCD
   * Eigene Istio-Erweiterungen mit WebAssembly schreiben

1. FAQ & Best Practices
   * Zusammenfassung der wichtigsten Erkenntnisse
   * Diskussion von Best Practices für Enterprise-Anwendungen
   * Fragen & weiterführende Ressourcen


## Backlog 

  1. Vorbereitung
     * [Self-Service Cluster ausrollen](/monitoring/training-stack/install.md)
     * [Self-Service Cluster destroy](/monitoring/training-stack/uninstall.md)

  1. Grundlagen / Recap
     * [Architektur Kubernetes](/kubernetes/architecture.md)

  1. Starting
     * [The truth about security](/security/truth.md)
     * [The architecture of Kubernetes](kubernetes/architecture.md)
     * [Architecture DeepDive](https://github.com/jmetzger/training-kubernetes-advanced/assets/1933318/1ca0d174-f354-43b2-81cc-67af8498b56c)
     * [Layers to protect (Security)](security/overview/layers-2-protect.md)
     * [AttackVectors](security/overview/attack-vectors.md)
     * [The route from development to production to secure](security/overview/route-2-production.md)
     * [Kill Chain](kill-chain.md)

  1. Benchmarking / Security Scans
     * [CIS Benchmarking Kubernetes](https://www.cisecurity.org/benchmark/kubernetes)
     * [Exercise: kube-bench - scanning with cis-benchmark-kubernetes](security/cis-benchmark-kubernetes/exercise.md)
     * [OWASP - Top Ten Kubernetes Risks](https://owasp.org/www-project-kubernetes-top-ten/2022/en/src/K09-misconfigured-cluster-components)

  1. Scanning for outdated versions of images and helm-charts
     * [Scanning for outdated versions](security/veraltete-images-und-helm-charts-finden.md)

  1. Encrypting Node-2-Node traffic (wireguard with calico)
     * [Securing Node-2-Node with calico and wireguard](security/wireguard-calico/overview-and-exercise.md)

  1. Checklist
     * [Security Checklists](/security/checklist/security-checklist.md)
   
  1. Getting hacked
     * [Why is a cluster so rewarding to hack](security/getting-hacked/kubernetes-rewarding.md)
     * [Starting with Tesla](https://arstechnica.com/information-technology/2018/02/tesla-cloud-resources-are-hacked-to-run-cryptocurrency-mining-malware/)

  1. Category 1 by Layer: OS / Kernel
     * [Securing the OS and the Kernel](security/os-kernel/01-harden-os-kernel.md)
     * [Kernel Hardening Checker](kernel/hardening.md)
   
  1. Category 2 by Layer: Cluster
     * [Securing kubelet](security/cluster/components/kubelet.md)
     * [Least Privileges with RBAC](kubernetes/rbac/00-rbac-and-least-privileges.md)
     * [Admission Controller](/security/admissionController/01-overview.md)
   
  1. Category 3 by Layer: Pods Container
     * [The runAs Options in SecurityContext](security/by.layer/pods-container/runAs/overview.md)
     * [sysctls in pods/containers](security/by.layer/pods-container/sysctls/overview.md)
     * [Overview capabilities](security/by.layer/pods-container/capabilities/00-overview.md)
     * [Start pod without capabilities & how can we see this](security/by.layer/pods-container/capabilities/01-nocap.md)
     * [Hacking and exploration session HostPID](explore/01-hack-session-hostpid.md)
     * [Disable ServiceLinksEnable false](https://github.com/jmetzger/training-kubernetes-security-en/blob/main/security/by.layer/pods-container/enableServiceLinks/disable-howto-and-why.md
) 
     * [Great but still alpha User Namespaces]()
     * [Use kubectl in containe inClusterConfig](security/kubectl-within-container.md)
    
  1. Reaction 
     * [The Audit Logs](/security/reaction/auditlog.md)

  1. Securing cluster with iptables
     * [firewall Regeln festlegen für die Cluster Nodes](/security/firewall-linux.md)

  1. RBAC
     * [How does RBAC work ?](kubernetes/rbac/01-how-does-rbac-work.md)
     * [Where does RBAC play a role ?](kubernetes/rbac/02-where-does-rbac-play-a-role.md)
     * [kubeconfig decode certificate](kubernetes/rbac/decode-local-certificate.md)
     * [kubectl check your permission - can-i](kubernetes/rbac/can-i.md)
     * [use kubectl in pod - default service account](/kubernetes/rbac/pod-automount-sa.md)
     * [create user for kubeconfig with using certificate](kubernetes/rbac/create-kubeconfig-with-cert.md)
     * Components / moving parts of RBAC
     * [practical exercise rbac](kubernetes/rbac-create-user-kubernetes-1-25.md)

  1. Obey Security Policies (AdmissionControllers)
     * [Admission Controller](/security/admissionController/01-overview.md)
     * PSA (PodSecurity Admission)
     * [Exercise with PSA](kubernetes-security/pod-security-admission.md)
     * [OPA Gatekeeper 01-Overview](/security/admissionController/opa-gatekeeper/01-overview.md)
     * [OPA Gatekeeper 02-Install with Helm](/security/admissionController/opa-gatekeeper/02-install.md)
     * [OPA Gatekeeper 03-Simple Exercise](/security/admissionController/opa-gatekeeper/05-example-exercise.md)
     * [OPA Gatekeeper 04-Example-Job-Debug](/security/admissionController/opa-gatekeeper/06-example-job-debug.md)
     * [Connaisseur: Verifying images before Deployment](/security/admissionController/connaisseur-image-verification/02-walkthrough.md)
   
  1. Obey Security Policy (AdmissionControllers - Part II)
     * [Exercise Kyverno - non-root-pod](/security/admissionController/kyverno/exercise-pod-non-root.md)
   
  1. Pod Security
     * [Automount ServiceAccounts or not ?](security/by.layer/pods-container/serviceAccount/do-not-mount-if-not-needed.md)
     * Does every pod need to access the kubenernetes api server?
   
  1. Unprivilegierte Pods/Container
     * [Which images to use ?](security/unprivileged-containers/which-images.md) 
     * How can i debug non-root - Container/Pods?
    
  1. The SecurityContext
     * seccomp
     * privileged/unprivileged
     * [appArmor example](https://kubernetes.io/docs/tutorials/security/apparmor/)
     * SELinux
     
  1. Network Policies
     * Understand NetworkPolicies
     * [Exercise NetworkPolicies](kubernetes-networkpolicy/00-simple-exercises-group-en.md)
     * [CNI Benchmarks](https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-40gbit-s-network-2024-156f085a5e4e)

  1. ServiceMesh
     * [Why a ServiceMesh ?](istio/overview/benefits-of-a-service-mesh.md)
     * [How does a ServiceMeshs work? (example istio](/istio/overview/overview-classic-sidecar.md)
     * [istio security features](istio/overview/security-features.md)
     * [istio-service mesh - ambient mode](/istio/overview/ambient-mode.md)
     * [Performance comparison - baseline,sidecar,ambient](/istio/overview/performance-comparison-baseline-sidecar-ambient.md)

  1. Passwörter speichern
     * [HashiCorp Vault as Password Safe](security/hashicorp-vault/overview.md)
  
  1. Image Security
     * [When to scan ?](security/scanning-containers/01-when-to-scan.md)
     * [Example Image Security Scanning - using gitlab and trivy](security/scanning-containers/02-example-trivy-gitlab.md)

  1. Hacking Sessions
     * [Hacking with HostPID](/security/explore/01-hack-session-hostpid.md)
    
  1. Extras
     * [Canary deployment with basic kubernetes mechanisms ](kubectl-examples/08-canary-deployment.md)

  1. Documentation
     * [Great video about attacking kubernetes - older, but some stuff is still applicable](https://www.youtube.com/watch?v=HmoVSmTIOxM)
     * [Straight forward hacking session of kubernetes](https://youtu.be/iD_klswHJQs?si=97rWNuAbGjLwCjpa)
     * [github with manifests for creating bad pods](https://bishopfox.com/blog/kubernetes-pod-privilege-escalation#pod8)
