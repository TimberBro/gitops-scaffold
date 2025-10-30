Для установки релизов используется kustomize
`kustomize build --enable-helm infrastructure/overlays/local/argocd --load-restrictor=LoadRestrictionsNone | kubectl apply -f -`
