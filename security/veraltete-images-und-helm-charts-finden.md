# Tools zum Aufspüren von veralteten Helm - Charts und Images 

## Empfohlene Tools 

  * Trivy Operator 
  * kube-version-tracker  

## Trivvy -> speziell Trivvy Operator 

```
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm repo update
```

```
helm install trivy-operator aqua/trivy-operator \
     --namespace trivy-system \
     --create-namespace \
     --version 0.29.3
```

## Vorgehen. In regelmäßigen Abständen per cronjob scannen 


## VulnerabilityReport werden erstellt 

```yaml

apiVersion: aquasecurity.github.io/v1alpha1
kind: VulnerabilityReport
metadata:
  name: replicaset-nginx-6d4cf56db6-nginx
  namespace: default
  labels:
    trivy-operator.container.name: nginx
    trivy-operator.resource.kind: ReplicaSet
    trivy-operator.resource.name: nginx-6d4cf56db6
    trivy-operator.resource.namespace: default
    resource-spec-hash: 7cb64cb677
  uid: 8aa1a7cb-a319-4b93-850d-5a67827dfbbf
  ownerReferences:
    - apiVersion: apps/v1
      controller: true
      kind: ReplicaSet
      name: nginx-6d4cf56db6
      uid: aa345200-cf24-443a-8f11-ddb438ff8659
report:
  artifact:
    repository: library/nginx
    tag: '1.16'                                   # 👈 indicates this outdated version
  registry:
    server: index.docker.io
  scanner:
    name: Trivy
    vendor: Aqua Security
    version: 0.35.0
  summary:
    criticalCount: 2
    highCount: 0
    mediumCount: 0
    lowCount: 0
    unknownCount: 0
  vulnerabilities:
    - vulnerabilityID: CVE-2019-20367
      resource: libbsd0
      severity: CRITICAL
      installedVersion: 0.9.1-2
      fixedVersion: 0.9.1-2+deb10u1
      primaryLink: https://avd.aquasec.com/nvd/cve-2019-20367
      target: library/nginx:1.21.6               # indicates version where it's fixed
    - vulnerabilityID: CVE-2018-25009
      resource: libwebp6
      severity: CRITICAL
      installedVersion: 0.6.1-2
      fixedVersion: ""                          # no patch yet
      primaryLink: https://avd.aquasec.com/nvd/cve-2018-25009
      target: library/nginx:1.16

```

## Nutzung von Trivy und kube-version-tracker 


### Variante 1: CronJob - 

   * Die entsprechende Vulnerability - Reports auswerten und dann bei incident an Ziel schicken (Slack)

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vulnreport-check
  namespace: monitoring
spec:
  schedule: "0 7 * * *"  # täglich um 07:00 Uhr
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: vuln-checker
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              echo "🔍 Kubernetes Trivy Vulnerability Summary:" > /tmp/summary.txt

              # Alle Reports iterieren
              for report in $(kubectl get vulnerabilityreports -A -o=jsonpath='{range .items[*]}{.metadata.namespace}{";"}{.metadata.name}{"\n"}{end}'); do
                ns=$(echo "$report" | cut -d';' -f1)
                name=$(echo "$report" | cut -d';' -f2)

                # Alle CRITICAL + HIGH CVEs extrahieren
                vulns=$(kubectl get vulnerabilityreports -n "$ns" "$name" -o=jsonpath="{range .report.vulnerabilities[?(@.severity=='CRITICAL' || @.severity=='HIGH')]}{.vulnerabilityID}{' ('}{.severity}{')'}{'\n'}{end}")

                if [ ! -z "$vulns" ]; then
                  while IFS= read -r v; do
                    echo "🔴 $v in $ns/$name" >> /tmp/summary.txt
                  done <<< "$vulns"
                fi
              done

              # Wenn kritische Funde vorhanden, an Slack senden
              if [ -s /tmp/summary.txt ]; then
                payload=$(cat /tmp/summary.txt | sed 's/"/\\"/g' | tr '\n' '\\n')
                curl -X POST -H 'Content-type: application/json' \
                  --data "{\"text\":\"$payload\"}" "$SLACK_WEBHOOK_URL"
              else
                echo "✅ Keine kritischen Sicherheitslücken gefunden."
              fi
            env:
            - name: SLACK_WEBHOOK_URL
              valueFrom:
                secretKeyRef:
                  name: slack-webhook
                  key: url
          restartPolicy: OnFailure

```
