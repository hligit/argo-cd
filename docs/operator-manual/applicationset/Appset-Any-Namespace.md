# ApplicationSet in any namespace

**Current feature state**: Beta

!!! warning
    Please read this documentation carefully before you enable this feature. Misconfiguration could lead to potential security issues.

## Introduction

As of version 2.8, Argo CD supports managing `ApplicationSet` resources in namespaces other than the control plane's namespace (which is usually `argocd`), but this feature has to be explicitly enabled and configured appropriately.

Argo CD administrators can define a certain set of namespaces where `ApplicationSet` resources may be created, updated and reconciled in. 

As Applications generated by an ApplicationSet are generated in the same namespace as the ApplicationSet itself, this works in combination with [App in any namespace](../app-any-namespace.md).
    
## Prerequisites

### App in any namespace configured

This feature needs [App in any namespace](../app-any-namespace.md) feature activated. The list of namespaces must be the same.

### Cluster-scoped Argo CD installation

This feature can only be enabled and used when your Argo CD ApplicationSet controller is installed as a cluster-wide instance, so it has permissions to list and manipulate resources on a cluster scope. It will *not* work with an Argo CD installed in namespace-scoped mode.

## Implementation details

### Overview

In order for an ApplicationSet to be managed and reconciled outside the Argo CD's control plane namespace, two prerequisites must match:

1. The namespace list from which `argocd-applicationset-controller` can source `ApplicationSets` must be explicitly set using environment variable `ARGOCD_APPLICATIONSET_CONTROLLER_NAMESPACES` or alternatively using parameter `--applicationset-namespaces`.
2. The enabled namespaces must be entirely covered by the [App in any namespace](../app-any-namespace.md), otherwise the generated Applications generated outside the allowed Application namespaces won't be reconciled

It can be achieved by setting the environment variable `ARGOCD_APPLICATIONSET_CONTROLLER_NAMESPACES` to argocd-cmd-params-cm `applicationsetcontroller.namespaces`

`ApplicationSets` in different namespaces can be created and managed just like any other `ApplicationSet` in the `argocd` namespace previously, either declaratively or through the Argo CD API (e.g. using the CLI, the web UI, the REST API, etc).

### Reconfigure Argo CD to allow certain namespaces

#### Change workload startup parameters

In order to enable this feature, the Argo CD administrator must reconfigure the and `argocd-applicationset-controller` workloads to add the `--applicationset-namespaces` parameter to the container's startup command.

### Safely template project

As [App in any namespace](../app-any-namespace.md) is a prerequisite, it is possible to safely template project. 

Let's take an example with two teams and an infra project:

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: infra-project
  namespace: argocd
spec:
  destinations:
    - namespace: '*'  
```

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: team-one-project
  namespace: argocd
spec:
  sourceNamespaces:
  - team-one-cd
```

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: team-two-project
  namespace: argocd
spec:
  sourceNamespaces:
  - team-two-cd
```

Creating following `ApplicationSet` generates two Applications `infra-escalation` and `team-two-escalation`. Both will be rejected as they are outside `argocd` namespace, therefore `sourceNamespaces` will be checked

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-one-product-one
  namespace: team-one-cd
spec:
  generators:
    list:
    - id: infra
      project: infra-project
    - id: team-two
      project: team-two-project
    template:
      metadata:
        name: '{{name}}-escalation'
      spec:
        project: "{{project}}"
```
  
### ApplicationSet names

For the CLI, applicationSets are now referred to and displayed as in the format `<namespace>/<name>`. 

For backwards compatibility, if the namespace of the ApplicationSet is the control plane's namespace (i.e. `argocd`), the `<namespace>` can be omitted from the applicationset name when referring to it. For example, the application names `argocd/someappset` and `someappset` are semantically the same and refer to the same application in the CLI and the UI.

### Applicationsets RBAC

The RBAC syntax for Application objects has been changed from `<project>/<applicationset>` to `<project>/<namespace>/<applicationset>` to accomodate the need to restrict access based on the source namespace of the Application to be managed.

For backwards compatibility, Applications in the argocd namespace can still be refered to as `<project>/<applicationset>` in the RBAC policy rules.

Wildcards do not make any distinction between project and applicationset namespaces yet. For example, the following RBAC rule would match any application belonging to project foo, regardless of the namespace it is created in:


```
p, somerole, applicationsets, get, foo/*, allow
```

If you want to restrict access to be granted only to `ApplicationSets` with project `foo` within namespace `bar`, the rule would need to be adapted as follows:

```
p, somerole, applicationsets, get, foo/bar/*, allow
```

## Managing applicationSets in other namespaces

### Using the CLI

You can use all existing Argo CD CLI commands for managing applications in other namespaces, exactly as you would use the CLI to manage applications in the control plane's namespace.

For example, to retrieve the `ApplicationSet` named `foo` in the namespace `bar`, you can use the following CLI command:

```shell
argocd appset get foo/bar
```

Likewise, to manage this applicationSet, keep referring to it as `foo/bar`:

```bash
# Delete the application
argocd appset delete foo/bar
```

There is no change on the create command as it is using a file. You just need to add the namespace in the `metadata.namespace` field.

As stated previously, for applicationSets in the Argo CD's control plane namespace, you can omit the namespace from the application name.

### Using the REST API

If you are using the REST API, the namespace for `ApplicationSet` cannot be specified as the application name, and resources need to be specified using the optional `appNamespace` query parameter. For example, to work with the `ApplicationSet` resource named `foo` in the namespace `bar`, the request would look like follows:

```bash
GET /api/v1/applicationsets/foo?appsetNamespace=bar
```

For other operations such as `POST` and `PUT`, the `appNamespace` parameter must be part of the request's payload.

For `ApplicationSet` resources in the control plane namespace, this parameter can be omitted.

## Secrets consideration

By allowing ApplicationSet in any namespace you must be aware that clusters, API token secrets (etc...) can be discovered and used. 

Example:

Following will discover all clusters

```yaml
spec:
  generators:
  - clusters: {} # Automatically use all clusters defined within Argo CD
```

If you don't want to allow users to discover secrets with ApplicationSets from other namespaces you may consider deploying ArgoCD in namespace scope or use OPA rules.