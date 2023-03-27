# Introduction

OpenShift stores logs and metrics inside the cluster by default, however it also provides tooling to forward both to various locations. Here we will configure your ROSA cluster to ship logs to AWS CloudWatch.


## Environment Setup

1. Create a working directory

    ```bash
    mkdir -p ~/workshop/o11y/cloudwatch
    cd ~/workshop/o11y/cloudwatch
    ```

1. Set some environment variables to use through this chapter

    ```bash
    export CLUSTER_NAME="${WS_USER/_/-}"
    export REGION="${AWS_DEFAULT_REGION}"
    export OIDC_ENDPOINT=$(oc get authentication.config.openshift.io \
      cluster -o json | jq -r .spec.serviceAccountIssuer | \
      sed  's|^https://||')
    export POLICY_ARN=$(aws iam list-policies --query \
      "Policies[?PolicyName=='RosaCloudWatch'].{ARN:Arn}" \
      --output text)
    ```

1. Create an IAM Role trust policy for the cluster

    ```bash
    cat <<EOF > ./trust-policy.json
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::${WS_AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT}"
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

    ROLE_ARN=$(aws iam create-role --role-name "${CLUSTER_NAME}-RosaCloudWatch" \
      --assume-role-policy-document file://./trust-policy.json \
      --tags "Key=rosa-workshop,Value=true" \
      --query Role.Arn --output text)
    echo ${ROLE_ARN}
    ```

1. Attach the pre-created `RosaCloudWatch` IAM Policy to the new Role

    ```bash
    aws iam attach-role-policy \
      --role-name "${CLUSTER_NAME}-RosaCloudWatch" \
      --policy-arn "${POLICY_ARN}"
    ```

1. Deploy the Cluster Logging Operator

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

1. Wait for the Cluster Logging Operator to be ready

    ```bash
    oc -n openshift-logging rollout status deployment \
      cluster-logging-operator
    ```

    After a minute or so you should see

    ```{.text .no-copy}
    deployment "cluster-logging-operator" successfully rolled out
    ```

1. Create a secret containing the Role ARN for the Cluster Logging resource to use

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

1. Create a Cluster Log Fowarding Resource

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
            groupPrefix: rosa-${CLUSTER_NAME}
            region: ${REGION}
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

1. Create a Cluster Logging Resource

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

1. Wait a few minutes then check Cloud Watch for log groups

    ```bash
    aws logs describe-log-groups \
      --log-group-name-prefix rosa-${CLUSTER_NAME}
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

1. Change back to your home directory

    ```bash
    cd ~
    ```
