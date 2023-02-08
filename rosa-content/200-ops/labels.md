## Introduction

Labels are a useful way to select which nodes that an application will run on. These nodes are created by machines which are defined by the MachineSets we worked with in previous sections of this workshop. An example of this would be running a memory intensive application only on a specific node type.

While you can directly add a label to a node, it is not recommended because nodes can be recreated, which would cause the label to disappear. Therefore we need to label the Machine pool itself. 

## Set a label for the Machine Pool

1. Just like the last section, let's use the default machine pool to add our label. To do so, run the following command:

    ```bash
    rosa edit machinepool -c ${WS_USER/_/-} --labels tier=frontend Default
    ```

1. Now, let's verify the nodes are properly labeled. To do so, run the following command:

    ```bash
    oc get nodes --selector='tier=frontend' -o name
    ```

    Your output will look something like this:

    ```{.text .no-copy}
    node/ip-10-0-130-107.{{ aws_region }}.compute.internal
    node/ip-10-0-146-138.{{ aws_region }}.compute.internal
    node/ip-10-0-161-54.{{ aws_region }}.compute.internal
    node/ip-10-0-190-160.{{ aws_region }}.compute.internal
    node/ip-10-0-218-118.{{ aws_region }}.compute.internal
    node/ip-10-0-221-22.{{ aws_region }}.compute.internal
    ```

    Pending that your output shows one or more node, this demonstrates that our machine pool and associated nodes are properly annotated!

## Deploy an app to the labeled nodes

Now that we've successfully labeled our nodes, let's deploy a workload to demonstrate app placement using `nodeSelector`. This should force our app to only our labeled nodes.

1. First, let's create a namespace (also known as a project in OpenShift). To do so, run the following command:

    ```bash
    oc new-project nodeselector-ex
    ```

1. Next, let's deploy our application and associated resources that will target our labeled nodes. To do so, run the following command:

    ```yaml
    cat << EOF | oc create -f -
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: nodeselector-app
      namespace: nodeselector-ex
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nodeselector-app
      template:
        metadata:
          labels:
            app: nodeselector-app
        spec:
          nodeSelector:
            tier: frontend
          containers:
            - name: hello-openshift
              image: "docker.io/openshift/hello-openshift"
              ports:
                - containerPort: 8080
                  protocol: TCP
                - containerPort: 8888
                  protocol: TCP
    EOF
    ```

1. Now, let's validate that the application has been deployed to one of the labeled nodes. To do so, run the following command:

    ```bash
    oc -n nodeselector-ex get pod -l app=nodeselector-app -o json \
      | jq -r .items[0].spec.nodeName
    ```

    Your output will look something like this:

    ```{.text .no-copy}
    user1-mobbws-cluster-zljxp-worker-eastus1-gkhgf
    ```

1. Double check the name of the node to compare it to the output above to ensure the node selector worked to put the pod on the correct node

    ```bash
    oc get nodes --selector='tier=frontend' -o name
    ```

    Your output will look something like this (look for the final string to match, in this example `gkhgf`)

    ```{.text .no-copy}
    node/user1-mobbws-cluster-zljxp-worker-eastus1-gkhgf
    ```


1. Next create a `service` using the `oc expose` command

    ```bash
    oc expose deployment nodeselector-app
    ```

1. Expose the newly created `service` with a `route`

    ```bash
    oc create route edge --service=nodeselector-app
    ```

1.  Fetch the URL for the newly created `route`

    ```bash
    oc get routes/nodeselector-app -o json | jq -r '.spec.host'
    ```

    Then visit the URL presented in a new tab in your web browser (using HTTPS). For example, your output will look something similar to:

    ```{.text .no-copy}
    nodeselector-app-nodeselector-ex.apps.ce7l3kf6.eastus.aroapp.io
    ```

1. In the above case, you'd visit `https://nodeselector-app-nodeselector-ex.apps.user1-mobbws.2ep4.p1.openshiftapps.com` in your browser.

    > Note the application is exposed over the default ingress using a predetermined URL and trusted TLS certificate. This is done using the OpenShift `Route` resource which is an extension to the Kubernetes `Ingress` resource.

Congratulations! You've successfully demonstrated the ability to label nodes and target those nodes using `nodeSelector`.
