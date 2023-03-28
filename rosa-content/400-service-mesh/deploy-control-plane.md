Based on the open source Istio project, Red Hat OpenShift Service Mesh adds a transparent layer on existing distributed applications without requiring any changes to the service code. You add Red Hat OpenShift Service Mesh support to services by deploying a special sidecar proxy to relevant services in the mesh that intercepts all network communication between microservices. You configure and manage the Service Mesh using the Service Mesh control plane features. To learn more about the OpenShift Service Mesh, review the [OpenShift documentation](https://docs.openshift.com/rosa/service_mesh/v2x/ossm-about.html){:target="_blank"}.

## Deploy Control Plane

1. First, let's create a project (namespace) for us to deploy the service mesh control plane into. To do so, run the following command:

     ```bash
     oc new-project istio-system
     ```

1. Next, let's deploy the service mesh control plane. To do so, run the following command:

    ```yaml
    cat << EOF | oc apply -f -
    apiVersion: maistra.io/v2
    kind: ServiceMeshControlPlane
    metadata:
      name: basic
      namespace: istio-system
    spec:
      version: v2.2
      security:
        identity:
          type: ThirdParty
      tracing:
        type: Jaeger
        sampling: 10000
      addons:
        jaeger:
          name: jaeger
          install:
            storage:
              type: Memory
        kiali:
          enabled: true
          name: kiali
        grafana:
          enabled: true
    EOF
    ```

    !!! warning
            If you receive an error that is similar to the below error, ensure your operators have finished installing and try again:

            ```{.text .no-copy}
            Internal error occurred: failed calling webhook "smcp.mutation.maistra.io": failed to call webhook: Post "https://maistra-admission-controller.openshift-operators.svc:443/mutate-smcp?timeout=10s": dial tcp 10.128.2.63:11999: connect: connection refused
            ```

1. Next, let's watch the progress of the service mesh control plane rollout. To do so, run run the following command:

    ```bash
    oc get pods -n istio-system -w
    ```
    You should see output similar to the following:

    ```{.text .no-copy}
    NAME                                   READY   STATUS    RESTARTS   AGE
    grafana-b4d59bd7-mrgbr                 2/2     Running   0          65m
    istio-egressgateway-678dc97b4c-wrjkp   1/1     Running   0          108s
    istio-ingressgateway-b45c9d54d-4qg6n   1/1     Running   0          108s
    istiod-basic-55d78bbbcd-j5556          1/1     Running   0          108s
    jaeger-67c75bd6dc-jv6k6                2/2     Running   0          65m
    kiali-6476c7656c-x5msp                 1/1     Running   0          43m
    prometheus-58954b8d6b-m5std            2/2     Running   0          66m
    wasm-cacher-basic-8c986c75-vj2cd       1/1     Running   0          65m
    ```

    Once all the pods are running, hit Ctrl-C and proceed to the next step.

1. Next, let's verify that the service mesh control plane is successfully installed. To do so, run the following command:

    ```bash
    oc -n istio-system get smcp
    ```
    The installation has finished successfully when the STATUS column says `ComponentsReady`.

    ```{.text .no-copy}
    NAME    READY   STATUS            PROFILES      VERSION   AGE
    basic   10/10   ComponentsReady   ["default"]   2.1.1     66m
    ```

    Congratulations! You've successfully deployed the OpenShift Service Mesh control plane to your cluster.