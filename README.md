# Istio + Gatekeeper

Resources and walkthrough for the [Istio + Gateekper](https://events.istio.io/istiocon-2022/sessions/gatekeeper-istio/) talk at IstioCon 2022.

- [Setup](#setup)
  - [Create Kubernetes cluster](#create-kubernetes-cluster)
  - [Install Istio](#install-istio)
  - [Install Gatekeeper](#install-gatekeeper)
  - [Deploy Ingress Gateway](#deploy-ingress-gateway)
  - [Deploy sample apps](#deploy-sample-apps)
- [Demos](#demos)
  - [Enforce Istio sidecar injection](#enforce-istio-sidecar-injection)
  - [Enforce `STRICT` mTLS in the Mesh](#enforce-strict-mtls-in-the-mesh)
  - [Enforce `AuthorizationPolicies`](#enforce-authorizationpolicies)
  - [Shift enforcement left](#shifting-left-the-detection-of-the-constraints-violations)

## Setup

As prerequisites, you need to have these tools installed in order to run this demo:
- [`istioctl`](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl)
- [`kpt`](https://kpt.dev/installation/)
- [`helm`](https://helm.sh/docs/intro/install/)
- [`kubectl`](https://open-policy-agent.github.io/gatekeeper/website/docs/gator/)
- [`gator`](https://open-policy-agent.github.io/gatekeeper/website/docs/gator/)

### Create Kubernetes cluster

Create a Kubernetes clusters, for example in GCP you could run this:
```bash
CLUSTER_NAME=istio-gatekeeper-demo
CLUSTER_ZONE=us-east4-a
gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_ZONE} \
    --machine-type=e2-standard-2
```

## Install Istio

Install Istio in your cluster:
```bash
istioctl install --set profile=minimal -y
```

## Install Gatekeeper

Install Gatekeeper in your cluster:
```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace --set auditInterval=5
```

## Deploy Ingress Gateway

Deploy an Istio Ingress Gateway in a dedicated namespace with the `istio-ingress istio-injection=enabled` label:
```bash
kubectl apply -f istio-ingressgateway/namespace.yaml
kubectl apply -f istio-ingressgateway/base/
kubectl apply -f istio-ingressgateway/gateway.yaml
```

## Deploy sample apps

Deploy the Online Boutique sample apps in a dedicated namespace with the `istio-ingress istio-injection=enabled` label:
```bash
kubectl apply -f onlineboutique/namespace.yaml
kubectl apply -f onlineboutique/apps-manifests.yaml
kubectl apply -f onlineboutique/frontend-virtualservice.yaml
```

Wait for the public IP address to be provisioned:
```bash
until kubectl get svc istio-ingressgateway -n istio-ingress -o jsonpath='{.status.loadBalancer}' | grep "ingress"; do : ; done
```

Open the generated public IP address to browse the Online Boutique website:
```bash
echo -n "http://" && \
kubectl get svc istio-ingressgateway -n istio-ingress -o json | jq -r '.status.loadBalancer.ingress[0].ip'
```

Check the number of proxies (13) running in your mesh:
```bash
istioctl version
```

Output similar to:
```output
client version: 1.13.3
control plane version: 1.13.3
data plane version: 1.13.3 (13 proxies)
```

## Demos

_In case you run multiple times this section of demos in the same cluster, here is the cleanup routine you can run to have a clean setup:_
```bash
kubectl delete constraints --all
kubectl delete constrainttemplates --all
kubectl delete peerauthentication strict-mtls -n istio-system
kubectl delete authorizationpolicy deny-all -n istio-system
kubectl delete authorizationpolicy istio-ingressgateway -n istio-ingress
kubectl delete authorizationpolicy --all -n onlineboutique
```

### Enforce Istio sidecar injection

- `K8sRequiredLabels` - requires any `Namespace` in the mesh to contain the specific Istio sidecar proxy injection label: `istio-injection` with the value `enabled`
- `SidecarInjectionAnnotation` - prohibits any `Pod` in the mesh to by-pass the Istio proxy sidecar injection

Let's deploy these two `constraints` and `constrainttemplates`:
```bash
kubectl apply -f constrainttemplates/sidecar-injection
kubectl apply -f constraints/sidecar-injection
```

Verify that the two `constrainttemplates` has been deployed successfully:
```bash
kubectl get constrainttemplates
```

Output similar to:
```output
NAME                         AGE
k8srequiredlabels            47s
sidecarinjectionannotation   47s
```

Verify that the two `constraints` has been deployed successfully:
```bash
kubectl get constraints
```

Output similar to:
```output
NAME                                                                                ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
sidecarinjectionannotation.constraints.gatekeeper.sh/sidecar-injection-annotation   deny                 0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
```

Try to create a `test` namespace without the required `label`:
```bash
kubectl create namespace test
```

Output similar to:
```output
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: [namespace-sidecar-injection-label] you must provide labels: {"istio-injection"}
```

### Enforce `STRICT` mTLS in the Mesh

- `PeerAuthnMeshStrictMtls` - requires a default `STRICT` mTLS `PeerAuthentication` for the entire mesh in the `istio-system` namespace
- `PeerAuthnStrictMtls` - prohibits disabling `STRICT` mTLS for all `PeerAuthentications`
- `DestinationRuleTLSEnabled` - prohibits disabling `STRICT` mTLS for all hosts and host subsets in `DestinationRules`

Let's extend the default Gatekeeper config in order to take into account Istio resources:
```bash
kubectl -n gatekeeper-system apply -f gatekeeper-system/referential-constraints-config.yaml
```

Let's deploy these two `constraints` and `constrainttemplates`:
```bash
kubectl apply -f constrainttemplates/strict-mtls
kubectl apply -f constraints/strict-mtls
```

Verify that the two `constrainttemplates` has been deployed successfully:
```bash
kubectl get constrainttemplates
```

Output similar to:
```output
NAME                         AGE
destinationruletlsenabled    18s
k8srequiredlabels            56m
peerauthnmeshstrictmtls      17s
peerauthnstrictmtls          17s
sidecarinjectionannotation   56m
```

Verify that the two `constraints` has been deployed successfully:
```bash
kubectl get constraints
```

Output similar to:
```output
NAME                                                                               ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
destinationruletlsenabled.constraints.gatekeeper.sh/destination-rule-tls-enabled   deny                 0

NAME                                                                       ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls   dryrun               1

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnstrictmtls.constraints.gatekeeper.sh/peer-authentication-strict-mtls   deny                 0

NAME                                                                                ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
sidecarinjectionannotation.constraints.gatekeeper.sh/sidecar-injection-annotation   deny                 0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
```

We could look at the violation detected for the  `PeerAuthnMeshStrictMtls` `Constraint` to get more details:
```bash
kubectl get peerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls -ojsonpath='{.status.violations}' | jq
```

The output is similar to:
```output
[
  {
    "enforcementAction": "dryrun",
    "kind": "Namespace",
    "message": "Root namespace <istio-system> does not have a strict mTLS PeerAuthentication",
    "name": "istio-system"
  }
]
```

We could fix this violation by deploying the default `STRICT` mTLS `PeerAuthentication` in the `istio-system` namespace:
```bash
kubectl apply -f istio-system/default-strict-peerauthentication.yaml
```

After a few minutes, verify that the `Constraints` don't have any remaining violations:
```bash
kubectl get constraints
```

The output is similar to:
```output
NAME                                                                               ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
destinationruletlsenabled.constraints.gatekeeper.sh/destination-rule-tls-enabled   deny                 0

NAME                                                                       ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls   dryrun               0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnstrictmtls.constraints.gatekeeper.sh/peer-authentication-strict-mtls   deny                 0

NAME                                                                                ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
sidecarinjectionannotation.constraints.gatekeeper.sh/sidecar-injection-annotation   deny                 0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
```

### Enforce `AuthorizationPolicies`

- `AuthzPolicyDefaultDeny` - requires a default `deny` `AuthorizationPolicy` for the entire mesh in the `istio-system` namespace

Let's deploy these two `constraints` and `constrainttemplates`:
```bash
kubectl apply -f constrainttemplates/authorization-policies
kubectl apply -f constraints/authorization-policies
```

Verify that the two `constrainttemplates` has been deployed successfully:
```bash
kubectl get constrainttemplates
```

Output similar to:
```output
NAME                         AGE
authzpolicydefaultdeny       10s
destinationruletlsenabled    13h
k8srequiredlabels            13h
peerauthnmeshstrictmtls      13h
peerauthnstrictmtls          13h
sidecarinjectionannotation   13h
```

Verify that the two `constraints` has been deployed successfully:
```bash
kubectl get constraints
```

Output similar to:
```output
NAME                                                                                ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
sidecarinjectionannotation.constraints.gatekeeper.sh/sidecar-injection-annotation   deny                 0

NAME                                                                               ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
destinationruletlsenabled.constraints.gatekeeper.sh/destination-rule-tls-enabled   deny                 0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnstrictmtls.constraints.gatekeeper.sh/peer-authentication-strict-mtls   deny                 0

NAME                                                                                   ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
authzpolicydefaultdeny.constraints.gatekeeper.sh/default-deny-authorization-policies   dryrun               1

NAME                                                                       ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls   dryrun               0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
```

We could look at the violation detected for the `AuthzPolicyDefaultDeny` `Constraint` to get more details:
```bash
kubectl get authzpolicydefaultdeny.constraints.gatekeeper.sh/default-deny-authorization-policies -ojsonpath='{.status.violations}' | jq
```

The output is similar to:
```output
[
  {
    "enforcementAction": "dryrun",
    "kind": "Namespace",
    "message": "Root namespace <istio-system> does not have a default deny AuthorizationPolicy",
    "name": "istio-system"
  }
]
```

We could fix this violation by deploying the default deny `AuthorizationPolicy` in the `istio-system` namespace:
```bash
kubectl apply -f istio-system/default-deny-authorizationpolicy.yaml
```

After a few minutes, verify that the `Constraints` don't have any remaining violations:
```bash
kubectl get constraints
```

The output is similar to:
```output
NAME                                                                                ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
sidecarinjectionannotation.constraints.gatekeeper.sh/sidecar-injection-annotation   deny                 0

NAME                                                                               ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
destinationruletlsenabled.constraints.gatekeeper.sh/destination-rule-tls-enabled   deny                 0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnstrictmtls.constraints.gatekeeper.sh/peer-authentication-strict-mtls   deny                 0

NAME                                                                                   ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
authzpolicydefaultdeny.constraints.gatekeeper.sh/default-deny-authorization-policies   dryrun               0

NAME                                                                       ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls   dryrun               0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
```

Visit the Online Boutique website from your browser, you should receive the error: `RBAC: access denied` which confirms that the default deny `AuthorizationPolicy` applies to the entire mesh.

Fix this issue by deploying more granular `AuthorizationPolicy` resources in both the Ingress Gateway and the Online Boutique namespaces:
```bash
kubectl apply -f istio-ingressgateway/authorizationpolicy.yaml
kubectl apply -f onlineboutique/authorizationpolicies.yaml 
```

Visit again the Online Boutique website from your browser, you should now see it working successfully now.


Congrats! You have secured your cluster, your mesh and your Online Boutique website with `STRICT` mTLS and fine granular `AuthorizationPolicies`, while enforcing this secure setup with associated policies and `Constraints`!

### Shift enforcement left

But now, what if you want to detect `Constraints` violations earlier in the process, without waiting for an actual deployment into your Kubernetes cluster?

Let's see how we could shift left this detection, even from your local machine! You have 2 options, you could either with `gator` or `kpt`.

Let's use a dedicated `pr-with-errors` branch with some errors generated for the purpose of this section:
```bash
git checkout pr-with-errors
```

With `kpt`:
```bash
kpt fn eval . --image gcr.io/kpt-fn/gatekeeper:v0.2
```

Output similar to:
```output
[RUNNING] "gcr.io/kpt-fn/gatekeeper:v0.2"
[FAIL] "gcr.io/kpt-fn/gatekeeper:v0.2" in 1.8s
  Results:
    [error] v1/Service/onlineboutique/emailservice: the service <emailservice> port name <bad-port-name> has a disallowed prefix, allowed prefixes are ["http", "grpc", "tcp"] violatedConstraint: port-name-constraint
    [error] v1/Namespace/onlineboutique: you must provide labels: {"istio-injection"} violatedConstraint: namespace-sidecar-injection-label
  Stderr:
    "[error] v1/Service/onlineboutique/emailservice : the service <emailservice> port name <bad-port-name> has a disallowed prefix, allowed prefixes are [\"http\", \"grpc\", \"tcp\"]"
    "violatedConstraint: port-name-constraint"
    ""
    "[error] v1/Namespace/onlineboutique : you must provide labels: {\"istio-injection\"}"
    ...(1 line(s) truncated, use '--truncate-output=false' to disable)
  Exit code: 1
```

With `gator`:
```bash
rm .github/workflows/* 
rm -rf test/
gator test -f .
```

Output similar to:
```output
Message: "the service <emailservice> port name <bad-port-name> has a disallowed prefix, allowed prefixes are [\"http\", \"grpc\", \"tcp\"]"Message: "you must provide labels: {\"istio-injection\"}"
```

You could even do this in your own CI/CD pipelines like Jenkins, Azure Devops, Cloud Build, GitHub actions, etc.

You can see the 2 options illustrated in this [`.github/workflows/ci.yml`](.github/workflows/ci.yml) file. You can [see this in action in this PR opened on this `pr-with-errors` branch](https://github.com/mathieu-benoit/istio-gatekeeper-demos/actions/runs/3119574752).
