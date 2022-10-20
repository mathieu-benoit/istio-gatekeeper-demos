# Shift enforcement left

What if you want to detect `Constraints` violations earlier in the process, without waiting for an actual deployment into your Kubernetes cluster?

Let's see how we could shift left this detection, even from your local machine! You have 2 options to evaluate your Gatekeeper policies against your Kubernetes manifets, you could use either `gator` or `kpt`.

Let's use a dedicated `new-test-app` branch with some errors generated for the purpose of this demo:
```bash
git checkout new-test-app
```

As prerequisites, you need to have these tools installed:
- [`kpt`](https://kpt.dev/installation/)
- [`gator`](https://open-policy-agent.github.io/gatekeeper/website/docs/gator/#installation)

## With `kpt`
```bash
kpt fn eval . \
    --image gcr.io/kpt-fn/gatekeeper:v0.2 \
    --truncate-output=false
```

Output similar to:
```output
[RUNNING] "gcr.io/kpt-fn/gatekeeper:v0.2"
[FAIL] "gcr.io/kpt-fn/gatekeeper:v0.2" in 900ms
  Results:
    [error] v1/Service/best-app-ever/best-app-ever: the service <best-app-ever> port name <test> has a disallowed prefix, allowed prefixes are ["http", "grpc", "tcp"] violatedConstraint: port-name-constraint
    [error] apps/v1/Deployment/best-app-ever/best-app-ever: The annotation sidecar.istio.io/inject: false should not be applied on workload pods violatedConstraint: sidecar-injection-annotation
    [error] security.istio.io/v1beta1/PeerAuthentication/best-app-ever/best-app-ever: PeerAuthentication mtls mode can only be set to UNSET or STRICT violatedConstraint: peer-authentication-strict-mtls
  Stderr:
    "[error] v1/Service/best-app-ever/best-app-ever : the service <best-app-ever> port name <test> has a disallowed prefix, allowed prefixes are [\"http\", \"grpc\", \"tcp\"]"
    "violatedConstraint: port-name-constraint"
    ""
    "[error] apps/v1/Deployment/best-app-ever/best-app-ever : The annotation sidecar.istio.io/inject: false should not be applied on workload pods"
    "violatedConstraint: sidecar-injection-annotation"
    ""
    "[error] security.istio.io/v1beta1/PeerAuthentication/best-app-ever/best-app-ever : PeerAuthentication mtls mode can only be set to UNSET or STRICT"
    "violatedConstraint: peer-authentication-strict-mtls"
  Exit code: 1
```

## With `gator`
```bash
rm .github/workflows/* 
rm -rf test/
gator test \
    -f .
```

Output similar to:
```output
Message: "PeerAuthentication mtls mode can only be set to UNSET or STRICT"Message: "The annotation sidecar.istio.io/inject: false should not be applied on workload pods"Message: "the service <best-app-ever> port name <test> has a disallowed prefix, allowed prefixes are [\"http\", \"grpc\", \"tcp\"]"Message: "you must provide labels: {\"istio-injection\"}"
```

## In CI pipelines

You could even do this in your own CI pipelines like Jenkins, Azure Devops, Cloud Build, GitHub actions, etc. With GitHub actions that's illustrated in this repo, you could look at the definition of these GitHub actions workflows:
- [`.github/workflows/ci-apps-gator.yml`](.github/workflows/ci-apps-gator.yml)
- [`.github/workflows/ci-apps-kpt.yml`](.github/workflows/ci-apps-kpt.yml)

You can [see this in action in this PR opened on this `ob-with-errors` branch](https://github.com/mathieu-benoit/istio-gatekeeper-demos/pull/15).
