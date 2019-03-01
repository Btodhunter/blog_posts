# Introducting the Anchore Engine inline scan!


# Using the Anchore Engine inline scan script
We have included a wrapper script for easily interacting with our inline-scan container. All that is required on your system to use this script is Docker & BASH.

To run the script on your workstation, run it directly from github using the following command.
  
  ```
  curl -s https://raw.githubusercontent.com/anchore/ci-tools/scripts/inline_scan | bash -s -- [options] <IMAGE_NAME>
  ```

## Examples for utilizing the inline_scan script

Pull multiple images from dockerhub, scan and generate reports.
```bash
curl -s https://raw.githubusercontent.com/anchore/ci-tools/scripts/inline_scan | bash -s -- -p -r alpine:latest ubuntu:latest centos:latest
```

Pass Dockerfile to image scan, after docker build.
```bash
docker build -t example-image:latest -f ./Dockerfile .
curl ... | bash -s -- -d ./Dockerfile example-image:latest
```

Scan image using custom policy bundle, fail script if policy evaluation doesn't pass.
```bash
curl ... | bash -s -- -f -p ./policy-bundle.json example-image:latest
```

Save docker image archives to a directory. Then mount entire directory for analysis, custom timeout set to 500s.
```bash
mkdir images
docker save example-image:latest -o images/example-image+latest.tar
docker save example-image:dev -o images/example-image+dev.tar
curl ... | bash -s -- -v ./images -t 500
```

# Using the inline_scan script in your CI/CD Pipeline

This same fundamental concept can be utilized on any CI/CD tool that allows access to the docker daemon. Here are some examples of using the new Anchore Engine inline scan on many popular CI/CD tools.

All the following examples can be found in this repository - https://github.com/Btodhunter/ci-demos

## CircleCI implementation
CircleCI version 2.0+ allows access to the docker daemon with the `setup_remote_docker` job step. By using this functionality, combined with an official docker:stable image, we can build, scan, and push our images within the same job. This workflow requires the following environment variables - `DOCKER_USER, DOCKER_PASS` - to be set in a context called `dockerhub` in your CircleCI account settings at `settings -> context -> create`

#### .circleci/config.yml
```yaml
version: 2.1
jobs:
  build_scan_image:
    docker:
    - image: docker:stable-git
    environment:
      IMAGE_NAME: btodhunter/anchore-ci-demo
      IMAGE_TAG: circleci
    steps:
    - checkout
    - setup_remote_docker
    - run:
        name: Build image
        command: docker build -t "${IMAGE_NAME}:ci" .
    - run:
        name: Scan image
        command: |
          apk add curl bash
          curl -s https://raw.githubusercontent.com/anchore/ci-tools/scripts/inline_scan | bash -s -- -r "${IMAGE_NAME}:ci"
    - run:
        name: Push to Dockerhub
        command: |
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
          docker tag "${IMAGE_NAME}:ci" "${IMAGE_NAME}:${IMAGE_TAG}"
          docker push "${IMAGE_NAME}:${IMAGE_TAG}"
    - store_artifacts:
        path: anchore-reports/
  
workflows:
  scan_image:
    jobs:
    - build_scan_image:
        context: dockerhub
```

## GitLab implementation
#### .gitlab-ci.yml
```yaml
variables:
  IMAGE_NAME: ${CI_REGISTRY_IMAGE}/build:${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHA}

stages:
- build

container_build:
  stage: build
  image: docker:stable-git
  services:
  - docker:stable-dind

  variables:
    DOCKER_DRIVER: overlay2

  script:
  - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin "${CI_REGISTRY}"
  - docker build -t "$IMAGE_NAME" .
  - apk add bash curl 
  - curl -s "https://raw.githubusercontent.com/anchore/ci-tools/scripts/inline_scan" | bash -s -- -f "$IMAGE_NAME"
  - docker push "$IMAGE_NAME"

  artifacts:
    name: ${CI_JOB_NAME}-${CI_COMMIT_REF_NAME}
    paths:
    - anchore-reports/*
```

## Codeship implementation
#### codeship-services.yml
```yaml
dind:
  add_docker: true
  image: docker:stable-git
  environment:
    IMAGE_NAME: btodhunter/anchore-ci-demo
    IMAGE_TAG: codeship
  encrypted_env_file: env.encrypted
```

#### codeship-steps.yml
```yaml
- name: build-scan
  service: dind
  command: sh -c 'apk add bash curl &&
    mkdir -p /build && 
    cd /build &&
    git clone https://github.com/Btodhunter/ci-demos.git . &&
    docker build -t "${IMAGE_NAME}:ci" . &&
    curl -s "https://raw.githubusercontent.com/anchore/ci-tools/scripts/inline_scan" | bash -s -- -r  "${IMAGE_NAME}:ci" &&
    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin &&
    docker tag "${IMAGE_NAME}:ci" "${IMAGE_NAME}:${IMAGE_TAG}" &&
    docker push "${IMAGE_NAME}:${IMAGE_TAG}"'
```

## Jenkins pipeline implementation
#### Jenkinsfile
```groovy
pipeline{
    agent {
        docker {
            image 'docker:stable-git'
        }
    }
    environment {
        IMAGE_NAME = 'btodhunter/anchore-ci-demo'
        IMAGE_TAG = 'jenkins'
    }
    stages {
        stage('Build Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:ci .'
            }
        }
        stage('Scan') {
            steps {        
                sh 'apk add bash curl'
                sh 'curl -s https://raw.githubusercontent.com/anchore/ci-tools/master/scripts/inline_scan | bash -s -- ${IMAGE_NAME}:ci'
            }
        }
        stage('Push Image') {
            steps {
                withDockerRegistry([credentialsId: "dockerhub-creds", url: ""]){
                    sh 'docker tag ${IMAGE_NAME}:ci ${IMAGE_NAME}:${IMAGE_TAG}'
                    sh 'docker push ${IMAGE_NAME}:${IMAGE_TAG}'
                }
            }
        }
    }
}
```

## TravisCI implementation
#### .travis.yml
```yaml
language: node_js

services:
  - docker

env:
  - IMAGE_NAME="btodhunter/anchore-ci-demo" IMAGE_TAG="travisci"

before_install:
  - docker build -t "${IMAGE_NAME}:ci" .
  - curl -s "https://raw.githubusercontent.com/anchore/ci-tools/scripts/inline_scan" | bash -s -- "${IMAGE_NAME}:ci"
  - echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
  - docker tag "${IMAGE_NAME}:ci" "${IMAGE_NAME}:${IMAGE_TAG}"
  - docker push "${IMAGE_NAME}:${IMAGE_TAG}"
```

## AWS CodeBuild implementation
#### buildspec.yml
```yaml
version: 0.2

phases:
  build:
    commands:
      - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

  post_build:
    commands:
      - curl -s https://raw.githubusercontent.com/anchore/ci-tools/master/scripts/inline_scan | bash -s -- -f ${IMAGE_NAME}:${IMAGE_TAG}
      - echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
      - docker push ${IMAGE_NAME}:${IMAGE_TAG}
```

# Running the Anchore inline-scan container as a service
For CI/CD tools that don't support access to the docker daemon, the anchore/inline-scan container can be stood up as a service within your pipeline infrastructure. This allows interaction with anchore engine as though it was running on premisis, but without the need to manage any of the infrastructure. On certain platforms (such as GitLab) this method can offer better performance over using the inline_scan script.

## Codefresh implementation
For this job to run successfully, the following environment variables must be setup - `IMAGE_NAME, IMAGE_TAG, DOCKER_USER, DOCKER_PASS`. These variables can be set in the Codefresh console at `settings -> pipelines -> environment variables`. This method also requires setting up a Codefresh image registry token at `user settings -> codefresh registry -> generate`, this token can then be used in the `DOCKER_PASS` variable to give anchore engine pull permissions.

#### codefresh.yml
```yaml
version: '1.0'
steps:
  build_docker_image:
    title: Building Docker Image
    type: build
    image_name: ${{IMAGE_NAME}}
    tag: ci
    working_directory: ./
    dockerfile: Dockerfile

  anchore_scan:
    type: composition
    title: Scanning with Anchore Engine
    composition:
      version: '2'
      services:
        anchore-engine:
          image: anchore/inline-scan:latest
          ports:
            - 8228
          command: bash -c 'docker-entrypoint.sh start &> /dev/null'
    composition_candidates:
      scan_image:
        image: anchore/inline-scan:latest
        links:
          - anchore-engine
        environment:
          - ANCHORE_CLI_URL=http://anchore-engine:8228/v1/
          - IMAGE_NAME=${{IMAGE_NAME}}
          - IMAGE_TAG=${{IMAGE_TAG}}
          - DOCKER_USER=${{DOCKER_USER}}
          - DOCKER_PASS=${{DOCKER_PASS}}
        command: bash -xc 'anchore-cli system wait && anchore-cli registry add r.cfcr.io $DOCKER_USER $DOCKER_PASS --skip-validate && anchore_ci_tools.py -a -r --image r.cfcr.io/btodhunter/${IMAGE_NAME}:ci'
    composition_variables:
      - IMAGE_NAME=${{IMAGE_NAME}}
      - IMAGE_TAG=${{IMAGE_TAG}}
      - DOCKER_USER=${{DOCKER_USER}}
      - DOCKER_PASS=${{DOCKER_PASS}}

  push_to_registry:
    title: Pushing to Docker Registry 
    type: push
    candidate: '${{build_docker_image}}'
    tag: ${{IMAGE_TAG}}
    registry: dockerhub
```

## GitLab implementation
#### .gitlab-ci.yml
```yaml
variables:
  IMAGE_NAME: ${CI_REGISTRY_IMAGE}/build:${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHA}

stages:
- build
- scan
- publish

container_build:
  stage: build
  image: docker:stable-git
  services:
  - docker:stable-dind

  variables:
    DOCKER_DRIVER: overlay2

  script:
  - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin "${CI_REGISTRY}"
  - docker build -t "$IMAGE_NAME" .

container_scan_service:
  stage: scan
  variables:
    ANCHORE_CLI_URL: "http://anchore-engine:8228/v1"
    GIT_STRATEGY: none
  image: docker.io/anchore/inline-scan:latest
  services:
  - name: docker.io/anchore/inline-scan:latest
    alias: anchore-engine
    command: ["start"]
  
  script:
  - anchore-cli system wait
  - anchore-cli --u admin --p foobar registry add "$CI_REGISTRY" gitlab-ci-token "$CI_JOB_TOKEN" --skip-validate 
  - anchore_ci_tools.py -a -r --image $IMAGE_NAME

  artifacts:
    name: ${CI_JOB_NAME}-${CI_COMMIT_REF_NAME}
    paths:
    - anchore-reports/*

container_publish:
  stage: publish
  image: docker:stable
  services:
  - docker:stable-dind

  variables:
    DOCKER_DRIVER: overlay2
    GIT_STRATEGY: none

  script:
  - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin "${CI_REGISTRY}"
  - docker pull "$IMAGE_NAME"
  - docker tag "$IMAGE_NAME" "${CI_REGISTRY_IMAGE}:latest"
  - docker push "${CI_REGISTRY_IMAGE}:latest"
```