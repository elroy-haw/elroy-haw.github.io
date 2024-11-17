---
title: "Headlamp Setup Guide"
date: 2024-06-01
tags: ["kubernetes", "k3s", "authentik", "headlamp", "dashboard", "oidc", "oauth2"]
---

## Introduction

For the longest time, I have been managing my K3s cluster directly from my laptop (favorite IDE of choice: K9s) either through the local network or Tailscale VPN. However, there are times when I just want to check on some of my services running in the cluster, straight from my phone without the hassle of bringing around my laptop. Therefore, I began searching for a web dashboard which I can access easily on my phone.

Most dashboards these days come with standard set of functionalities, which are sufficient for my basic use-cases. Common ones include [Kubernetes dashboard](https://github.com/kubernetes/dashboard) and [Rancher dashboard](https://github.com/rancher/dashboard). My requirements were simple: check the logs occasionally, integrates with my identity provider (Authentik), scale in or out my workloads, and consume minimal resources. Out of the box, Kubernetes dashboard does not support OIDC, and I would have to spin up an [OAuth2 proxy](https://github.com/oauth2-proxy/oauth2-proxy) alongside if I wanted it to work with Authentik. It seemed like Rancher dashboard would have fit the bill, but I decided to give [Headlamp](https://github.com/headlamp-k8s/headlamp) a shot.

## Configuring Authentik

My Authentik configurations are managed via Terraform. You can configure the same from the UI as well.

1. Create an OAuth2 provider.

```hcl
resource "authentik_provider_oauth2" "provider" {
  name                  = "headlamp"
  authorization_flow    = data.authentik_flow.authz_flow.id
  authentication_flow   = data.authentik_flow.authn_flow.id
  client_id             = "REPLACE_ME"
  client_secret         = "REPLACE_ME"
  property_mappings     = data.authentik_scope_mapping.oauth2_scope_mappings.ids
  redirect_uris         = ["https://headlamp.replace.me/oidc-callback"]
  access_code_validity  = "minutes=1"
  access_token_validity = "minutes=30"
  signing_key           = data.authentik_certificate_key_pair.self_signed_key_pair.id
}

data "authentik_flow" "authz_flow" {
  slug = "default-provider-authorization-explicit-consent" # you can use your own custom authZ flow
}

data "authentik_flow" "authn_flow" {
  slug = "default-authentication-flow" # you can use your own custom authN flow
}

data "authentik_scope_mapping" "oauth2_scope_mappings" {
  managed_list = [
    "goauthentik.io/providers/oauth2/scope-email",
    "goauthentik.io/providers/oauth2/scope-openid",
    "goauthentik.io/providers/oauth2/scope-profile"
  ]
}

data "authentik_certificate_key_pair" "self_signed_key_pair" {
  name = "authentik Self-signed Certificate"
}
```

2. Create an application.

```hcl
resource "authentik_application" "application" {
  name              = "headlamp"
  slug              = "headlamp"
  meta_launch_url   = "https://headlamp.replace.me"
  meta_icon         = "https://raw.githubusercontent.com/headlamp-k8s/headlamp/c3d2412cbec3af786730958931f65e606d398a82/docker-extension/headlamp.svg"
  protocol_provider = authentik_provider_oauth2.provider.id
}
```

## Configuring K3s server

My K3s server is currently configured by a config file located in `/etc/rancher/k3s/config.yaml`. You can use whichever way you have previously configured your K3s server e.g. CLI, environment variables, to configure the same.

1. Add OIDC config to API server.

```yaml
kube-apiserver-arg:
  - oidc-issuer-url=https://auth.replace.me/application/o/headlamp/
  - oidc-client-id=REPLACE_ME
  - oidc-username-claim=email
```

2. Restart K3s server.

```sh
sudo systemctl restart k3s
```

With this, you can use access tokens from Authentik to authenticate with the API server, and bind roles at cluster or namespace level to the emails of your users in Authentik. Note that this can be use outside of Headlamp as well!

## Deploying Headlamp

In my setup, I have created the ClusterRoleBinding and Secret outside of the Helm chart. This allows me to control the User/Group to give cluster admin access to, and to leverage SealedSecret for my GitOps workflow.

1. ClusterRoleBinding configuration.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: headlamp-admin
subjects:
  - kind: User
    name: your_email@replace.me # retrieved from the email scope
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

2. Secret configuration.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: headlamp-oidc
type: Opaque
stringData:
  clientID: REPLACE_ME
  clientSecret: REPLACE_ME
  issuerURL: https://auth.replace.me/application/o/headlamp/
  scopes: openid,profile,email
```

3. Helm values configuration.

```yaml
replicaCount: 1

config:
  oidc:
    secret:
      create: false
      name: headlamp-oidc

# Due to [this issue](https://github.com/headlamp-k8s/headlamp/issues/2022), we need to manually add the OIDC env vars for now
env:
  - name: OIDC_CLIENT_ID
    valueFrom:
      secretKeyRef:
        name: headlamp-oidc
        key: clientID
  - name: OIDC_CLIENT_SECRET
    valueFrom:
      secretKeyRef:
        name: headlamp-oidc
        key: clientSecret
  - name: OIDC_ISSUER_URL
    valueFrom:
      secretKeyRef:
        name: headlamp-oidc
        key: issuerURL
  - name: OIDC_SCOPES
    valueFrom:
      secretKeyRef:
        name: headlamp-oidc
        key: scopes

clusterRoleBinding:
  create: false

ingress:
  enabled: true
  ingressClassName: nginx # my ingress of choice
  hosts:
    - host: headlamp.replace.me
      paths:
        - path: /
          type: Prefix
  tls:
    - hosts:
        - headlamp.replace.me

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

Now you can login to your Headlamp instance via Authentik and view all your resources!

## References

- [Kubernetes authentication with OIDC tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
- [Passing additional arguments for API server in K3s](https://docs.k3s.io/cli/server#customized-flags-for-kubernetes-processes)
