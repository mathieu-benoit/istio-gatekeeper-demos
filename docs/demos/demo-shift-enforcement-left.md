# Shift enforcement left

What if you want to detect `Constraints` violations earlier in the process, without waiting for an actual deployment into your Kubernetes cluster?

Let's see how we could shift left this detection, even from your local machine! You have 2 options, you could use either `gator` or `kpt`.

Let's use a dedicated `ob-with-errors` branch with some errors generated for the purpose of this demo:
```bash
git checkout ob-with-errors
```

## With `kpt`
```bash
kpt fn eval . \
    --image gcr.io/kpt-fn/gatekeeper:v0.2
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

## With `gator`
```bash
rm .github/workflows/* 
rm -rf test/
gator test \
    -f .
```

Output similar to:
```output
Message: "the service <emailservice> port name <bad-port-name> has a disallowed prefix, allowed prefixes are [\"http\", \"grpc\", \"tcp\"]"Message: "you must provide labels: {\"istio-injection\"}"
```

## In CI pipelines

You could even do this in your own CI pipelines like Jenkins, Azure Devops, Cloud Build, GitHub actions, etc. With GitHub actions that's illustrated in this repo:
- [`.github/workflows/ci-apps-gator.yml`](.github/workflows/ci-apps-gator.yml)
- [`.github/workflows/ci-apps-kpt.yml`](.github/workflows/ci-apps-kpt.yml)

You can [see this in action in this PR opened on this `ob-with-errors` branch](https://github.com/mathieu-benoit/istio-gatekeeper-demos/pull/11).
