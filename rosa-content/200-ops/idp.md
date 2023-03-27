# Configuring ROSA to use Amazon Cognito for authentication

Your Red Hat OpenShift Service on AWS (ROSA) cluster has a built-in OAuth server. Developers and administrators do not really directly interact with the OAuth server itself, but instead interact with an external identity provider (such as Amazon Cognito) which is brokered by the OAuth server in the cluster. To learn more about cluster authentication, visit the [Red Hat documentation for identity provider configuration](https://docs.openshift.com/container-platform/latest/authentication/understanding-identity-provider.html){:target="_blank"}.

In this section of the workshop, we'll configure Amazon Cognito as the cluster identity provider in your ROSA cluster.

## Configure Amazon Cognito to allow authentication from ROSA

1. First, we need to determine the OAuth callback URL, which we will use to tell Amazon Cognito where it should send authentication responses. To do so, run the following command:

    ```bash
    domain=$(rosa describe cluster -c ${WS_USER/_/-} | grep "DNS" | grep -oE '\S+.openshiftapps.com')
    echo "OAuth callback URL: https://oauth-openshift.apps.$domain/oauth2callback/Cognito"
    ```

1. Next, let's create an app client in Amazon Cognito. To do so, run the following command:

    ```bash
    aws cognito-idp create-user-pool-client \
    --user-pool-id ${WS_COGNITO_ID} \
    --client-name ${WS_USER/_/-} \
    --generate-secret \
    --callback-urls ${IDP_CALLBACK} \
    --supported-identity-providers COGNITO \
    --allowed-o-auth-scopes "phone" "email" "openid" "profile" \
    --allowed-o-auth-flows code \
    --allowed-o-auth-flows-user-pool-client
    ```

1. Next, let's use the manifest we created above to create an Azure AD App for your cluster. To do so, run the following command:

    ```bash
    az ad app create \
      --display-name ${AZ_USER}-idp \
      --web-redirect-uris ${IDP_CALLBACK} \
      --sign-in-audience AzureADMyOrg \
      --optional-claims @manifest.json
    APPID=$(az ad app list --display-name ${AZ_USER}-idp --query "[].appId" -o tsv)
    ```

    The output of the command will look something like this:

    ```json
    "UserPoolClient": {
        "UserPoolId": "{{ aws_region }}_i5V2Mxaya",
        "ClientName": "user1-mobbws",
        "ClientId": "abcdefghijklmnopqrstuvwxyz",
        "ClientSecret": "redacted",
        ...
    ```

    **Ensure that you capture the `ClientId` and `ClientSecret` before moving on to the next steps and store it somewhere safe.**

## Configure our ROSA cluster to use Amazon Cognito for authentication

1. Finally, we need to configure OpenShift to use Amazon Cognito as its identity provider. While Red Hat OpenShift Service on AWS (ROSA) offers the ability to configure identity providers via the OpenShift Cluster Manager (OCM),we're going to configure the cluster’s OAuth provider via the rosa CLI. To do so, run the following command, making sure to replace the variable specified:
    ```bash
    CLIENT_ID=replaceme # Replace this with the ClientId from the step above
    CLIENT_SECRET=replaceme # Replace this with the ClientSecret from the step above
    rosa create idp \
    --cluster ${WS_USER/_/-} \
    --type openid \
    --name Cognito \
    --client-id ${CLIENT_ID} \
    --client-secret ${CLIENT_SECRET} \
    --issuer-url https://cognito-idp.{{ aws_region }}.amazonaws.com/${WS_COGNITO_ID} \
    --email-claims email \
    --name-claims name \
    --username-claims preferred_username
    ```

1. Next, let's give Cluster Admin permissions to your Amazon Cognito user by running the following commands:

    ```bash
    oc adm policy add-cluster-role-to-user cluster-admin ${WS_USER}
    ```

    !!! note "You're logged in to the OpenShift CLI already, so you can run this command. You can also grant these permissions via the rosa CLI by using the `rosa grant user cluster-admin` command.

1. Logout from your OpenShift Web Console and browse back to the Console URL (`rosa describe cluster -c mrmc-idp-tst -o json | jq -r '.console.url'` if you have forgotten it) and you should see a new option to login called `Cognito`. Select that, and log in using your workshop Azure credentials.

    !!! warning "If you do not see a new **Cognito** login option, wait a few more minutes as this process can take a few minutes to deploy across the cluster and revisit the Console URL."
