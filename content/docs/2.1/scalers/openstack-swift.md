+++
title = "OpenStack Swift"
layout = "scaler"
availability = "v2.1+"
maintainer = "Community"
description = "Scale applications based on the count of objects in a given OpenStack Swift container."
go_file = "openstack_swift_scaler"
+++

### Trigger Specification

This specification describes the `openstack-swift` trigger for OpenStack Swift containers. It scales based on the count of objects in a given container.

```yaml
triggers:
- type: openstack-swift
  metadata:
    swiftURL: http://localhost:8080/v1/b161dc815cd24bda84d94d9a0e73cf78  # Required
    containerName: my-container # Required
    objectCount: "2"  # Optional
    objectPrefix: "my-prefix" # Optional
    objectDelimiter: "/"  # Optional 
    objectLimit: "10" # Optional
    onlyFiles: "true" # Optional
    timeout: "2"  # Optional
```

> Please, always provide the protocol (http or https) when specifying URLs. This is needed due to Go URL parsing issues :sweat_smile:

**Parameter list:**

- `swiftURL` - The URL to query the Swift API. It must contain the hostname, the Swift port, the API version and the account ID. The pattern is: `http://<host>:<swift_port>/<swift_version>/<account_id>`.
- `containerName` - Name of Swift container in an OpenStack account.
- `objectCount` - Average target value to trigger scaling actions. (Default: `2`, Optional)
- `objectPrefix` - Prefix for the object. Only objects with this prefix will be returned. Use this prefix to specify sub-paths for the objects. (Default: `""`, Optional)
- `objectDelimiter` - Delimiter for identifying the object prefix. It is the character used to split object names. (Default: `""`, Optional)
- `objectLimit` - The maximum number of objects returned by the API. By default, the Swift API only returns up to 10000 names. (Default: `""`, Optional)
- `onlyFiles` - Specifies if the scaler should be triggered only by the number of files, without considering folders. Inside a container, one can have files and folders. Folders (empty or not) are counted as objects, just as files are. If one wants to scale based on only files, this parameter must be set to `true`. (Values: `true`, `false`, Default: `false`, Optional)
- `timeout` - The timeout, in seconds, for the HTTP client requests that will query the Swift API. (Default: `30`, Optional)

For more information about `prefix`, `delimiter`, and `limit`, please, refer to the [Object Store API](https://docs.openstack.org/api-ref/object-store/).

### Authentication Parameters

To authenticate, this scaler uses tokens. Tokens are automatically retrieved by the scaler from [Keystone](https://docs.openstack.org/keystone/latest/), the official OpenStack Identity Provider. You can provide your credentials using Secrets either by using the "password" method or the "application credentials" method. Both cases use `TriggerAuthentication`.

#### Password

- `authURL` - The Keystone authentication URL. The pattern is: `http://<host>:<keystone_port>/<keystone_version>/`.
- `userID` - The OpenStack project user ID.
- `password` - The password for the provided user.
- `projectID` - The OpenStack project ID.

#### Application Credentials

- `authURL` - The Keystone authentication URL. The pattern is: `http://<host>:<keystone_port>/<keystone_version>/`.
- `appCredentialID` - The Application Credential ID.
- `appCredentialSecret` - The Application Credential secret.

### Example

#### Password method

Here is an example of how to deploy a scaled object with the `openstack-swift` scale trigger which uses `TriggerAuthentication` and the Password method from above.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openstack-secret-password
  namespace: default
type: Opaque
data:
  authURL: aHR0cDovL2xvY2FsaG9zdDo1MDAwL3YzLw==
  userID: MWYwYzI3ODFiNDExNGQxM2E0NGI4ODk4Zjg1MzQwYmU=
  password: YWRtaW4=
  projectID: YjE2MWRjNTE4Y2QyNGJkYTg0ZDk0ZDlhMGU3M2ZjODc=
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: swift-password-trigger-authentication
  namespace: default
spec:
  secretTargetRef:
  - parameter: authURL
    name: openstack-secret-password
    key: authURL
  - parameter: userID
    name: openstack-secret-password
    key: userID
  - parameter: password
    name: openstack-secret-password
    key: password
  - parameter: projectID
    name: openstack-secret-password
    key: projectID
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: swift-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  pollingInterval: 10
  cooldownPeriod: 10
  minReplicaCount: 0
  triggers:
  - type: openstack-swift
    metadata:
      swiftURL: http://localhost:8080/v1/AUTH_b161dc518cd24bda84d94d9a0e73fc87
      containerName: my-container
      objectCount: "1"
      onlyFiles: "true"
    authenticationRef:
        name: swift-password-trigger-authentication
```

#### Application Credentials method

You can also use the Application Credentials method. 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openstack-secret-appcredentials
  namespace: default
type: Opaque
data:
  authURL: aHR0cDovL2xvY2FsaG9zdDo1MDAwL3YzLw==
  appCredentialID: OWYyY2UyYWRlYmFkNGQxNzg0NTgwZjE5ZTljMTExZTQ=
  appCredentialSecret: LVdSbFJBZW9sMm91Z3VmZzNEVlBqcll6aU9za1pkZ3c4Y180XzRFU1pZREloT0RmajJkOHg0dU5yb3NudVIzWmxDVTZNLTVDT3R5NDFJX3M5R1N5Wnc=
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: swift-appcredentials-trigger-authentication
  namespace: default
spec:
  secretTargetRef:
  - parameter: authURL
    name: openstack-secret-appcredentials
    key: authURL
  - parameter: appCredentialID
    name: openstack-secret-appcredentials
    key: appCredentialID
  - parameter: appCredentialSecret
    name: openstack-secret-appcredentials
    key: appCredentialSecret
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: swift-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  pollingInterval: 10
  cooldownPeriod: 10
  minReplicaCount: 0
  triggers:
  - type: openstack-swift
    metadata:
      swiftURL: http://localhost:8080/v1/AUTH_b161dc518cd24bda84d94d9a0e73fc87
      containerName: my-container
      objectCount: "1"
    authenticationRef:
        name: swift-appcredentials-trigger-authentication
```