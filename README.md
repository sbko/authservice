# Authservice

Helm chart to deploy authservice from the istio-ecosystem.

# Table of Contents
- [Prerequisites](#pre-requisites)
- [Iron Bank](#iron-bank-authservice)
- [Deployment](#deploy-authservice)
- [Values](#values-authservice)
- [Chains](#chains-authservice)
- [Istio configuration](docs/README.md)
- [Keycloak configuration](docs/KEYCLOAK.md)

## Pre-Requisites

* Kubernetes Cluster deployed
* Kubernetes config installed in `~/.kube/config`
* Helm installed

Install Helm

https://helm.sh/docs/intro/install/


## Iron Bank

You can `pull` the registry1 image(s) [here](https://registry1.dso.mil/harbor/projects/3/repositories/istio-ecosystem%2Fauthservice) and view the container approval [here](https://ironbank.dso.mil/repomap/istio-ecosystem/authservice).

## Deployment

```bash
git clone https://repo1.dso.mil/platform-one/big-bang/apps/core/authservice.git
cd authservice
helm install authservice chart
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Configurable [affinity][] for the authservice pod(s)
| chains | object | `{}` | Configurable chains for the authservice daemon to act on, see [Chains](#chains-authservice) & [values.yaml](./chart/values.yaml)
| fullnameOverride | string | `authservice` | Configurable override used when templating out fully qulified app name
| global | object | ` ` | Configurable global settings to apply to each generated authservice chain see [values.yaml](./chart/values.yaml)
| istio.namespace | string | `istio-system` | Configurable namespace where istio is deployed
| image.repository | string | `registry1.dso.mil/ironbank/istio-ecosystem/authservice` | Configurable repository from where to pull the image (first half of image from docker pull command)
| image.pullPolicy | string | `IfNotPresent` | Configurable [imagePullPolicy][] setting 
| image.tag | string | `v0.3.1` | Configurable setting for specifying what tag for the repository above
| imagePullSecrets | string | `[ ]` | Configurable [imagePullSecrets][] setting for specifying name of kubernetes secret used to pull from private registry
| nameOverride | string | `authservice` | Configurable name used when creating name of helm chart install
| nodeSelector | object | `{}` | Configurable [nodeSelector][] to target specific nodes to run on based on resource labels
| podAnnotations | object | `{}` | Configurable annotations to apply to authservice pod(s)
| podSecurityContext | object | `{}` | Configurable [securityContext][] to set for authservice pod(s)
| replicaCount | string | `1` | Configurable number of replicas (pods) to run
| resources | object | `{}` | Configurable settings for setting container [resource][] limits and requests
| serviceAccount.create | boolean | `true` | Configurable setting to create service account for deployment
| serviceAccount.annotations | object | `{}` | Configurable annotations to add to metadata of service account for deployment
| serviceAccount.name | string | `""` | Configurable name of service account for deployment, if not set, it's populated from the fullNameOverride set
| service.type | string | `ClusterIP` | Configurable type of [service][] resource to create for authservice
| service.port | string | `10003` | Configurable port for the service resource
| selector.key | string | `protect` | Configurable key of a label on a pod which enovy will use to place authservice in routing chain eg: label: protect=keycloak
| selector.value | string| `keycloak` | Configurable value of a label on a pod which envoy will use to place authservice in routing chain eg: label: protect=keycloak
| tolerations | object | `{}` | Configurable [tolerations][] for the authservice pod(s)

## Chains

# Individual chains.  Must have a `name` value (in this case `minimal`) and a `callback_uri`
```yaml
chains:
  minimal:
    # Inherits other settings from global: block in values
    callback_uri: https://minimal.bigbang.dev/
  full:
    # Has all settings laid out
    match:
      header: ":authority"
      prefix: "localhost"
    client_id: platform1_a8604cc9-f5e9-4656-802d-d05624370245_hello-world-authservice
    client_secret: secret_value
    callback_uri: https://localhost/login
    cookie_name_prefix: "hello-world"
    redis_server_uri: tcp://localhost:6379/
    logout:
      path: "/logout"
    oidc:
      host: local_oidc_host
      realm: local_oidc_relm
    jwks: local_jwks
    certificate_authrority: "-----BEGIN CERTIFICATE-----\nMIIE4jCCAsqgAwIBAgIBATANBgkqhkiG9w0BAQsFADARMQ8wDQYDVQQDEwZzZm8t\nY2EwHhcNMTkxMjIwMDAxNjI1WhcNMjEwNjIwMDAxNjIxWjARMQ8wDQYDVQQDEwZz\n...\nwPAYZhormzV5LxXMSd3BMdyexNvNiGffULhnYEebFI9GouFxV1LPdVu058LRW/db\n6PDm7+GEq/CcQhTgYOELmmcnC89zNxcCXiahxqKIMTuid295N4NldyK/IT4Tn4GN\nVknTT/Hr\n-----END CERTIFICATE-----"
```

## AuthN and AuthZ

In addition to an Authservice filter, all applications _must_ have an AuthN `RequestAuthentication` resource in their namespace for Authservice to run (and to validate the JWT). Additionally apps must have an AuthZ `RequestAuthentication` policy to ensure all requests have a JWT.

See [`authn-auth.yaml`](examples/authn-authz.yaml) for an example configuration to be installed in each app's namespace.

The full authentication flow is:  
Token acquisition (Authservice) → AuthN [`RequestAuthentication`](https://istio.io/latest/docs/reference/config/security/request_authentication/) → AuthZ [`AuthorizationPolicy`](https://istio.io/latest/docs/reference/config/security/authorization-policy/) → app

* The EnvoyFilter is configured as INSERT_BEFORE `envoy.filters.http.jwt_authn`. Without an `RequestAuthentication` resource, your sidecars won't have `jwt_authn` filter and **Authservice won't be installed in the filter chain either**.
* Authservice **can be bypassed** if requests have a preexisting Authorization header. Its _only_ role is acquiring JWT tokens. (However [a bug in Authservice](https://github.com/istio-ecosystem/authservice/issues/141) forces some requests through Authservice unnecessarily.)
* `RequestAuthentication` **can be bypassed** if requests have no Authorization header or a non-bearer Authorization header. Its _only_ role is validating the issuer/audience/etc of JWTs. It will block requests with an expired JWT or with the wrong issuer, etc.
* `AuthorizationPolicy` actually enforces policy on requests. It is used to ensure that all requests have a JWT (and any other specified policy on the token).
* Using an `AuthorizationPolicy` ensures a fail-closed configuration. If Authservice filters are misconfigured, AuthZ will block the request with a 403 `RBAC: access denied`.

[affinity]: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
[imagePullPolicy]: https://kubernetes.io/docs/concepts/containers/images/#updating-images
[imagePullSecrets]: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret
[securityContext]: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod
[service]: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
[resource]: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-requests-and-limits-of-pod-and-container
[nodeSelector]: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector
[tolerations]: https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/