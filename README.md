# Building a Websocker serverless Quarkus component

This repo resume files and instructions needed to execute a Demo of YAML Online system over Openshift.

## What else is needed to start?

* One Openshift Container Platform cluster (with at least 32Gi RAM and 8 CPUs).
* Access to the Openshift cluster as Cluster Admin.
* Installed in your MacOS or Linux workstation:
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

Then deploy a Sonatype Nexus to speed up dependency download:

```bash
oc new-project nexus --display-name="Nexus"
helm template nexus sonatype-nexus --repo=https://redhat-cop.github.io/helm-charts | oc apply -n nexus -f -
```

## Start demo

This demo is about deploy the whole YAML Online system that consist in 1 frontend component and 2 backend components:

* [YAML Online](https://github.com/caiomedeirospinto/yaml-online) Frontend with Angular 11.
* [YAML Online Microservice](https://github.com/caiomedeirospinto/yaml-ms-online-session) with Quarkus.
* [YAML Online Websocket](https://github.com/caiomedeirospinto/yaml-ws-online-session) with Quarkus.
* MariaDB as Database.

This Demo has 3 parts that must be run sequentially for it to work:

* First part: YAML Online Web Tool MVP.
* Second part: Blue/Green deployment new features.
* Lat part: Serverless Java backend with Quarkus.

Before to start check if Serverless Operator and Nexus are installed, then execute these steps:

> Any `<name>` must be replaced by its value.

## First part: YAML Online Web Tool MVP

1. Create a project for **YAML Online** system:

    ```bash
    oc login --token=<token> --server=<api-url>
    oc new-project yaml-online --display-name="YAML Online System"
    ```

2. Build, deploy, and configure frontend:

    ```bash
    # Generate image build 
    oc new-build nginx --name=frontend -l app=yaml-online \
      --binary=true -n=yaml-online -o yaml | oc apply -f -
    oc annotate bc/frontend app.openshift.io/vcs-uri=https://github.com/caiomedeirospinto/yaml-online.git -n yaml-online
    # Clone code and build artefact
    git clone https://github.com/caiomedeirospinto/yaml-online.git
    cd yaml-online
    git checkout 3ef6e77836ec1d1406352a60e93869edd09bdab4
    npm install
    npm run build
    # Configure web server
    cp nginx.conf ./dist/yaml-online/
    # Start image build
    oc patch bc/frontend -p '{"spec":{"output":{"to":{"kind":"ImageStreamTag","name":"frontend:v1"}}}}' -n yaml-online
    oc start-build frontend --from-dir=./dist/yaml-online/ --follow=true --wait=true -n=yaml-online
    # Deploy and configure
    kn service create --force frontend --revision-name v1 \
      --image image-registry.openshift-image-registry.svc:5000/yaml-online/frontend:v1 \
      --port 8080 --annotation=app.openshift.io/vcs-uri=https://github.com/caiomedeirospinto/yaml-online.git \
      --annotation=app.openshift.io/vcs-ref=3ef6e77836ec1d1406352a60e93869edd09bdab4 -l app=yaml-ws-online-session \
      -l app.kubernetes.io/part-of=yaml-online -n yaml-online
    ```

Now try it.

## Second part: Blue/Green deployment new features

The code is already developed, now you just need to deploy it:

```bash
git checkout d11224f1355ac98591f77ff7604632e157943ec1
npm install
git checkout master angular.json
npm run build
# Configure web server
cp nginx.conf ./dist/yaml-online/
# Start image build
oc patch bc/frontend -p '{"spec":{"output":{"to":{"kind":"ImageStreamTag","name":"frontend:v2"}}}}' -n yaml-online
oc start-build frontend --from-dir=./dist/yaml-online/ --follow=true --wait=true -n=yaml-online
# Deploy and configure
kn service update frontend --revision-name v2 \
  --image image-registry.openshift-image-registry.svc:5000/yaml-online/frontend:v2 \
  --annotation=app.openshift.io/vcs-ref=d11224f1355ac98591f77ff7604632e157943ec1 \
  -n yaml-online
kn service update frontend --traffic frontend-v2=100 -n yaml-online
```

Try it again, you'll find some new features. And if you want, you always can take back to version 1:

```bash
kn service update frontend --traffic frontend-v1=100 -n yaml-online
```

## Lat part: Serverless Java backend with Quarkus

> First check that your project quota o limitrange doesn't limit you to assign 4 CPU and 20Gi to a pod.

1. Deploy a MariaDB instance:

    ```bash
    oc new-app --name=mariadb --template=mariadb-persistent \
      -l=app=mariadb,app.kubernetes.io/part-of=yaml-online \
      -o yaml | oc apply -n yaml-online -f -
    # TODO: Create data structure
    ```

2. Create backend configuration files:

    ```bash
    for backend in ms ws; do
      curl https://raw.githubusercontent.com/caiomedeirospinto/yaml-$backend-online-session/master/src/main/resources/openshift.properties -o application.properties
      oc create configmap yaml-$backend-online-session \
        --from-file=application.properties=application.properties \
        -n yaml-online
      rm -rf application.properties
    done;
    ```

3. Build, deploy, and configure backends:

    ```bash
    # Prepare build and deploy
    oc tag quay.io/quarkus/ubi-quarkus-native-s2i:21.0-java11 yaml-online/ubi-quarkus-native-s2i:21.0-java11 -n yaml-online
    oc policy add-role-to-user view system:serviceaccount:yaml-online:default -n yaml-online
    # Build backend image and deploy
    # TODO: add -e MAVEN_MIRROR_URL="https://$(oc get route nexus -o jsonpath="{ .spec.host }" -n nexus)/repository/maven-public/"
    for backend in ms ws; do
      oc new-build --name=yaml-$backend-online-session \
        https://github.com/caiomedeirospinto/yaml-$backend-online-session.git \
        --docker-image=quay.io/quarkus/ubi-quarkus-native-s2i:21.0-java11 \
        -l=app=yaml-$backend-online-session,app.kubernetes.io/part-of=yaml-online \
        -o yaml | oc apply -n yaml-online -f -
      oc cancel-build bc/yaml-$backend-online-session -n yaml-online
      oc annotate bc/yaml-$backend-online-session \
        app.openshift.io/vcs-uri=https://github.com/caiomedeirospinto/yaml-$backend-online-session.git --overwrite
      oc patch bc/yaml-$backend-online-session \
        -p '{"spec":{"resources":{"limits":{"cpu":"4","memory":"20Gi"}}}}' -n yaml-online
      oc start-build bc/yaml-$backend-online-session --follow=true --wait=true -n yaml-online
      kn service create --force yaml-$backend-online-session --revision-name v1 \
        --image image-registry.openshift-image-registry.svc:5000/yaml-online/yaml-$backend-online-session \
        --port 8080 -e QUARKUS_PROFILE=openshift --env-from secret:mariadb -l app=yaml-$backend-online-session \
        --annotation=app.openshift.io/vcs-uri=https://github.com/caiomedeirospinto/yaml-$backend-online-session.git \
        --annotation=app.openshift.io/vcs-ref=master -l app.kubernetes.io/part-of=yaml-online -n yaml-online
    done;
    ```

4. Create frontend configuration:

    ```bash
    # Get base file
    curl https://raw.githubusercontent.com/caiomedeirospinto/yaml-online/master/src/assets/config/config.json -o config.json
    # Set backend values
    cat config.json | jq '.onlineSession.enabled = true' | \
      jq '.onlineSession.backends.apiRest = $apiRest' --arg apiRest "$(oc get route.serving.knative.dev yaml-ms-online-session -o jsonpath="{ .status.url }" -n yaml-online)/online-session" | \
      jq '.onlineSession.backends.ws = $ws' --arg ws "${$(oc get route.serving.knative.dev yaml-ws-online-session -o jsonpath="{ .status.url }" -n yaml-online)/http/ws}/online-session" > config.json
    oc create configmap frontend-config --from-file=config.json=config.json -n yaml-online
    rm -rf config.json
    ```

5. Build, deploy, and configure frontend:

    ```bash
    git checkout master
    npm install
    npm run build
    # Configure web server
    cp nginx.conf ./dist/yaml-online/
    # Start image build
    oc patch bc/frontend -p '{"spec":{"output":{"to":{"kind":"ImageStreamTag","name":"frontend:v3"}}}}' -n yaml-online
    oc start-build frontend --from-dir=./dist/yaml-online/ --follow=true --wait=true -n=yaml-online
    # Deploy and configure
    kn service update frontend --revision-name v3 \
      --image image-registry.openshift-image-registry.svc:5000/yaml-online/frontend:v3 \
      --annotation=app.openshift.io/vcs-ref=master \
      --mount /opt/app-root/src/assets/config=cm:frontend-config -n yaml-online
    kn service update frontend --traffic frontend-v3=100 -n yaml-online
    ```

Now try it and check if websocket is running well, if it's not just take back the frontend version:

```bash
kn service update frontend --traffic frontend-v2=100 -n yaml-online
```

## Same process but automated

WIP.
