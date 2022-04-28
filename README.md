# Resources and walkthrough for the Istio+Gateekper, FTW talk at IstioCon 2022

TOC:
- [Setup](#setup)
  - [Create Kubernetes cluster](#create-kubernetes-cluster)
  - [Install Istio](#install-istio)
  - [Install Gatekeeper](#install-gatekeeper)
- [Demos](#demos)

## Setup

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

[Install Istio]():
```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.3 sh -
cd istio-$ISTIO_VERSION
export PATH=$PWD/bin:$PATH
istioctl install --set profile=minimal -y
```

## Install Gatekeeper

[Install Gatekeeper]():
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.8/deploy/gatekeeper.yaml
```

## Deploy Ingress Gateway

Deploy an Istio Ingress Gateway:
```bash
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
kubectl apply -f istio-ingressgateway/base/
kubectl apply -f istio-ingressgateway/gateway.yaml
```

## Deploy sample apps

Deploy the [Online Boutique sample]() apps:
```bash
kubectl create namespace onlineboutique
kubectl label namespace onlineboutique istio-injection=enabled
kubectl apply -f onlineboutique/base/
kubectl apply -f onlineboutique/frontend-virtualservice.yaml
```

Open the generated public IP address to browse the Online Boutique website:
```bash
echo -n "http://" && \
kubectl get svc istio-ingressgateway -n istio-ingress -o json | jq -r '.status.loadBalancer.ingress[0].ip'
```
_Note: you may need to re-run this command above couple of times in order to get the public IP address value different from `null`._

## Demos

### Enforce Istio sidecar injection

- `K8sRequiredLabels`
- `SidecarInjectionAnnotation`

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

- `PeerAuthnMeshStrictMtls`
- `PeerAuthnStrictMtls`
- `DestinationRuleTLSEnabled`

Let's extend the default Gatekeeper config in order to take into account Istio resources:
```bash
kubectl -n gatekeeper-system apply -f ${WORKDIR}/gatekeeper-system/config-referential-constraints.yaml
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

We could see after a few minutes that Gatekeeper will raise violations for the `AsmPeerAuthnMeshStrictMtls` `Constraint`:
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
kubectl -n istio-system apply -f istio-system/default-strict-peerauthentication.yaml
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

- `AuthzPolicyDefaultDeny`

FIXME

### Shifting left the detection of the `Constraints` violations

Install `kpt`:
```bash
curl -L https://github.com/GoogleContainerTools/kpt/releases/download/v1.0.0-beta.14/kpt_linux_amd64 > ./kpt
chmod +x kpt
```

Let's evaluate the `Constraints` locally against the Kubernetes manifests we have in the current folder:
```bash
./kpt fn eval . --image gcr.io/kpt-fn/gatekeeper:v0.2
```

Output similar to:
```output
[RUNNING] "gcr.io/kpt-fn/gatekeeper:v0.2"
[FAIL] "gcr.io/kpt-fn/gatekeeper:v0.2" in 2.2s
  Results:
    [error] v1/Service/onlineboutique/redis-cart: the service port name <redis> has a disallowed prefix, allowed prefixes are ["http", "grpc", "tcp"] violatedConstraint: port-name-constraint
  Stderr:
    "[error] v1/Service/onlineboutique/redis-cart : the service port name <redis> has a disallowed prefix, allowed prefixes are [\"http\", \"grpc\", \"tcp\"]"
    "violatedConstraint: port-name-constraint"
  Exit code: 1
```

We could even show this in action from within your own CI/CD pipelines like Jenkins, Azure Devops, Cloud Build, etc. Let's do the demo with GitHub actions here!

This command above is included in this [`.github/workflows/ci.yaml`] file. Let's see this in actions via the GitHub UI.