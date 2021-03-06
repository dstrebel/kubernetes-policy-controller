# kubernetes-policy-controller

Every organization has some rules. Some of these are essential to meet governance, and legal requirements and other are based on learning from past experience and not repeating the same mistakes. These decisions cannot tolerate human response time as they need near a real-time action. Services that are policy enabled to make the organization agile and are essential for long-term success as they are more adaptable as violations and conflicts can be discovered consistently as they are not prone to human error. 

Kubernetes allows decoupling complex logic such as policy decisions from the inner working of the API Server by means of "admission controllers”. Admission control is custom logic executed by a webhook. `kubernetes-policy-controller` is a mutating and a validating webhook that gets called for matching Kubernetes API server requests by the admission controller. It uses Open Policy Agent ([OPA](https://github.com/open-policy-agent/opa)), a policy engine for Cloud Native environments hosted by CNCF as a sandbox-level project.

Kubernetes compliance is enforced at the “runtime” via tools such as network policy and pod security policy. [kubernetes-policy-controller](https://github.com/Azure/kubernetes-policy-controller) extends the compliance enforcement at “create” event not at “run“ event. For example, a kubernetes service could answer questions like :

* Can we whitelist / blacklist registries.
* Not allow conflicting hosts for ingresses.
* Label objects based on a user from a department.

In addition to the `admission` scenario  it helps answer the `audit` question such as:

* What are the policies that my cluster is violating.

## Status

This is a new project and is in alpha state.

## Slack Channel

To participate and contribute in defining and creating kubernetes policies.
Channel Name: `kubernetes-policy`
[slack channel] (https://openpolicyagent.slack.com/messages/CDTN970AX)
[Sign up] (https://slack.openpolicyagent.org/)

## Using kubernetes-policy-controller

## 1. Deployment

Access to a Kubernetes cluster with "cluster-admin" permission is the only prerequisite.

Deploy `kubernetes-policy-controller`:

```bash
./deploy/deploy-all.sh
```

Deploy sample policies:

```bash
./deploy/deploy-admission-policy.sh
```

## scenarios

There are two scenarios of the policy engine namely Validation and Mutation

* Validation: "all resources R in namespace N are taged with annotation A"
* Mutation: "before a resource R in namespace N is created tag it with tag T"  

## 1. `validation` scenario

Load the policy as a ConfigMap:

```bash
kubectl create configmap example --from-file ./policy/admission/ingress-host-fqdn.rego
```

```bash
kubectl create ns qa
```

The following call should fail with policy:

```bash
kubectl -n qa apply -f ~/opa/ingress-bad.yaml
```

## 2. `mutation` scenario

This policy will mutate resources that define an annotation with the key `"test-mutation"`. The resouces will be updated to include the annotation `"foo": "bar"`.

Load the policy as a ConfigMap:

```bash
kubectl create configmap example --from-file ./policy/admission/annotate.rego
```

First create a Deployment:

```bash
kubectl run nginx --image nginx
```

Check that the Deployment was not mutated:

```bash
kubectl get deployment nginx -o json | jq '.metadata'
```

Annotate the Deployment to indicate that it should be mutated:

```bash
kubectl annotate deployment nginx test-mutation=true
```

Check that the Deployment was mutated:

```bash
kubectl get deployment nginx -o json | jq '.metadata'
```

## create-policy

### policy language

The `kubernetes-policy-controller` uses OPA as the policy engine. OPA provides a high-level declarative language for authoring policies and simple APIs to answer policy queries.
Policy rules are created as a rego files. 

### package admission

`kubernetes-policy-controller` defines a special package name `admission` which is used to logically execute all the rules.
So any rule defined should be part of this package.

```go
package admission
```

### deny rule

Each violation of a policy is a `deny` rule. So all we need to capture is all `deny` matches in order to validate.  
In the `policy` package any validation rule should be defined as special name called `deny`. In order to understand the basic idea lets consider a case where we want to create a rule which will block all API server requests i.e fail validation of all requests. The following models a that will always `deny` event

```go
package admission

deny[{
    "type": "always",
    "resource": {"kind": kind, "namespace": namespace, "name": name},
    "resolution": {"message": "test always violate"},
}] {
    true
}
```

### matches[[kind, namespace, name, matched_resource_output]]  

When defining a deny rule, you must find Kubernetes resources that match specific criteria, such as Ingress resources in a particular namespace. `kubernetes-policy-controller` provides the matches functionality by importing `data.kubernetes.matches`.

```go
import data.kubernetes.matches
```

Here are some exampples of how matching can be used:

* Find matching Ingress resources  

```go
import data.kubernetes.matches

matches[["ingress", namespace, name, matched_ingress]]
```

* Find matching "ingress" resources in "production" namespace

```go
import data.kubernetes.matches

matches[["ingress", "production", name, matched_ingress]]
```

* Find matching "ingress" resources in "production" namespace with name "my-ingress"

```go
import data.kubernetes.matches

matches[["ingress", "production", "my-ingress", matched_ingress]]
```

Here is an example of a policy which validates that Ingress hostnames must be unique across Namespaces. This policy shows how you can express a pair-wise search. In this case, there is a violation if any two ingresses in different namespaces have the same hostname. Note, you can query OPA to determine whether a single Ingress violates the policy (in which case the cost is linear with the # of Ingresses) or you can query for the set of all Ingresses that violate the policy (in which case the cost is (# of Ingresses)^2.).
Author : [Torrin Sandall](https://github.com/tsandall)

```go
package admission

import data.kubernetes.matches

deny[{
    "id": "ingress-conflict",
    "resource": {"kind": "ingresses", "namespace": namespace, "name": name},
    "resolution": {"message": "ingress host conflicts with an existing ingress"},
}] {
    matches[["ingresses", namespace, name, matched_ingress]]
    matches[["ingresses", other_ns, other_name, other_ingress]]
    namespace != other_ns
    other_ingress.spec.rules[_].host == matched_ingress.spec.rules[_].host
}

```

## patches resolution

Patches field allows mutation of objects.

### Example patch

```js
package admission

import data.k8s.matches

##############################################################################
#
# Policy : Construct JSON Patch for annotating boject with foo=bar if it is
# annotated with "test-mutation"
#
##############################################################################

deny[{
    "id": "conditional-annotation",
    "resource": {"kind": kind, "namespace": namespace, "name": name},
    "resolution": {"patches":  p, "message" : "conditional annotation"},
}] {
    matches[[kind, namespace, name, matched_object]]
    matched_object.metadata.annotations["test-mutation"]
    p = [{"op": "add", "path": "/metadata/annotations/foo", "value": "bar"}]
}
```

### Video

[Demo video of Kubernetes Policy Controller](https://youtu.be/1WObJiTZDHc)

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
