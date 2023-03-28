Now, let's proceed with deploying a workload to test our service mesh. 

### Configure our project for the service mesh

1. First, let's create a project (namespace) for us to deploy our workload into. To do so, run the following command:

    ```bash
    oc new-project bookinfo
    ```

1. Next, let's label that project (namespace) to enable the service mesh injection for all applications in the project.

    ```bash
    oc label namespace bookinfo istio-injection=enabled
    ```

1. Even though we've enabled service mesh injection in the project, we also need to add our project (namespace) to the `ServiceMeshMemberRoll`, which limits the scope of the service mesh control plane to only projects in the member roll. To add the project to the `ServiceMeshMemberRoll`, run the following command:

    ```bash
    cat << EOF | oc apply -f -
    apiVersion: maistra.io/v1
    kind: ServiceMeshMemberRoll
    metadata:
      name: default
      namespace: istio-system
    spec:
      members:
      - bookinfo
    EOF
    ```

1. Next, let's verify the `ServiceMeshMemberRoll` was created successfully. To do so, run the following command:

    ```bash
    oc -n istio-system get smmr -o wide
    ```

    The service mesh member roll was successfully configured when the STATUS column is `Configured` and your project shows up in the MEMBERS column.

    ```{.text .no-copy}
    NAME      READY   STATUS       AGE   MEMBERS
    default   1/1     Configured   70s   ["bookinfo"]
    ```

### Deploy our test workload

1. Now that we've configured the service mesh for our project, let's deploy our bookinfo workload into our project. To do so, run the following command to create the necessary resources:

    ```bash
    oc -n bookinfo apply -f \
      https://raw.githubusercontent.com/rh-mobb/rosa-workshop-content/main/rosa-content/assets/scripts/bookinfo.yaml
    ```

    You should see output similar to the following:

    ```{.text .no-copy}
    service/details created
    serviceaccount/bookinfo-details created
    deployment.apps/details-v1 created
    service/ratings created
    serviceaccount/bookinfo-ratings created
    deployment.apps/ratings-v1 created
    service/reviews created
    serviceaccount/bookinfo-reviews created
    deployment.apps/reviews-v1 created
    deployment.apps/reviews-v2 created
    deployment.apps/reviews-v3 created
    service/productpage created
    serviceaccount/bookinfo-productpage created
    deployment.apps/productpage-v1 created
    ```

    Interested in seeing the configuration you're deploying? Check it out on GitHub [here](https://github.com/rh-mobb/rosa-workshop-content/blob/main/rosa-content/assets/scripts/bookinfo.yaml){:target="_blank"}.

1. Now, let's create the service mesh ingress gateway. To do so, run the following command:

    ```bash
    oc -n bookinfo apply -f \
      https://raw.githubusercontent.com/rh-mobb/rosa-workshop-content/main/rosa-content/assets/scripts/bookinfo-gateway.yaml
    ```

    You should see output similar to the following:

    ```bash
    gateway.networking.istio.io/bookinfo-gateway created
    virtualservice.networking.istio.io/bookinfo created
    ```

1. Next, let's add some destination rules. Destination rules define policies that apply to traffic intended for a service after routing has occurred. To create the rules, run the following command:

    ```bash
    oc -n bookinfo apply -f \
      https://raw.githubusercontent.com/rh-mobb/rosa-workshop-content/main/rosa-content/assets/scripts/destination-rule-all.yaml
    ```

    You should see output similar to the following:

    ```{.text .no-copy}
    destinationrule.networking.istio.io/productpage created
    destinationrule.networking.istio.io/reviews created
    destinationrule.networking.istio.io/ratings created
    destinationrule.networking.istio.io/details created
    ```

### Verifying the Bookinfo installation

1. First, let's verify that all pods are running and ready by running the following command:

    ```bash
    oc -n bookinfo get pods
    ```

    All pods should have a status of `Running`. You should see output similar to the following:

    ```{.text .no-copy}
    NAME                              READY   STATUS    RESTARTS   AGE
    details-v1-55b869668-jh7hb        2/2     Running   0          12m
    productpage-v1-6fc77ff794-nsl8r   2/2     Running   0          12m
    ratings-v1-7d7d8d8b56-55scn       2/2     Running   0          12m
    reviews-v1-868597db96-bdxgq       2/2     Running   0          12m
    reviews-v2-5b64f47978-cvssp       2/2     Running   0          12m
    reviews-v3-6dfd49b55b-vcwpf       2/2     Running   0          12m
    ```

1. Next, let's get the URL for the product page. To do so, run the following command:

    ```bash
    echo "http://$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')/productpage"
    ```

1. Copy and paste the URL provided in the previous step into your web browser and verify the Bookinfo product page is successfully deployed. You should see a book review of "The Comedy of Errors".
