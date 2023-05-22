# Login to the ROSA CLI

During this workshop, you will be working on a cluster that has been pre-created for you. This cluster is dedicated to you. In addition, all the AWS resources that you will be creating will be tagged with your user ID to ensure that only you can modify it.

1. While logged in to the cloud bastion host that you should still have open from the "Environment Setup" section, run the following command to ensure the system has the correct environment variables for your user (if not, request help from the workshop team):

    ```bash
    env | grep -E  'WS_'
    ```

1. Before we can modify our ROSA cluster, we need to login to the ROSA CLI using a token from the OpenShift Cluster Manager. To do so, [click here](https://console.redhat.com/openshift/token/rosa){:target="_blank"} and login to OCM using the provided workshop credentials.

1. Next, click the *Load Token* button to generate a new login token. 

    ![OpenShift Cluster Manager - Generate Token](../assets/images/ocm-generate-token.png){ align=center }

1. Next, copy the newly generated *Authentication Command*, **not the token itself**. 

    ![OpenShift Cluster Manager - Copy Login Command](../assets/images/ocm-copy-login-command.png){ align=center }

1. Finally, paste the newly generated command into your open terminal to login to the ROSA CLI.

    ```{.text .no-copy}
    rosa login --token="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ....."
    ```

    If everything has worked as expected, you should see output that looks like this:
    ```{.text .no-copy}
    I: Logged in as 'user#_mobbws' on 'https://api.openshift.com'
    ```

!!! note

    Normally, you would need to also authenticate against the AWS CLI in addition to logging in to the ROSA CLI. Luckily, we've already done that for you in this workshop environment!