# Instrumenting Go applications with EDOT SDKs on Kubernetes

We don't have an EDOT Go SDK.

This document focuses on instrumenting Go applications on Kubernetes, using [Elastic Distribution of OpenTelemetry Go (EDOT Go)](https://github.com/elastic/elastic-otel-go) together with the OpenTelemetry Operator and the EDOT Collectors described in the [getting started](./README.md) guide.

- For general knowledge about the EDOT Go SDK, refer to the [EDOT Go docs](https://github.com/elastic/elastic-otel-go/blob/main/docs/get-started.md).

- For general information about instrumenting applications on kubernetes, refer to [instrumenting applications](./instrumenting-applications.md).

(TBD - CHECK)
- To manually instrument your Go application code (by customizing transactions, traces, and spans), refer to [EDOT Go manual instrumentation](https://github.com/elastic/elastic-otel-go/blob/main/docs/manual-instrumentation.md#Manually-instrument-your-Go-application).

(TBD, check if we have to add some language specifics)
## Supported environments and configuration

## Guided example to instrument a Go app on Kubernetes

In the following example you will learn how to:

- Deploy an example Go app in a dedicated namespace.
- Enable auto-instrumentation of the application following any of the supported methods, such as:
  - Adding an annotation to the namespace.
  - Adding an annotation to the deployment pods.
- Verify that auto-instrumentation libraries are injected and configured correctly.
- Confirm data is flowing to **Kibana Observability**.

Before continuing, ensure you have performed the [installation of the operator](./README.md), and confirm that the following `Instrumentation` object exists in the system:

```bash
$ kubectl get instrumentation -n opentelemetry-operator-system
NAME                      AGE    ENDPOINT                                                                                                
elastic-instrumentation   107s   http://opentelemetry-kube-stack-daemon-collector.opentelemetry-operator-system.svc.cluster.local:4318
```

Example auto-instrumentation steps:

(TBD - double check the example)
1. Create a `go` namespace and run a deployment named `go-app`

    ```bash
    # Go Namespace and Deployment
    kubectl create namespace go
    kubectl apply -f - <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: go-app
      name: go-app
      namespace: go
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: go-app
      template:
        metadata:
          name: go-app
          namespace: go
          labels:
            app: go-app
          annotations:
            instrumentation.opentelemetry.io/otel-go-auto-target-exe: "/app/server"
        spec:
          shareProcessNamespace: true
          containers:
            - image: rcolles/k8s-webhook-test:0.0.4
              imagePullPolicy: Always
              name: go-test-app
              env:
              - name: OTEL_LOG_LEVEL
                value: "debug"
              securityContext:
                runAsUser: 0
                privileged: true
    EOF
    ```


2. Enable auto-instrumentation of the Go application using one of the following methods:

  - Add an annotation at namespace level (this will make all Pods of the namespace to be instrumented):

    ```bash
    kubectl annotate namespace go instrumentation.opentelemetry.io/inject-go=opentelemetry-operator-system/elastic-instrumentation
    ```

  - Alternatively, edit the `go-app` deployment to include the annotation under `spec.template.metadata.annotations`:

    ```yaml
    spec:
    ...
      template:
        metadata:
          labels:
            app: go-app
          annotations:
            instrumentation.opentelemetry.io/inject-go: opentelemetry-operator-system/elastic-instrumentation
    ...
    ```

3. Restart application:

  Once the annotation has been set, the Pods need to be recreated for the instrumentation libraries to be injected.

    ```bash
    kubectl rollout restart deployment go-app -n go
    ```

4. Verify the auto-instrumentation resources are injected in the Pod:

  Go apps are instrumented by the OpenTelemetry Operator with the following actions:

  - It adds an init container in the Pod with the objective of copying the SDK to a shared volume.

  - Defines an `emptyDir volume` mounted in both containers.

  - Adds `PYTHONPATH` and other OTEL related environment variables.

  Run `kubectl describe pod go-app-xxxx` and check:

  - There should be an init container named `opentelemetry-auto-instrumentation-go` in the Pod:

    ```bash
    $ kubectl describe pod go-app-8d84c47b8-8h5z2 -n go
    ...
    ...
    Init Containers:
      opentelemetry-auto-instrumentation-go:
        Container ID:  containerd://fdc86b3191e34ef5ec872853b14a950d0af1e36b0bc207f3d59bd50dd3caafe9
        Image:         docker.elastic.co/observability/elastic-otel-go:0.3.0
        Image ID:      docker.elastic.co/observability/elastic-otel-go@sha256:de7b5cce7514a10081a00820a05097931190567ec6e18a384ff7c148bad0695e
        Port:          <none>
        Host Port:     <none>
        Command:
          cp
          -r
          /autoinstrumentation/.
          /otel-auto-instrumentation-go
        State:          Terminated
          Reason:       Completed
    ...
    ```

  - The main container has new environment variables: 

    ```bash
    ...
    Containers:
      go-app:
    ...
        Environment:
    ...
          PYTHONPATH:                          /otel-auto-instrumentation-go/opentelemetry/instrumentation/auto_instrumentation:/otel-auto-instrumentation-go
          OTEL_EXPORTER_OTLP_PROTOCOL:         http/protobuf
          OTEL_TRACES_EXPORTER:                otlp
          OTEL_METRICS_EXPORTER:               otlp
          OTEL_SERVICE_NAME:                   go-app
          OTEL_EXPORTER_OTLP_ENDPOINT:         http://opentelemetry-kube-stack-daemon-collector.opentelemetry-operator-system.svc.cluster.local:4318
    ...
    ```

  - The Pod has an `EmptyDir` volume named `opentelemetry-auto-instrumentation-go` mounted in both the main and the init containers in path `/otel-auto-instrumentation-go`:

  Ensure the environment variable `OTEL_EXPORTER_OTLP_ENDPOINT` points to a valid endpoint and there's network communication between the Pod and the endpoint.

5. Confirm data is flowing through in **Kibana**:

  - Open Observability -> Applications -> Service Inventory, and determine if:
    - The application appears in the list of services.
    - The application shows transactions and metrics.
    - If [go logs instrumentation](https://opentelemetry.io/docs/kubernetes/operator/automatic/#auto-instrumenting-go-logs) is enabled, the application logs should  appear in the Logs tab.
  
  - For application container logs, open **Kibana Discovery** and filter for your pods log, with any of:
    - `k8s.deployment.name: "go-app"`
    - `k8s.pod.name: go-app*`

    Note that the container logs are not provided by the instrumentation library, but by the DaemonSet collector deployed as part of the [operator installation](./README.md)

## Troubleshooting

- Refer to [troubleshoot auto-instrumentation](./troubleshoot-auto-instrumentation.md) for further analysis.
