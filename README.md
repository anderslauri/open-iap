# Open Programmatic Identity Aware Proxy for Google Cloud
Open implementation of Programmatic Identity Aware Proxy for Google Cloud workload to workload authentication using Google Service Account.
Performs validity and signature verification of Google Service Account JWT - while also verifying subject of claim `email` has role bidning
`roles/iap.httpsResourceAccess` inside project. Role `roles/iap.httpsResourceAccess` (as with Identity Aware Proxy) is required for authentication.
Conditional expressions are supported. A role binding can be restricted to only allow communication to a single specific `host` and `path`.

The following token types are supported for Google Service Account:

- **ID-Token**
- **Self-Signed JWTs**

Please reference [Google Cloud Token Types][Google Cloud Token Types] for more information. For **Self-Signed JWTs** please ensure to follow
[specification as required by Google][Self-Signed JWTs].

## Why?
- Identity Aware Proxy for Google Cloud is only available in BeyondCorp Enterprise.
- There is other solutions available on GitHub, however, none of are really suitable and made compatible with existing IAM on Google Cloud.

## Authentication of Google Cloud Service Account
1. Signature verification using `JWK`. Source of `JWK` is determined given type of JWT.
2. `iat` and `exp` claim verification. Leeway for `JWT` is configurable. Default is 1 minute.
3. `aud` claim must be equal to request url.
4. Role `roles/iap.httpsResourceAccessor` is verified given subject of claim email. Role binding can be granted directly on project,
   or indirectly, via membership in Google Workspace group.

:exclamation: Steps `{1..3}` follow [JWT-verification as described by Google Cloud][JWT-Verification]. Step `4` is custom step following
the ideas of `Identity Aware Proxy`.

:exclamation: After successful `{1..4}`. Value of claim `email` is cached. Key is hash, in `SHA256`, of `{JWT || Request URL}`. 
`ttl` for cache value is `exp - <interval of cleaning routine>`. Once token is found in cache - only `exp` claim validity and step `4` is performed per each request.

:exclamation: The code strives to retain a performance aware profile. Caching is used aggressivly on multiple layers to ensure an overall
low 90th percentile response time. To benefit from cache locality, use a ring hash for routing.

## Role bindings
:warning: All role bindings are consumed asynchronously given a defined time interval (see configuration). This may or
may not be acceptable - depends on your choice. Bindings are kept in memory for performance reasons. Default interval is `5min`.

### Conditional expressions
`request.path`, `request.host` and `request.time` are supported with conditional expressions with role `roles/iap.httpsResourceAccessor`. 
If role binding has conditional expression, this conditional expression is compiled and evaluated in memory using `cel-go`. All conditional
expressions are only compiled once - after first compilation - the program (representing conditional expression) is cached for performance reasons.

## How to run
:exclamation: Use `Dockerfile` as example.

Configuration uses [pkl-lang][pkl-lang]. `pkl` must be available in `$PATH`. As well, `default_config.pkl` 
and `app_config.pkl` must be present in same directory as application executable when starting application.

### Required Prerequisites
* **Groups Reader** is required on Google Workspace. Reference [Google Workspace Administrator Roles][Google Workspace Administrator Roles].
* **resourcemanager.projects.getIamPolicy** is required to list all bindings for role `roles/iap.httpsResourceAccess` 
for Google Service Account inside project. Usage of custom role is recommended!
* **Admin API** and **Cloud Resource Manager API** is required on project.

## API 

### /auth (GET)
Authentication endpoint. Return code `200 OK` given successful authentication, else `401 Unauthorized`.

#### Zero Trust with NetworkPolicy and nginx
Use the following example (as inspiration), to enable secure, zero trust based communication of workload to workload communication to services on `GKE`.

##### Configuration NetworkPolicy
Enforce connectivity restrictions to service, traffic must only be permitted through `namespace` of `nginx`.

````
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-service
spec:
  ingress:
    - from:
        - namespaceSelector:
            matchExpressions:
              - key: kubernetes.io/metadata.name
                operator: In
                values:
                - nginx
      ports:
      - port: 8080
        protocol: TCP
  podSelector:
    matchLabels:
      app: my-service
  policyTypes:
  - Ingress
`````

##### Global nginx configuration
Define following configuration on `controller` of `nginx`. Ensuring, requests (all `ingress` registered with `controller`), only are permitted given valid authentication.

```
global-auth-snippet: |
  proxy_set_header Proxy-Authorization $http_proxy_authorization;
global-auth-url: http://<service name>.<namespace>:8080/auth
```

#### Required headers
:warning: `X-Original-URL`, i.e. from `nginx` has assumed trust.

1. `Authorization` or `Proxy-Authorization`.
2. `X-Original-URL` is configured to be present. This can be changed using `HeaderMapping` in configuration.

### /healthz (GET)
Kubernetes health endpoint for liveness and readiness. Return code `200 OK`.

## Future changes
In scope for `open-iap`.

1. Role `roles/iap.tunnelResourceAccessor` for `TUNNEL`-proxy.
2. Role bindings on project level are only used. Folders and organization must be implemented as optional feature.
3. Consume `IAM Role Audit Events`. Ensuring close to real time changes to policy bindings.

[Google Workspace Groups API]: <https://developers.google.com/admin-sdk/directory/reference/rest/v1/groups> "Google Workspace Groups API"
[Google Workspace Administrator Roles]: <https://support.google.com/a/answer/2405986> "Google Workspace Administrator Roles"
[Google Cloud Token Types]: <https://cloud.google.com/docs/authentication/token-types> "Google Cloud Token Types"
[Programmatic Authentication]: <https://cloud.google.com/iap/docs/authentication-howto#authenticating_from_proxy-authorization_header> "Programmatic Authentication"
[JWT-verification]: <https://cloud.google.com/docs/authentication/token-types#id-aud> "JWT-verification"
[cel-go]: <https://github.com/google/cel-go> "cel-go"
[pkl-lang]: <https://pkl-lang.org/go/current/index.html> "pkl-lang"
[Self-Signed JWTs]: <https://cloud.google.com/iam/docs/create-short-lived-credentials-direct#create-jwt> "Self-Signed JWTs"
