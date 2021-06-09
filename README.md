# Building a Websocker serverless Quarkus component

This repo resume files and instructions needed to execute a Demo of YAML Online system over Openshift.

## What else is needed to start?

* One Openshift Container Platform cluster,
* Access to the cluster as Cluster Admin.
* Helm CLI installed in your workstation.
* Knative CLI installed in your workstation.

## Preparing Openshift cluster

The following steps will help you to prepare your Openshift environment:

```bash
oc login --token=<token> --server=<api-url>
cd ./prepare
helm template -f prepare/values.yml prepare | oc apply -f -
```

## Start demo

This demo is about deploy the whole YAML Online system that consist in 1 frontend component and 2 backend components:

* [YAML Online](https://github.com/caiomedeirospinto/yaml-online) Frontend with Angular 11.
* [YAML Online Microservice](https://github.com/caiomedeirospinto/yaml-ms-online-session) with Quarkus.
* [YAML Online Websocket](https://github.com/caiomedeirospinto/yaml-ws-online-session) with Quarkus.
* MariaDB as Database.

Below both process to deploy it on Openshift: **Manual** and **Automated**.

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
    oc new-app --name=mariadb --template=mariadb-persistent -l=app=mariadb,app.kubernetes.io/part-of=yaml-online -o yaml | oc apply -n yaml-online -f -
    ```

4. Register credentials to fetch repository:

    ```bash
    oc create secret generic github-credentials --from-literal=username=<your-username> --from-literal=password=<your-password-or-token> -o yaml | oc apply -n yaml-online -f -
    ```

5. Create backend configuration files:

    ```bash
    oc create configmap generic github-credentials --from-literal=username=<your-username> --from-literal=password=<your-password-or-token> -o yaml | oc apply -n yaml-online -f -
    ```

6. Build, deploy, and configure backends:

    ```bash
    # Build backend images
    oc new-build --name=yaml-ms-online-session https://github.com/<your-username>/yaml-ms-online-session.git --source-image=quay.io/quarkus/ubi-quarkus-native-s2i:20.3-java11 -l=app=yaml-ms-online-session,app.kubernetes.io/part-of=yaml-online --source-secret=github-credentials -o yaml | oc apply -n yaml-online -f -
    oc new-build --name=yaml-ms-online-session https://github.com/<your-username>/yaml-ms-online-session.git --source-image=quay.io/quarkus/ubi-quarkus-native-s2i:20.3-java11 -l=app=yaml-ms-online-session,app.kubernetes.io/part-of=yaml-online --source-secret=github-credentials -o yaml | oc apply -n yaml-online -f -
    # Deploy and configure
    kn service create --force yaml-ms-online-session --image image-registry.openshift-image-registry.svc:5000/yaml-online/yaml-ms-online-session --port 8080 --env-from secret:mariadb -n yaml-online 
    kn service create --force yaml-ms-online-session --image image-registry.openshift-image-registry.svc:5000/yaml-online/yaml-ms-online-session --port 8080 -e= --env-from secret:mariadb -n yaml-online 
    ```