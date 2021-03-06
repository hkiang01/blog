---
title: "Keycloak"
weight: 1
draft: false
---

[Keycloak] is an identity management solution useful for securing applications in Kubernetes.
This post shows how to set up Keycloak and secure an application in Kubernetes with Keycloak.
Sample code available at https://github.com/hkiang01/keycloak-demo.


- [Installing Keycloak](#installing-keycloak)
  - [Simple first](#simple-first)
  - [Database for production readiness](#database-for-production-readiness)
    - [Creating the chart](#creating-the-chart)
    - [Adding Postgres](#adding-postgres)
- [Preparing Keycloak](#preparing-keycloak)
  - [Create a Realm](#create-a-realm)
  - [Create a Client](#create-a-client)
  - [Create a User](#create-a-user)
  - [Testing out your User](#testing-out-your-user)
- [Securing your application](#securing-your-application)
  - [Setting up the gatekeeper sidecar](#setting-up-the-gatekeeper-sidecar)
  - [Deploy secured app](#deploy-secured-app)
  - [Fixing the audience](#fixing-the-audience)
- [Some understanding](#some-understanding)
- [Next steps](#next-steps)


## Installing Keycloak

### Simple first

We're going to see what it takes to minimally get Keycloak to run in Kubernetes.
First, we'll create a namespace called keycloak

```bash
kubectl create namespace keycloak
```

We'll then get the quickstart manifest from https://github.com/keycloak/keycloak-quickstarts/tree/latest/kubernetes-examples.
I've made a few edits, namely:

- Service type changed to ClusterIP
- namespaces changed to "keycloak"

I'll call mine keycloak.yaml

```yaml
# keycloak.yaml
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
  labels:
    app: keycloak
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: keycloak
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
  labels:
    app: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:11.0.2
        env:
        - name: KEYCLOAK_USER
          value: "admin"
        - name: KEYCLOAK_PASSWORD
          value: "admin"
        - name: PROXY_ADDRESS_FORWARDING
          value: "true"
        ports:
        - name: http
          containerPort: 8080
        - name: https
          containerPort: 8443
        readinessProbe:
          httpGet:
            path: /auth/realms/master
            port: 8080
```

Let's apply it
```zsh
kubectl apply -f keycloak.yaml
```

Let's make sure it's up and running:

```zsh
kubectl get all -n keycloak
```

You should see that the pods are all ready.
Below, 1 of 1 pods are ready, as indicated by 1/1.

```txt
NAME                            READY   STATUS    RESTARTS   AGE
pod/keycloak-6bc5f6d94c-bdbln   1/1     Running   0          3m25s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/keycloak   ClusterIP   10.152.183.91   <none>        8080/TCP   3m26s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keycloak   1/1     1            1           3m25s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/keycloak-6bc5f6d94c   1         1         1       3m25s
```

To access the app, we'll need to port-forward the service:
```zsh
kubectl -n keycloak port-forward svc/keycloak 8080
```

You should see output like the following:
```txt
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

We should now be able to access the instance at http://localhost:8080 in our browser.
Before we continue, let's do some cleanup:

```zsh
kubectl delete -f keycloak.yaml
```

### Database for production readiness

#### Creating the chart

Keycloak is a stateful application that is backed by a database like Postgres.
We're going to take the above manifest from [Installing Keycloak](##installing-keycloak) and paste it as a template in a new Helm chart with Postgres as a package [dependency](https://helm.sh/docs/chart_best_practices/dependencies/#helm).
You can use an existing chart, but I think it's valuable to see what a minimal chart looks like.

Let's first create a chart called keycloak:

```zsh
helm create keycloak
```

This is the file tree of what's created

```
├── keycloak
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── NOTES.txt
│   │   ├── serviceaccount.yaml
│   │   ├── service.yaml
│   │   └── tests
│   │       └── test-connection.yaml
│   └── values.yaml
```

We can get rid of the following in templates/:
- hpa.yaml
- tests/

```zsh
cd keycloak/templates/
rm -rf hpa.yaml tests/
```

Let's configure the chart to use the keycloak image.
We can accomplish this with the follwing in values.yaml at the root of the chart.

```yaml
# keycloak/values.yaml
image:
  repository: quay.io/keycloak/keycloak
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "11.0.2"
```

And configure our credentials:

{{% notice warning %}}
You should never store secrets in plaintext in your repo history.
The snippet below is for demonstration purposes only.
{{% /notice %}}

```yaml
# keycloak/values.yaml
username: admin
password: supersecretpassword
```

{{% notice tip %}}
You can use [Sealed Secrets for Kubernetes](https://github.com/bitnami-labs/sealed-secrets) and store secrets in encrypted form in your chart's templates.
See [Usage](https://github.com/bitnami-labs/sealed-secrets#usage) to get started.
{{% /notice %}}

The container environment variables, ports, and probes will have to be copied over as well.
```yaml
# keycloak/templates/deployment.yaml
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
          - name: KEYCLOAK_USER
            value: {{ .Values.username }}
          - name: KEYCLOAK_PASSWORD
            value: {{ .Values.password }}
          - name: PROXY_ADDRESS_FORWARDING
            value: "true"
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: https
              containerPort: 8443
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /auth/realms/master
              port: http
```

We can now test our chart to make sure we have the same level of access as before.
Perform the following command in a directory containing the keycloak/ directory that is the chart we created:

```zsh
helm -n keycloak upgrade --install keycloak ./keycloak
```

Port-forward the service:

```
kubectl -n keycloak port-forward svc/keycloak 8080:80
```

We should now be able to access the instance at http://localhost:8080 in our browser.

#### Adding Postgres

The reason we want to add Postgresql via a Helm dependency is that a lot of the legwork with respect to ensuring availability and persistence is already done for you.
There are packages that offer [high availability](https://artifacthub.io/packages/helm/bitnami/postgresql-ha), but here we'll just go for the standard [postgresql](https://artifacthub.io/packages/helm/bitnami/postgresql) package.

Let's add the dependency:

```yaml
# keycloak/Chart.yaml
dependencies:
- name: postgresql
  version: 9.8.6
  repository: https://charts.bitnami.com/bitnami
```

Pull the dependency:

```zsh
helm dependency update ./keycloak
```

You should see output like below:

```txt
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading postgresql from repo https://charts.bitnami.com/bitnami
Deleting outdated charts
```

You can now observe the chart itself.
We're going to configure it by continuing to edit the same values.yaml file we used to set the chart's image to keycloak to configure out secrets

{{% notice warning %}}
You should never store secrets in plaintext in your repo history.
The snippet below is for demonstration purposes only.
{{% /notice %}}


```yaml
# keycloak/values.yaml
postgresql:
  postgresqlUsername: postgres
  postgresqlPassword: secretpassword
  postgresqlDatabase: keycloak
  service:
    port: 5432
```

{{% notice tip %}}
Use a Kubernetes Secret to define your credentials and point `postgresql.existingSecret` in values.yaml to it.
You can use [Sealed Secrets for Kubernetes](https://github.com/bitnami-labs/sealed-secrets) and store secrets in encrypted form in your chart's templates.
See [Usage](https://github.com/bitnami-labs/sealed-secrets#usage) to get started.
{{% /notice %}}

Now we have to make Keycloak talk to Postgres.

```yaml
# keycloak/templates/deployment.yaml
          env:
          - name: KEYCLOAK_USER
            value: {{ .Values.username }}
          - name: KEYCLOAK_PASSWORD
            value: {{ .Values.password }}
          - name: PROXY_ADDRESS_FORWARDING
            value: "true"
          - name: DB_VENDOR
            value: postgres
          - name: DB_ADDR
            value: {{ include "keycloak.fullname" . }}-postgresql
          - name: DB_PORT
            value: {{ .Values.postgresql.service.port | quote }}
          - name: DB_DATABASE
            value: {{ .Values.postgresql.postgresqlDatabase }}
          - name: DB_USER
            value: {{ .Values.postgresql.postgresqlUsername }}
          - name: DB_PASSWORD
            value: {{ .Values.postgresql.postgresqlPassword }}
```

Deploy the chart:

```zsh
helm -n keycloak upgrade --install keycloak ./keycloak
```

You should see output like the following:

```txt
Release "keycloak" has been upgraded. Happy Helming!
NAME: keycloak
LAST DEPLOYED: Sun Oct 25 01:02:10 2020
NAMESPACE: keycloak
STATUS: deployed
REVISION: 5
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace keycloak -l "app.kubernetes.io/name=keycloak,app.kubernetes.io/instance=keycloak" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace keycloak port-forward $POD_NAME 8080:80
```


By following the instructions from the output above, you should be able to access an instance of keycloak backed by Postgres.

To validate that your instance is backed by Postgres, you can tail the logs of the Keycloak pod:

```zsh
kubectl -n keycloak logs -f -l=app.kubernetes.io/name=keycloak
```

You should see output like below:

```txt
=========================================================================

  Using PostgreSQL database

=========================================================================
...
```

## Preparing Keycloak

### Create a Realm
Keycloak has Realms:

> Keycloak supports multiple tenancy where all users, clients, and so on are grouped in what is called a realm. Each realm is independent of other realms.
>
> \- [Securing Applications and Services Guide](https://www.keycloak.org/docs/latest/securing_apps/index.html)

By default, Keycloak sets up a "Master" Realm that can only be used to create other Realms.
Let's create a Realm and call it "demo" by following the instructions outlined in [Creating a realm](https://www.keycloak.org/docs/latest/getting_started/#_create-realm).
Once the Realm is created you should see something like this:

![demo-realm](https://www.keycloak.org/docs/latest/getting_started/keycloak-images/demo-realm.png)

### Create a Client

> In order for an application or service to utilize Keycloak it has to register a client in Keycloak.
>
> \- [Client Registration](https://www.keycloak.org/docs/latest/securing_apps/#_client_registration)

We'll have to create an OIDC client.
Let's do so by following the instructions outlined in [OIDC Clients](https://www.keycloak.org/docs/latest/server_admin/#oidc-clients).
It's a good idea to enter a "Root URL" as the resultant Client Settings page will fill out all the nuanced fields for you.

![sample_client](https://www.keycloak.org/docs/latest/server_admin/keycloak-images/add-client-oidc.png)

In Client Settings, scroll down and ensure "Direct Access Grants Enabled" is toggled to "On" and change the "Access Type" to confidential, then click "Save".
You should now see a "Credentials" tab appear in the client like below (see [Confidential Client Credentials](https://www.keycloak.org/docs/latest/server_admin/#_client-credentials)):

![credentials_tab](https://www.keycloak.org/docs/latest/server_admin/keycloak-images/client-credentials.png)

Note the value in "Secret", we're going to use that when performing our password grant.

### Create a User

Next, create a user by following the instructions outlined in [Creating a user](https://www.keycloak.org/docs/latest/getting_started/#creating-a-user).

![sample_user](https://www.keycloak.org/docs/latest/getting_started/keycloak-images/add-user.png)

Then give the user a password by filling out the form under the "Credentials" tab for the user.

![user_creds](https://www.keycloak.org/docs/latest/getting_started/keycloak-images/user-credentials.png)

For demonstration purposes, I'm opting to flip the "Temporary" switch to "Off".

You should then be able to log in by following the instructions outlined in [Logging into the account console](https://www.keycloak.org/docs/latest/getting_started/#logging-into-the-account-console).
You should see something like this:

![johndoe](https://www.keycloak.org/docs/latest/getting_started/images/account-console.png)

Verify that your user has access to your application by observing your client that you created in the list of "Applications"

![applictions](../kubernetes_files/applications.png)

Note that "myapp" is in the above list of Applications.

### Testing out your User

Now make a Direct Grant as described in the [Resource Owner Password Credentials](https://www.keycloak.org/docs/latest/securing_apps/#_resource_owner_password_credentials_flow).
Here is an example:

```zsh
curl --request POST 'https://keycloak.harrisonkiang.com/auth/realms/demo/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'username=johndoe' \
--data-urlencode 'password=johndoe' \
--data-urlencode 'client_id=myapp' \
--data-urlencode 'client_secret=90a47b27-6de6-43e9-a300-4eb02f18b447' \
--data-urlencode 'grant_type=password'
```

The `username` and `password` are that of the user, the `client_id` is the name of the client, and the `client_secret` is the value of "Secret" in the "Credentials" tab of the client.
You should get back a response with an access token, refresh token, etc.

## Securing your application

### Setting up the gatekeeper sidecar

We're going to use [Keycloak Gatekeeper](https://www.keycloak.org/docs/latest/securing_apps/#_keycloak_generic_adapter).
It will be set up as a [sidecar](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers) in the same pod containing our application.


Let's first create an application:

```zsh
helm create app
```

The following files will be created:

```
app
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

We can get rid of the following in templates/:
- hpa.yaml
- tests/

```zsh
cd app/templates/
rm -rf hpa.yaml tests/
```

We're going to [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) to store our Keycloak.

```yaml
# app/templates/configmap.yaml
data:
    keycloak-gatekeeper.yaml: |
        # is the url for retrieve the OpenID configuration - normally the <server>/auth/realm/<realm_name>
        discovery-url: {{ .Values.keycloak.discoveryUrl }}
        # the client id for the 'client' application
        client-id: {{ .Values.keycloak.clientId }}
        # the secret associated to the 'client' application
        client-secret: {{ .Values.keycloak.clientSecret }}
        # enforces the cookie to be secure"`
        secure-cookie: false
        # the interface definition you wish the proxy to listen, all interfaces is specified as ':<port>', unix sockets as unix://<REL_PATH>|</ABS PATH>
        listen: :3000
        # whether to enable refresh tokens
        enable-refresh-tokens: true
        # the redirection url, essentially the site url, note: /oauth/callback is added at the end
        redirection-url: http://{{ get $hosts "host"}}
        # the encryption key used to encode the session state
        encryption-key: "{{ randAlphaNum 32 }}"
        # the upstream endpoint which we should proxy request
        upstream-url: http://127.0.0.1:80
```

See [Configuration options](https://www.keycloak.org/docs/latest/securing_apps/#configuration-options) for details on each config.
We'll need to set some of the values we use in the above config.

```yaml
# app/values.yaml
keycloak:
  clientId: app
  clientSecret: 90a47b27-6de6-43e9-a300-4eb02f18b447
  discoveryUrl: https://keycloak.harrisonkiang.com/auth/realms/demo
```

The client secret is relatively safe to share, as the user will require valid credentials to access the protected resources in your app, which is the whole point of securing your app in the first place.


I'm using a TLS secret (see below):

```yaml
# app/values.yaml
ingress:
  tls:
   - secretName: harrisonkiang-dot-com-wildcard-tls
```

{{% notice note %}}
The above TLS secret is not provided in the code samples at https://github.com/hkiang01/keycloak-demo.
You can configure your own [TLS secrets](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets) or else rely on the [Default SSL Certificate](https://kubernetes.github.io/ingress-nginx/user-guide/tls/#default-ssl-certificate) created by [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/).
{{% /notice %}}

Of course we'll need to use said configmap:

```yaml
# app/templates/deployment.yaml
      containers:
        - name: gatekeeper
          image: quay.io/louketo/louketo-proxy:1.0.0
          args:
          - --config=/etc/config/keycloak-gatekeeper.yaml
          volumeMounts:
          - name: gatekeeper-config
            mountPath: /etc/config
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
          - name: gatekeeper
            containerPort: 3000
            protocol: TCP
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      volumes:
      - name: gatekeeper-config
        configMap:
          name: {{ include "app.fullname" . }}-gatekeeper
```

### Deploy secured app

Let's deploy

```zsh
helm -n keycloak upgrade --install app app
```

Now let's make sure everything is ready

```zsh
helm -n keycloak get manifest app | kubectl -n keycloak get -f -
NAME                 SECRETS   AGE
serviceaccount/app   1         29m

NAME                       DATA   AGE
configmap/app-gatekeeper   1      29m

NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/app   ClusterIP   10.152.183.125   <none>        80/TCP    29m

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app   1/1     1            1           29m

NAME                            CLASS    HOSTS                   ADDRESS     PORTS     AGE
ingress.networking.k8s.io/app   <none>   app.harrisonkiang.com   127.0.0.1   80, 443   29m
```

and that the pod is up

```zsh
kubectl -n keycloak get pod -l=app.kubernetes.io/instance=app

NAME                   READY   STATUS    RESTARTS   AGE
app-78d9484d86-8hvsd   2/2     Running   0          31m
```

2/2 means we're good to go.
Now you should be able to _almost_ log in as normal.

### Fixing the audience

You'll find that after trying to log in your app via the browser you won't get what you expect.
This is due to the fact that you have to manually add your `client_id` to the audience field `aud` of the access token per [this SO post](https://stackoverflow.com/a/53627747), the important bit of which is in the snippet below:

> Configure audience in Keycloak
>
>   - Add realm or configure existing
>   - Add client my-app or use existing
>   - Goto to the newly added "Client Scopes" menu [1]
>       - Add Client scope 'good-service'
>       - Within the settings of the 'good-service' goto Mappers tab
>           - Create Protocol Mapper 'my-app-audience'
>               - Name: my-app-audience
>               - Choose Mapper type: Audience
>               - Included Client Audience: my-app
>               - Add to access token: on
>   - Configure client my-app in the "Clients" menu
>       - Client Scopes tab in my-app settings
>       - Add available client scopes "good-service" to assigned default client scopes

After following the above you should be able to successfully acces your application!

## Some understanding

To understand the flow, let's examine how the traffic travels from the outside to the app with and without Keycloak.

{{<mermaid align="left">}}
graph LR
    outside-->ingress
    subgraph Kubernetes
    ingress-->service
    service
    service-->app
    subgraph Pod
    app
    end
    end
{{< /mermaid >}}

Here's what it looks like with Keycloak

{{<mermaid align="left">}}
graph LR
    outside-->ingress
    subgraph Kubernetes
    ingress-->service
    service
    service-->gatekeeper
    subgraph Pod
    gatekeeper
    gatekeeper-->app
    end
    end
{{< /mermaid >}}

All traffic is gated via the gatekeeper, which ensures all requests are secure.
If the credentials are invalid or expired, then the user will be redirected to log back in via Keycloak.

If we look closer at the service, we'll see that the named `targetPort` points to the `gatekeeper` container in the pod:

```yaml
# app/templates/service.yaml
    - port: {{ .Values.service.port }}
      targetPort: gatekeeper # <--- LOOK HERE
      protocol: TCP
      name: http
```

## Next steps
- Secure secrets in encrypted form using something like [Sealed Secrets for Kubernetes](https://github.com/bitnami-labs/sealed-secrets)
- Make Postgres highly available by using the [postgresql-ha](https://artifacthub.io/packages/helm/bitnami/postgresql-ha) artifact
- Make your Keycloak instance highly available using [Clustering](https://www.keycloak.org/docs/latest/server_installation/#_clustering)
  - Worth considering if every client accessing every secured application in your cluster is accessing your Keycloak instance

[Keycloak]: https://www.keycloak.org/