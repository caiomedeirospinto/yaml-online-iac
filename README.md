# Building a Websocker serverless Quarkus component

This repo resume files and instructions needed to execute a Demo of YAML Online system over Openshift.

## What else is needed to start?

* One Openshift Container Platform cluster.
* Access to the Openshift cluster as Cluster Admin.
* Installed in your workstation:
  * Openshift CLI version 4.7 or later
  * Helm CLI version 3.2.0 or later.
  * Knative CLI version 0.20.0 or later.
  * NodeJS version Fermium or later.
  * JQ version 1.6 or later.

## Preparing Openshift cluster

The following steps will help you to prepare your Openshift environment:

```bash
oc login --token=<token> --server=<api-url>
helm template -f prepare/values.yaml prepare | oc apply -f -
```

> It creates a Knative Serving that needs Serverless Operator installed, 
> so during first execution will fail Knative Serving creation. Just wait 
> for 2 or 3 minutes and execute again.

## Start demo

This demo is about deploy the whole YAML Online system that consist in 1 frontend component and 2 backend components:

* [YAML Online](https://github.com/caiomedeirospinto/yaml-online) Frontend with Angular 11.
* [YAML Online Microservice](https://github.com/caiomedeirospinto/yaml-ms-online-session) with Quarkus.
* [YAML Online Websocket](https://github.com/caiomedeirospinto/yaml-ws-online-session) with Quarkus.
* MariaDB as Database.

Below both process to deploy it on Openshift: **Manual** and **Automated** (WIP).

## Manual Process

Before to start make a fork of each component repository, then execute these steps:

> Any `<name>` must be replaced by its value.

1. Login into Openshift:

    ```bash
    oc login --token=<token> --server=<api-url>
    ```

2. Create a project for **YAML Online** system:

    ```bash
    oc new-project yaml-online --display-name="YAML Online System"
    ```

3. Deploy a MariaDB instance:

    ```bash
    oc new-app --name=mariadb --template=mariadb-persistent \
      -l=app=mariadb,app.kubernetes.io/part-of=yaml-online \
      -o yaml | oc apply -n yaml-online -f -
    ```

4. Register credentials to fetch repository:

    ```bash
    oc create secret generic github-credentials \
      --from-literal=username=<your-username> \
      --from-literal=password=<your-password-or-token> \
      -o yaml | oc apply -n yaml-online -f -
    ```

5. Create backend configuration files:

    ```bash
    # Microservice
    curl https://raw.githubusercontent.com/<your-username>/yaml-ms-online-session/master/src/main/resources/openshift.properties -o application.properties
    oc create configmap yaml-ms-online-session \
      --from-file=application.properties=application.properties \
      -o yaml | oc apply -n yaml-online -f -
    rm -rf application.properties
    # Websocket
    curl https://raw.githubusercontent.com/<your-username>/yaml-ws-online-session/master/src/main/resources/openshift.properties -o application.properties
    oc create configmap yaml-ws-online-session \
      --from-file=application.properties=application.properties \
      -o yaml | oc apply -n yaml-online -f -
    rm -rf application.properties
    ```

6. Build, deploy, and configure backends:

    ```bash
    # Build backend images
    oc new-build --name=yaml-ms-online-session \
      https://github.com/<your-username>/yaml-ms-online-session.git \
      --docker-image=quay.io/quarkus/ubi-quarkus-native-s2i:20.3-java11 \
      -l=app=yaml-ms-online-session,app.kubernetes.io/part-of=yaml-online \
      --source-secret=github-credentials -o yaml | oc apply -n yaml-online -f -
    oc cancel-build bc/yaml-ms-online-session
    oc start-build bc/yaml-ms-online-session --follow=true --wait=true
    oc new-build --name=yaml-ws-online-session \
      https://github.com/<your-username>/yaml-ws-online-session.git \
      --docker-image=quay.io/quarkus/ubi-quarkus-native-s2i:20.3-java11 \
      -l=app=yaml-ws-online-session,app.kubernetes.io/part-of=yaml-online \
      --source-secret=github-credentials -o yaml | oc apply -n yaml-online -f -
    oc cancel-build bc/yaml-ws-online-session
    oc start-build bc/yaml-ws-online-session --follow=true --wait=true
    # Deploy and configure
    kn service create --force yaml-ms-online-session \
      --image image-registry.openshift-image-registry.svc:5000/yaml-online/yaml-ms-online-session \
      --port 8080 --env-from secret:mariadb -l app=yaml-ws-online-session \
      -l app.kubernetes.io/part-of=yaml-online -n yaml-online 
    kn service create --force yaml-ws-online-session \
      --image image-registry.openshift-image-registry.svc:5000/yaml-online/yaml-ws-online-session \
      --port 8080 --env-from secret:mariadb -l app=yaml-ws-online-session \
      -l app.kubernetes.io/part-of=yaml-online -n yaml-online
    ```

7. Create frontend configuration:

    ```bash
    # Get base file
    curl https://raw.githubusercontent.com/<your-username>/yaml-online/master/src/assets/config/config.json -o=config.json
    # Set backend values
    cat config.json | \
      jq '.onlineSession.backends.apiRest = $apiRest' --arg apiRest "$(oc get route.serving.knative.dev yaml-ms-online-session -o jsonpath="{ .status.url }")" | \
      jq '.onlineSession.backends.ws = $ws' --arg ws "$(oc get route.serving.knative.dev yaml-ws-online-session -o jsonpath="{ .status.url }")" > config.json
    oc create configmap frontend-config --from-file=config.json=config.json
    rm -rf config.json
    ```

8. Build, deploy, and configure frontend:

    ```bash
    # Generate image build 
    oc new-build nginx --name=frontend -l app=yaml-online \
      --binary=true -n=yaml-online -o yaml | oc apply -f -
    # Clone code and build artefact
    git clone https://<your-username>:<your-password-or-token>@github.com/<your-username>/yaml-online.git
    cd yaml-online
    npm install
    npm run build
    # Configure web server
    cp nginx.conf ./dist/yaml-online/
    # Start image build
    oc start-build frontend --from-dir=./dist/yaml-online/ -n=yaml-online
    # Deploy and configure
    kn service create --force frontend \
      --image image-registry.openshift-image-registry.svc:5000/yaml-online/frontend \
      --port 8080 --volume config=cm:frontend-config -n yaml-online 
    ```

9. Expose front app:

    ```bash
    oc expose svc/frontend --port 8080
    ```

## Automated Process

WIP
