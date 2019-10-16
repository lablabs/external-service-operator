# External Service Operator
This project is based on  [helm operator-sdk](https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md).

External service operator maps external services to the kubernetes service object and manage related endpoint object. Endpoint object are populated and synced with external endpoints by [Kube Endpoint Manager](https://github.com/lablabs/kube-endpoint-manager). External services are services that lives outside of kubernetes cluster. Operator privides following:
* Creates kubernetes service account from ExternalService CRD
* Creates empty [1] kubernetes endpoints
* Synchronize autodiscovered external endpoints with kubernetes endpoint

One of the main use case of this project is to have in cluster ingress controller or api gateway that can discover external targets from kubernetes service object.

---
[1] - kubernetes endpoint cannot be empty, addresses or nonReadyAddresses must be specified, to workaround this limitation operator sets polaceholder IP 0.0.0.1 (yes! because kubernetes does not allow "special" addresses like 127.0.0.1, ...) to the nonReadyAddresses

## ExternalService CRD Description
Together with the Operator a new CRD named [ExternalService](deploy/crds/service_v1alpha1_external_crd.yaml) will be deployed.

```yaml
---
apiVersion: service.lablabs.io/v1alpha1
kind: ExternalService
metadata:
  name: test
spec:
  type: static
  endpoints:
    - 1.1.1.1
    - 2.2.2.2
  ports:
    http: 80
    https: 443
---
apiVersion: service.lablabs.io/v1alpha1
kind: ExternalService
metadata:
  name: test
spec:
  type: openstack
  sync: 1
  filters:
    name: '^instance-name-.*$'
    metadata:
      - '.*service$:pool-test.*'
  ports:
    http: 80
    https: 443
---
```
#### Common
Parameter | Description | Required
--- |--- | --- |
`spec.type` | External service type [static\|openstack] | yes
`spec.ports` | External service/endpoint named ports | yes
#### Type: static
Parameter | Description | Required
--- |--- | --- |
`spec.endpoints` | Static endpoints | yes

#### Openstack
Parameter | Description | Required
--- |--- | --- |
`spec.sync` | Sync loop period | yes
`spec.filters.name` | Selector rule based on VM instance name. Filter value should be regexp | no
`spec.filters.metadata` | Selector rule based on VM instance metadata. Metadata key and metadata value should be specified as regexp | no
To override [Kube Endpoint Manager](https://github.com/lablabs/kube-endpoint-manager) deployment specify `spec.manager.*`, for available options see chart default values.

## Installation
#### Openstack VM autodiscovery
```sh
kubectl create -f deploy/crds/service_v1alpha1_external_crd.yaml
kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/secret_openstack.yaml # <-- EDIT! OpenStack API credentials
kubectl create -f deploy/configmap.yaml # <-- EDIT! Additional env variables for endpoint manager
kubectl create -f deploy/operator.yaml
```

## Build
```sh
operator-sdk build ...
```
For more information see helm operator-sdk build and run [guide]((https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md#build-and-run-the-operator)).

## Future:
- ExternalService health checks