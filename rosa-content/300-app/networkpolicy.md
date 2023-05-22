# Applying Network Policies to lock down networking

NetworkPolicies are used to control and secure communication between pods within a cluster. They provide a declarative approach to define and enforce network traffic rules, allowing you to specify the desired network behavior. By using NetworkPolicies, you can enhance the overall security of your applications by isolating and segmenting different components within the cluster. These policies enable fine-grained control over network access, allowing you to define ingress and egress rules based on criteria such as IP addresses, ports, and pod selectors.

For this module we will be applying networkpolices to the previously created 'microsweeper-ex' namespace and using the 'microsweeper' app to test these policies. In addition, we will deploy two new applications to test against the 'microsweeper app 

1. Create a new project and a new app. We will be using this pod for testing network connectivity to the microsweeper application 

    ```bash
    oc new-project networkpolicy-test
    ```
    
    Create a new application within this namespace:  

    ```bash
    cat << EOF | oc apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
      namespace: networkpolicy-test
      labels:
        app: networkpolicy
    spec:
      securityContext:
        allowPrivilegeEscalation: false
      containers:
        - name: networkpolicy-pod
          image: registry.access.redhat.com/ubi9/ubi-minimal
          command: ["sleep", "infinity"]
    EOF

    ```

    Now we will change to the microsweeper-ex project to start applying the network policies

    ```bash
    oc project microsweeper-ex 
    ```
    
1. Fetch the IP address of the `microsweeper` Pod

    ```bash
    MS_IP=$(oc -n microsweeper-ex get pod -l \
      "app.kubernetes.io/name=microsweeper-appservice" \
      -o jsonpath="{.items[0].status.podIP}")
    echo $MS_IP
    ```

1. Check to see if the `networkpolicy-pod` app can access the Pod.

    ```bash
    oc -n networkpolicy-test exec -ti pod/networkpolicy-pod -- curl $MS_IP:8080 | head
    ```

    The output should show a successful connection

    ```{.html .no-copy}
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>Microsweeper</title>
        <link rel="stylesheet" href="css/main.css">
        <script
                src="https://code.jquery.com/jquery-3.2.1.min.js"
    ```

1. It's common to want to not allow Pods from another Project. This can be done by a fairly simple Network Policy.

    !!! info "This Network Policy will restrict Ingress to the Pods in the project `microsweeper-ex` to just the OpenShift Ingress Pods and only on port 8080."

    ```yaml
    cat << EOF | oc apply -f -
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-from-openshift-ingress
      namespace: microsweeper-ex
    spec:
      podSelector: {}
      policyTypes:
       - Ingress
      ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                  network.openshift.io/policy-group: ingress
          ports:
            - protocol: TCP
              port: 8080
    EOF
    ```

1. Try to access microsweeper from networkpolicy-pod again

    ```bash
    oc -n networkpolicy-test exec -ti pod/networkpolicy-pod -- curl $MS_IP:8080 | head
    ```

    This time it should fail to connect. Hit Ctrl-C to avoid having to wait until a timeout.

    !!! info "If you have your browser still open to the microsweeper app, you can refresh and see that you can still access it."

1. Sometimes you want your application to be accessible to other namespaces. You can allow access to just your microsweeper frontend from the networkpolicy-pod in the `networkpolicy-test` namespace like so

    ```yaml
    cat <<EOF | oc apply -f -
    kind: NetworkPolicy
    apiVersion: networking.k8s.io/v1
    metadata:
      name: allow-networkpolicy-pod-ap
      namespace: microsweeper-ex
    spec:
      podSelector:
        matchLabels:
          app.kubernetes.io/name: microsweeper-appservice
      ingress:
        - from:
          - namespaceSelector:
              matchLabels:
                kubernetes.io/metadata.name: networkpolicy-test
            podSelector:
              matchLabels:
                app: networkpolicy
    EOF
    ```

1. Check to see if 'networkpolicy-pod` can access the Pod.

    ```bash
   oc -n networkpolicy-test exec -ti pod/networkpolicy-pod -- curl $MS_IP:8080 | head
    ```

    The output should show a successful connection:

    ```{.html .no-copy}
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>Microsweeper</title>
        <link rel="stylesheet" href="css/main.css">
        <script
                src="https://code.jquery.com/jquery-3.2.1.min.js"
    ```

1. To verify that only the networkpolicy-pod app can access microsweeper: 

  Create a new pod with a different label in the networkpolicy-test namespace: 
    
    ```bash
    cat << EOF | oc apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: new-test
      namespace: networkpolicy-test
      labels:
        app: new-test
    spec:
      securityContext:
        allowPrivilegeEscalation: false
      containers:
        - name: new-test
          image: registry.access.redhat.com/ubi9/ubi-minimal
          command: ["sleep", "infinity"]
    EOF

    ```
    
    Try to curl the microsweeper-ex pod: 
    
    ```bash
     oc -n networkpolicy-test exec -ti pod/new-test -- curl $MS_IP:8080 | head
     ````
    
    

    This should fail.  Hit Ctrl-C to avoid waiting for a timeout.

!!! info "For information on setting default network policies for new projects you can read the OpenShift documentation on [modifying the default project template](https://docs.openshift.com/container-platform/4.10/networking/network_policy/default-network-policy.html)."
