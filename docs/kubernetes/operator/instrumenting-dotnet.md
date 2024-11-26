# Instrumenting .NET applications with EDOT SDKs on Kubernetes

This document focuses on instrumenting .NET applications on Kubernetes, using [Elastic Distribution of OpenTelemetry .NET (EDOT .NET)](https://github.com/elastic/elastic-otel-dotnet) together with the OpenTelemetry Operator and the EDOT Collectors described in the [getting started](./README.md) guide.

- For general knowledge about the EDOT .NET SDK, refer to the [EDOT .NET docs](https://github.com/elastic/elastic-otel-dotnet/blob/main/docs/get-started.md).

- For general information about instrumenting applications on kubernetes, refer to [instrumenting applications](./instrumenting-applications.md).

(TBD...)
- To manually instrument your .NET application code (by customizing transactions, traces, and spans), refer to [EDOT .NET manual instrumentation](https://github.com/elastic/elastic-otel-dotnet/blob/main/docs/manual-instrumentation.md).

(TBD DOTNET SPECIFICS, if any)
## Supported environments and configuration

## Guided example to instrument a .NET app with EDOT .NET SDK on Kubernetes

In the following example you will learn how to:

- Deploy an example .NET app in a dedicated namespace.
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

1. Create a `dotnet` namespace and run a deployment named `dotnet-app`

    ```bash
    # dotnet Namespace and Deployment
    kubectl create namespace dotnet
    kubectl apply -f - <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: dotnet-app
      name: dotnet-app
      namespace: dotnet
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: dotnet-app
      template:
        metadata:
          name: dotnet-app
          namespace: dotnet
          labels:
            app: dotnet-app
        spec:
          containers:
            - image: andrewgizas/dotnet-otel-auto:latest
              imagePullPolicy: Always
              name: dotnet-app
              env: 
              - name: OTEL_LOG_LEVEL
                value: "debug"
              - name: ELASTIC_OTEL_LOG_TARGETS
                value: "stdout"
    EOF    
    ```

2. Enable auto-instrumentation of the .NET application using one of the following methods:

  - Add an annotation at namespace level (this will make all Pods of the namespace to be instrumented):

    ```bash
    kubectl annotate namespace dotnet instrumentation.opentelemetry.io/inject-dotnet=opentelemetry-operator-system/elastic-instrumentation
    ```

  - Alternatively, edit the `dotnet-app` deployment to include the annotation under `spec.template.metadata.annotations`:

    ```yaml
    spec:
    ...
      template:
        metadata:
          labels:
            app: dotnet-app
          annotations:
            instrumentation.opentelemetry.io/inject-dotnet: opentelemetry-operator-system/elastic-instrumentation
    ...
    ```

3. Restart application:

  Once the annotation has been set, the Pods need to be recreated for the instrumentation libraries to be injected.

    ```bash
    kubectl rollout restart deployment dotnet-app -n dotnet
    ```


(TBD - REVIEW ALL CONTENT FROM THIS POINT, AS IT'S LANGUAGE SPECIFIC)
4. Verify the auto-instrumentation resources are injected in the Pod:

  Python apps are instrumented by the OpenTelemetry Operator with the following actions:

  - It adds an init container in the Pod with the objective of copying the SDK to a shared volume.

  - Defines an `emptyDir volume` mounted in both containers.

  - Adds `PYTHONPATH` and other OTEL related environment variables.

  Run `kubectl describe pod dotnet-app-xxxx` and check:

  - There should be an init container named `opentelemetry-auto-instrumentation-dotnet` in the Pod:

    ```bash
    $ kubectl describe pod dotnet-app-8d84c47b8-8h5z2 -n dotnet
    ...
    ...
    Init Containers:
      opentelemetry-auto-instrumentation-dotnet:
        Container ID:  containerd://fdc86b3191e34ef5ec872853b14a950d0af1e36b0bc207f3d59bd50dd3caafe9
        Image:         docker.elastic.co/observability/elastic-otel-dotnet:0.3.0
        Image ID:      docker.elastic.co/observability/elastic-otel-dotnet@sha256:de7b5cce7514a10081a00820a05097931190567ec6e18a384ff7c148bad0695e
        Port:          <none>
        Host Port:     <none>
        Command:
          cp
          -r
          /autoinstrumentation/.
          /otel-auto-instrumentation-dotnet
        State:          Terminated
          Reason:       Completed
    ...
    ```

  - The main container has new environment variables: 

    ```bash
    ...
    Containers:
      dotnet-app:
    ...
        Environment:
    ...
          PYTHONPATH:                          /otel-auto-instrumentation-dotnet/opentelemetry/instrumentation/auto_instrumentation:/otel-auto-instrumentation-dotnet
          OTEL_EXPORTER_OTLP_PROTOCOL:         http/protobuf
          OTEL_TRACES_EXPORTER:                otlp
          OTEL_METRICS_EXPORTER:               otlp
          OTEL_SERVICE_NAME:                   dotnet-app
          OTEL_EXPORTER_OTLP_ENDPOINT:         http://opentelemetry-kube-stack-daemon-collector.opentelemetry-operator-system.svc.cluster.local:4318
    ...
    ```

  - The Pod has an `EmptyDir` volume named `opentelemetry-auto-instrumentation-dotnet` mounted in both the main and the init containers in path `/otel-auto-instrumentation-dotnet`:

  Ensure the environment variable `OTEL_EXPORTER_OTLP_ENDPOINT` points to a valid endpoint and there's network communication between the Pod and the endpoint.

5. Confirm data is flowing through in **Kibana**:

  - Open Observability -> Applications -> Service Inventory, and determine if:
    - The application appears in the list of services.
    - The application shows transactions and metrics.
    - If [dotnet logs instrumentation](https://opentelemetry.io/docs/kubernetes/operator/automatic/#auto-instrumenting-dotnet-logs) is enabled, the application logs should  appear in the Logs tab.
  
  - For application container logs, open **Kibana Discovery** and filter for your pods log, with any of:
    - `k8s.deployment.name: "dotnet-app"`
    - `k8s.pod.name: dotnet-app*`

    Note that the container logs are not provided by the instrumentation library, but by the DaemonSet collector deployed as part of the [operator installation](./README.md)

## Troubleshooting

- Refer to [troubleshoot auto-instrumentation](./troubleshoot-auto-instrumentation.md) for further analysis.
