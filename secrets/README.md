# Secrets

Plaintext secrets and TLS private keys are NOT stored here.

TODO: version-control secrets via Sealed Secrets or SOPS, e.g.:
  - local-tls (wildcard *.local) in nginx-gateway ns
  - harbor-ca / docker-registry

Workflow (Sealed Secrets):
  kubectl create secret tls local-tls --cert=local.crt --key=local.key \
    -n nginx-gateway --dry-run=client -o yaml \
    | kubeseal --format yaml > local-tls-sealed.yaml
  # commit local-tls-sealed.yaml ONLY
