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

### Deploy a Python app to GKE

cloudbuild.yaml

```
steps:
  # Install dependencies
  - name: python
    entrypoint: pip
    args: ["install", "-r", "requirements.txt", "--user"]

  # Run unit tests
  - name: python
    entrypoint: python
    args: ["-m", "pytest", "--junitxml=${SHORT_SHA}_test_log.xml"]

  # Docker Build
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t',
           'us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/myimage:${SHORT_SHA}', '.']

  # Docker push to Google Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push',  'us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/myimage:${SHORT_SHA}']

  # Deploy to Cloud Run
  - name: google/cloud-sdk
    args: ['gcloud', 'run', 'deploy', 'helloworld-${SHORT_SHA}',
           '--image=us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/myimage:${SHORT_SHA}',
           '--region', 'us-central1', '--platform', 'managed',
           '--allow-unauthenticated']

# Save test logs to Google Cloud Storage
artifacts:
  objects:
    location: gs://${_BUCKET_NAME}/
    paths:
      - ${SHORT_SHA}_test_log.xml
# Store images in Google Artifact Registry
images:
  - us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/myimage:${SHORT_SHA}
```

Dockerfile

```
# Use the official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:3.10-slim

# Allow statements and log messages to immediately appear in the Knative logs
ENV PYTHONUNBUFFERED True

# Copy local code to the container image.
ENV APP_HOME /app
WORKDIR $APP_HOME
COPY . ./

# Install production dependencies.
RUN pip install Flask gunicorn

# Run the web service on container startup. Here we use the gunicorn
# webserver, with one worker process and 8 threads.
# For environments with multiple CPU cores, increase the number of workers
# to be equal to the cores available.
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 main:app
```

requirements.txt

```
Flask==2.1.1
pytest==7.1.1
```

### Deploy another Python app (connect to a Oracle DB) to GKE

oracle-connectivity-test.py

```
import cx_Oracle

try:
    cx_Oracle.init_oracle_client()
    conn = cx_Oracle.connect("username/password@//server:port/service")
    cur = conn.cursor()
    cur.execute("SELECT 'Oracle connected!' FROM dual")
    res = cur.fetchall()
    print(res)
except Exception as err:
    print("Error connecting: cx_Oracle.init_oracle_client()")
    print(err)
```

Dockerfile

```

# Use the official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:3.10-slim

# Allow statements and log messages to immediately appear in the Knative logs
ENV PYTHONUNBUFFERED True

# Copy local code to the container image.
#ENV APP_HOME /app
#WORKDIR $APP_HOME
#COPY . ./
WORKDIR /app
COPY ./Folder1/Folder2/oracle-connectivity-test.py /app

RUN ls -l /app

# Install production dependencies.
#RUN sudo yum install libaio
#RUN sudo yum install oracle-instantclient-basic-21.1.0.0.0-1.x86_64.rpm
#RUN pip install cx-Oracle

RUN apt-get update && apt-get install -y libaio1 wget unzip

WORKDIR /opt/oracle
RUN wget https://download.oracle.com/otn_software/linux/instantclient/instantclient-basiclite-linuxx64.zip && \
    unzip instantclient-basiclite-linuxx64.zip && rm -f instantclient-basiclite-linuxx64.zip && \
    cd /opt/oracle/instantclient* && rm -f *jdbc* *occi* *mysql* *README *jar uidrvci genezi adrci && \
    echo /opt/oracle/instantclient* > /etc/ld.so.conf.d/oracle-instantclient.conf && ldconfig
RUN python -m pip install cx_Oracle

# Run the app.
CMD [ "python", "/app/oracle-connectivity-test.py" ]
```

cloudbuild.yaml

```
# [START cloudbuild.yaml]
steps:
  # # [START cloudbuild_python_dependencies_yaml]
  # # Install dependencies
  # - name: python
  #   entrypoint: pip
  #   args: ["install", "-r", "requirements.txt", "--user"]
  # # [END cloudbuild_python_dependencies_yaml]

  # # [START cloudbuild_python_tests_yaml]
  # # Run unit tests
  # - name: python
  #   entrypoint: python
  #   args: ["-m", "pytest", "--junitxml=${SHORT_SHA}_test_log.xml"]
  # # [END cloudbuild_python_tests_yaml]

  # [START cloudbuild_python_image_yaml]
  # - name: 'ubuntu'
  #   args: [ "ls", "-l" ]

  # - name: 'ubuntu'
  #   args: [ "ls", "-l", "/" ]

  # - name: 'ubuntu'
  #   args: [ "ls", "-l", "/workspace" ]

  # Docker Build
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build',
           '--file', '/workspace/folder1/folder2/mydockerfile',
           '-t', 'us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/oracle-connectivity-test:${SHORT_SHA}',
           '.']
  # [END cloudbuild_python_image_yaml]

  # [START cloudbuild_python_push_yaml]
  # Docker push to Google Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push',  'us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/oracle-connectivity-test:${SHORT_SHA}']
  # [END cloudbuild_python_push_yaml]

  # [START cloudbuild_python_deploy_yaml]
  # Deploy to Cloud Run
  - name: google/cloud-sdk
    args: ['gcloud', 'run', 'deploy', 'oracle-connectivity-test-${SHORT_SHA}',
           '--image=us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/oracle-connectivity-test:${SHORT_SHA}',
           '--region', 'us-central1', '--platform', 'managed',
           '--allow-unauthenticated']
  # [END cloudbuild_python_deploy_yaml]

# # [START cloudbuild_python_logs_yaml]
# # Save test logs to Google Cloud Storage
# artifacts:
#   objects:
#     location: gs://${_BUCKET_NAME}/
#     paths:
#       - ${SHORT_SHA}_test_log.xml
# # [END cloudbuild_python_logs_yaml]

# Store images in Google Artifact Registry
images:
  - us-central1-docker.pkg.dev/${PROJECT_ID}/${_ARTIFACT_REGISTRY_REPO}/oracle-connectivity-test:${SHORT_SHA}

substitutions:
  _ARTIFACT_REGISTRY_REPO: cloud-run-source-deploy
  _BUCKET_NAME: mybucket
# [END cloudbuild.yaml]
```
