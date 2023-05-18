# Configure Log Forwarding to AWS CloudWatch

Red Hat OpenShift Service on AWS (ROSA) clusters store log data inside the cluster by default. You also have the ability to use the provided tooling to forward your cluster logs to various locations, including [AWS CloudWatch](https://aws.amazon.com/cloudwatch/){:target="_blank"}. 

In this section of the workshop, we'll configure ROSA to forward logs to AWS CloudWatch.

## Prepare AWS CloudWatch

1. First, let's set some helper variables that we'll need throughout this section of the workshop. To do so, run the following command:

    ```bash
    export OIDC_ENDPOINT=$(oc get authentication.config.openshift.io \
      cluster -o json | jq -r .spec.serviceAccountIssuer | \
      sed  's|^https://||')
    export POLICY_ARN=$(aws iam list-policies --query \
      "Policies[?PolicyName=='RosaCloudWatch'].{ARN:Arn}" \
      --output text)
    ```

1. Next, let's create a trust policy document which will define what service account can assume our role. To create the trust policy document, run the following command:

    ```bash
    cat <<EOF > ./cloudwatch-trust-policy.json
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::$(aws sts get-caller-identity --query "Account" --output text):oidc-provider/${OIDC_ENDPOINT}"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {
            "${OIDC_ENDPOINT}:sub": "system:serviceaccount:openshift-logging:logcollector"
          }
        }
      }]
    }
    EOF
    ```

1. Next, let's take the trust policy document and use it to create a role. To do so, run the following command:

    ```bash
    ROLE_ARN=$(aws iam create-role --role-name "${WS_USER/_/-}-RosaCloudWatch" \
      --assume-role-policy-document file://cloudwatch-trust-policy.json \
      --tags "Key=rosa-workshop,Value=true" \
      --query Role.Arn --output text)
    echo ${ROLE_ARN}
    ```

1. Now, let's attach the pre-created `RosaCloudWatch` IAM policy to the newly created IAM role. 

    ```bash
    aws iam attach-role-policy \
      --role-name "${WS_USER/_/-}-RosaCloudWatch" \
      --policy-arn "${POLICY_ARN}"
    ```

## Configure Cluster Logging

1. Now, we need to deploy the OpenShift Cluster Logging Operator. To do so, run the following command:

    ```bash
    cat << EOF | oc apply -f -
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      labels:
       operators.coreos.com/cluster-logging.openshift-logging: ""
      name: cluster-logging
      namespace: openshift-logging
    spec:
      channel: stable
      installPlanApproval: Automatic
      name: cluster-logging
      source: redhat-operators
      sourceNamespace: openshift-marketplace
    EOF
    ```

1. Now, we will wait for the OpenShift Cluster Logging Operator to install. To do so, we can run the following command to watch the status of the installation:

    ```bash
    oc -n openshift-logging rollout status deployment \
      cluster-logging-operator
    ```

    After a minute or two, your output should look something like this:

    ```{.text .no-copy}
    deployment "cluster-logging-operator" successfully rolled out
    ```

1. Next, we need to create a secret containing the ARN of the IAM role that we previously created above. To do so, run the following command:

    ```bash
    cat << EOF | oc apply -f -
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudwatch-credentials
      namespace: openshift-logging
    stringData:
      role_arn: ${ROLE_ARN}
    EOF
    ```

1. Next, let's configure the OpenShift Cluster Logging Operator by creating a Cluster Log Forwarding custom resource that will forward logs to AWS CloudWatch. To do so, run the following command:

    ```bash
    cat << EOF | oc apply -f -
    apiVersion: "logging.openshift.io/v1"
    kind: ClusterLogForwarder
    metadata:
      name: instance
      namespace: openshift-logging
    spec:
      outputs:
        - name: cw
          type: cloudwatch
          cloudwatch:
            groupBy: namespaceName
            groupPrefix: rosa-${WS_USER/_/-}
            region: ${AWS_DEFAULT_REGION}
          secret:
            name: cloudwatch-credentials
      pipelines:
         - name: to-cloudwatch
           inputRefs:
             - infrastructure
             - audit
             - application
           outputRefs:
             - cw
    EOF
    ```

1. Next, let's create a Cluster Logging custom resource which will enable the OpenShift Cluster Logging Operator to start collecting logs. 

    ```bash
    cat << EOF | oc apply -f -
    apiVersion: logging.openshift.io/v1
    kind: ClusterLogging
    metadata:
      name: instance
      namespace: openshift-logging
    spec:
      collection:
        logs:
           type: fluentd
      forwarder:
        fluentd: {}
      managementState: Managed
    EOF
    ```

1. After a few minutes, you should begin to see log groups inside of AWS CloudWatch. To do so, run the following command:

    ```bash
    aws logs describe-log-groups \
      --log-group-name-prefix rosa-${WS_USER/_/-}
    ```

    You should see two log groups (`audit` and `infrastructure`).

    !!! info "check back later after you've run some applications and you'll see a third `application` log group"

    ```{.json .no-copy}
        {
       "logGroups": [
          {
                "logGroupName": "rosa-xxxx.audit",
                "creationTime": 1661286368369,
                "metricFilterCount": 0,
                "arn": "arn:aws:logs:us-east-1:xxxx:log-group:rosa-xxxx.audit:*",
                "storedBytes": 0
          },
          {
                "logGroupName": "rosa-xxxx.infrastructure",
                "creationTime": 1661286369821,
                "metricFilterCount": 0,
                "arn": "arn:aws:logs:us-east-1:xxxx:log-group:rosa-xxxx.infrastructure:*",
                "storedBytes": 0
          }
       ]
    }
    ```

Congratulations! You've successfully forwarded your cluster's logs to the AWS CloudWatch service.

## Summary and Next Steps

Here you learned:

* Create trust policy and role to grant ROSA cluster access to AWS CloudWatch
* Install `cluster-logging` operator in the ROSA cluster
* Configure `ClusterLogForwarder` and `ClusterLogging` object to forward infrastructre, audit and application logs to AWS CloudWatch 

Next you will learn:

* Deploy and Expose an App
