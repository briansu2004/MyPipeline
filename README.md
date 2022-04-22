# MyPipeline

My Pipeline

## GCP Cloud Build

cloudbuild.yaml + Dockerfile

### Deploy a Java app to GKE

cloudbuild.yaml

```
steps:
  # Build the maven project, it will create the default multi-regional bucket ${PORJECT_ID}_cloudbuild
  - name: 'maven:3.6.2-jdk-8-openj9'
    id: ‘build-step’
    entrypoint: 'mvn'
    args: ['clean','install','package']
    dir: ${_DIR}

  # Build image
  - name: 'gcr.io/cloud-builders/docker'
    waitFor: [‘build-step’]
    id: 'build-image'
    args: ['build', '-t', 'gcr.io/${_REPO_ID}/${_NAMESPACE}/${_SERVICE_NAME}:$SHORT_SHA', '.']
    dir: ${_DIR}

  # tag app image with latest
  - name: 'gcr.io/cloud-builders/docker'
    id: 'tag-image-with-latest'
    args: ['tag', 'gcr.io/${_REPO_ID}/${_NAMESPACE}/${_SERVICE_NAME}:$SHORT_SHA', 'gcr.io/${_REPO_ID}/${_NAMESPACE}/${_SERVICE_NAME}:$TAG_NAME']
    dir: ${_DIR}

  # Copy Helm chart to Cloud Storage
  - name: 'gcr.io/cloud-builders/gsutil'
    id: 'publish-helm-chart'
    args: [ 'cp','-Z','helm/*', 'gs://${_REPO_ID}-kubernetes-manifests/${_NAMESPACE}/']
    dir: ${_DIR}

images:
  - 'gcr.io/${_REPO_ID}/${_NAMESPACE}/${_SERVICE_NAME}:$SHORT_SHA'
  - 'gcr.io/${_REPO_ID}/${_NAMESPACE}/${_SERVICE_NAME}:$TAG_NAME'

substitutions:
  _SERVICE_NAME: my-app
  _REPO_ID: my-gcp-project
  _NAMESPACE: my-gcp-namespace
  _DIR: my-folder
```

Dockerfile

```
# Start with a base image containing Java runtime
FROM openjdk:8-jre-slim
# The application's jar file
ARG JAR_FILE=target/*.jar
#JAR_FILE_MUST_BE_SPECIFIED_AS_BUILD_ARG

COPY src/main/resources/certs/btwp013979.corp.ads_TCSO-issuing-CA.crt $JAVA_HOME/lib/security
COPY src/main/resources/certs/btwp013981_TCSO-root-CA.crt $JAVA_HOME/lib/security
RUN keytool -import -trustcacerts -noprompt -storepass changeit -alias TCSO-issuing-CA -file  $JAVA_HOME/lib/security/btwp013979.corp.ads_TCSO-issuing-CA.crt -keystore $JAVA_HOME/lib/security/cacerts
RUN keytool -import -trustcacerts -noprompt -storepass changeit -alias TCSO-root-CA -file  $JAVA_HOME/lib/security/btwp013981_TCSO-root-CA.crt -keystore $JAVA_HOME/lib/security/cacerts

COPY ${JAR_FILE} app.jar
RUN apt-get update && \
    apt-get install -y curl --no-install-recommends && \
    apt-get install -y iputils-ping

ENTRYPOINT ["java", "-Djava.security.edg=file:/dev/./urandom","-jar","/app.jar"]
```

### Deploy a NodeJS app to GKE

cloudbuild.yaml

```
steps:
  # install node.js
  - name: 'gcr.io/cloud-builders/npm'
    id: 'install nodejs'
    args: ['install']
    dir: ${_BASE_DIR}

  # run unit tests
  - name: 'gcr.io/cloud-builders/npm'
    id: 'Unit Tests'
    args: ['run', 'unit']
    dir: ${_BASE_DIR}

  # start the docker build
  - name: 'gcr.io/cloud-builders/docker'
    id: 'Build Image'
    args: ['build', '--tag=gcr.io/${_REPO_ID}/${_NAMESPACE}/${_APP}:$TAG_NAME', '.']
    dir: ${_BASE_DIR}

  - name: 'gcr.io/cloud-builders/gsutil'
    id: 'Copy Helm Files to Cloud Storage'
    args: [ 'cp','-Z','helm/*', 'gs://${_REPO_ID}-kubernetes-manifests/${_NAMESPACE}/']
    dir: ${_BASE_DIR}

images: ['gcr.io/${_REPO_ID}/${_NAMESPACE}/${_APP}:$TAG_NAME']

substitutions:
  _APP: my-app
  _NAMESPACE: my-gcp-namespace
  _BASE_DIR: my-folder
  _REPO_ID:  my-gcp-project
# timeout: 600s
```

Dockerfile

```
FROM node:lts-alpine

WORKDIR /app
RUN set -ex && \
  adduser node root && \
  apk add --update --no-cache \
  curl

COPY . .

RUN npm install

USER node
EXPOSE 3001

ENTRYPOINT ["npm"]
CMD ["run", "start"]
```
