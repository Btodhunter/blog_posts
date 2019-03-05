# Introducing the Anchore Engine Inline Scan!
#### TLDR;
```
curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -p alpine:latest
```

With anchore-engine, users can scan container images to generate reports against several aspects of the container image - vulnerability scans, content reports (files, OS packages, language packages, etc), fully customized policy evaluations (dockerfile checks, OSS license checks, software package checks, security checks, and many more). With these capabilities, users have integrated an anchore-engine image scan into CI/CD build processes for both reporting and/or control decision purposes, as anchore policy evaluations include a 'pass/fail' result alongside a full report upon policy execution.  

Up until now, the general setup required to achieve such an integration has included the requirement to stand up an anchore-engine service, with it's API exposed to your CI/CD build process, and make thin anchore API client calls from the build process to the centralized anchore-engine deployment.  Generally, the flow starts with an API call to 'add' an image to anchore-engine via an API call to the engine, at which point the engine will pull the referenced image from a docker v2 registry, and then perform report generation queries and/or policy evaluation calls.  This method is still fully supported, and in many cases is a good architecture for integrating anchore into your CI/CD platform. However, there are other use cases where the same result is desired (image scans, policy evaluations, content reports, etc), but for a variety of reasons it is impractical for the user to operate a centralized, managed and stable anchore-engine deployment that is available to CI/CD build processes.  

To accommodate these cases, we are introducing a new way to interact with anchore to get image scans, evaluations, and content reports without requiring a central anchore-engine deployment to be available.  We call this new approach 'inline scan', to indicate that a single, one-time scan can be performed 'inline' against a local container image at any time, without the need for any persistent data or service state between scans.  Using this approach (which ultimately uses exactly the same analysis/vulnerability/policy evaluation and reporting functions of anchore-engine), users can achieve an integration with anchore that moves the analysis/scanning work to a local container process that can be run during the container image build pipeline, after an image has been built but before it is pushed to any registry.  

With this new functionality, we hope to provide another approach for users to get deep analysis, scanning and policy evaluation capabilities of anchore in situations where operating an central anchore-engine service is impractical.

# Using the inline_scan script
To make using our inline-scan container as easy as possible, we have provided a simple wrapper script called inline_scan. The only requirements to run the inline_scan script is the ability to execute Docker commands & bash. We host a versioned copy of this script that can be downloaded directly with curl and executed in a bash pipeline, providing you image inspection, reporting and policy enforcement with one command.

To run the script on your workstation, use the following command syntax.
  
  ```
  curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- [options] IMAGE_NAME(s)
  ```

### inline_scan options
```
-b  [optional] Path to local Anchore policy bundle.
-d  [optional] Path to local Dockerfile.
-v  [optional] Path to directory to be mounted as docker volume. All image archives in directory will be scanned.
-f  [optional] Exit script upon failed Anchore policy evaluation.
-p  [optional] Pull remote docker images.
-r  [optional] Generate analysis reports in your current working directory.
-t  [optional] Specify timeout for image scanning in seconds (defaults to 300s).
```
## Examples

Pull multiple images from DockerHub, scan them all and generate individual reports in ./anchore-reports.
```sh
curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -p -r alpine:latest ubuntu:latest centos:latest
```

Perform a local docker build, then pass the Dockerfile to anchore inline scan. Use a custom policy bundle to ensure Dockerfile compliance, failing the script if anchore policy evaluation does not pass.
```sh
docker build -t example-image:latest -f Dockerfile .
curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -f -d Dockerfile -b .anchore-policy.json example-image:latest
```

Save multiple docker image archives to a directory, then mount the entire directory for analysis using a timeout of 500s.
```sh
cd example1/
docker build -t example1:latest .
cd ../example2
docker build -t example2:latest .
cd ..
mkdir images/
docker save example1:latest -o images/example1+latest.tar
docker save example2:latest -o images/example2+latest.tar
curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -v ./images -t 500
```

# Using anchore inline scan in your build pipeline

This same functionality can be utilized on any CI/CD platform that allows execution of Docker commands. The remainder of this post will be going over implementations of the anchore inline scan on a variety of popular CI/CD platforms.

All of the following examples can be found in this [repository](https://github.com/Btodhunter/ci-demos).

## CircleCI implementation
CircleCI version 2.0+ allows native docker command execution with the `setup_remote_docker` job step. By using this functionality combined with an official `docker:stable` image, we can build, scan, and push our images within the same job. We will also create reports and save them as artifacts within CircleCI. These reports are all created in json format, allowing easy aggregation from CircleCI into your preferred reporting tool.

This workflow requires the `DOCKER_USER` & `DOCKER_PASS` environment variables to be set in a context called `dockerhub` in your CircleCI account settings at `settings -> context -> create`

#### config.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/.circleci/config.yml)
```yaml
version: 2.1
jobs:
  build_scan_image:
    docker:
    - image: docker:stable
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
          curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -r "${IMAGE_NAME}:ci"
    - run:
        name: Push to DockerHub
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
GitLab allows docker command execution through a `docker:dind` service container. This job pushes the image to the GitLab registry, using built-in environment variables for specifying the image name and registry login credentials. To prevent premature timeouts, the timeout has been increased to 500s with the `-t` option. Reports are generated using the `-r` option, which are then passed as artifacts to be stored in GitLab. Even if you're not using an aggregation tool for artifacts, the json format allows reports to be parsed and displayed within GitLab pipeline using simple command line tools like jq.

#### .gitlab-ci.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/.gitlab-ci.yml)
```yaml
variables:
  IMAGE_NAME: ${CI_REGISTRY_IMAGE}/build:${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHA}

stages:
- build

container_build:
  stage: build
  image: docker:stable
  services:
  - docker:stable-dind

  variables:
    DOCKER_DRIVER: overlay2

  script:
  - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin "${CI_REGISTRY}"
  - docker build -t "$IMAGE_NAME" .
  - apk add bash curl 
  - curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -r -t 500 "$IMAGE_NAME"
  - docker push "$IMAGE_NAME"
  - |
      echo "Parsing anchore reports."
      for f in anchore-reports/*; do
        if [[ "$f" =~ "content-os" ]]; then
          printf "\n%s\n" "The following OS packages are installed on ${IMAGE_NAME}:"
          jq '[.content | sort_by(.package) | .[] | {package: .package, version: .version}]' $f || true
        fi
        if [[ "$f" =~ "vuln" ]]; then
          printf "\n%s\n" "The following vulnerabilities were found on ${IMAGE_NAME}:"
          jq '[.vulnerabilities | group_by(.package) | .[] | {package: .[0].package, vuln: [.[].vuln]}]' $f || true
        fi
      done

  artifacts:
    name: ${CI_JOB_NAME}-${CI_COMMIT_REF_NAME}
    paths:
    - anchore-reports/*
```

## CodeShip implementation
Docker command execution is enabled by default in CodeShip, which allows the inline_scan script to run on the `docker:stable` image without any additional configuration. By specifying the `-f` option on the inline_scan script, this job ensures that an image which fails it's anchore policy evaluation will not be pushed to the registry. To ensure adherence to the organization's security compliance policy, a custom policy bundle can be utilized for this scan by passing the `-b <POLICY_BUNDLE_FILE>` option to the inline_scan script.

This job requires creating an encrypted environment variable file for loading the `DOCKER_USER` & `DOCKER_PASS` variables into your job. See - [Encrypting CodeShip Environment Variables](https://documentation.codeship.com/pro/builds-and-configuration/environment-variables/#encrypted-environment-variables).

#### codeship-services.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/codeship-services.yml)
```yaml
anchore:
  add_docker: true
  image: docker:stable-git
  environment:
    IMAGE_NAME: btodhunter/anchore-ci-demo
    IMAGE_TAG: codeship
  encrypted_env_file: env.encrypted
```

#### codeship-steps.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/codeship-steps.yml)
```yaml
- name: build-scan
  service: anchore
  command: sh -c 'apk add bash curl &&
    mkdir -p /build && 
    cd /build &&
    git clone https://github.com/Btodhunter/ci-demos.git . &&
    docker build -t "${IMAGE_NAME}:ci" . &&
    curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -f -b .anchore_policy.json "${IMAGE_NAME}:ci" &&
    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin &&
    docker tag "${IMAGE_NAME}:ci" "${IMAGE_NAME}:${IMAGE_TAG}" &&
    docker push "${IMAGE_NAME}:${IMAGE_TAG}"'
```

## Jenkins pipeline implementation
Jenkins configured with the Docker, BlueOcean, and Pipeline plugins supports docker command execution using the `sh` directive. By using the `-d <PATH_TO_DOCKERFILE>` option with the inline_scan script, you can pass your Dockerfile to anchore-engine for policy evaluation. With the `-b <PATH_TO_POLICY_BUNDLE>` option, a custom policy bundle can be passed to the inline scan to ensure your Dockerfile conforms to best practices.

To allow pushing to a private registry, the `dockerhub-creds` credentials must be created in the Jenkins server settings at - `Jenkins -> Credentials -> System -> Global credentials -> Add Credentials`

This example was tested against the Jenkins installation detailed here, using the declarative pipeline syntax - [Jenkins Pipeline Docs](https://jenkins.io/doc/tutorials/build-a-multibranch-pipeline-project/#run-jenkins-in-docker)

#### Jenkinsfile - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/Jenkinsfile)
```groovy
pipeline{
    agent {
        docker {
            image 'docker:stable'
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
                sh 'curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -d Dockerfile -b .anchore_policy.json ${IMAGE_NAME}:ci'
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
TravisCI allows docker command execution by default, which makes integrating Anchore Engine as simple as adding the inline_scan script to your existing image build pipeline. This analysis should be performed before pushing the image to your registry of choice. 

The `DOCKER_USER` & `DOCKER_PASS` environment variables must be setup in the TravisCI console at `repository -> settings -> environment variables`

#### .travis.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/.travis.yml)
```yaml
language: node_js

services:
  - docker

env:
  - IMAGE_NAME="btodhunter/anchore-ci-demo" IMAGE_TAG="travisci"

script:
  - docker build -t "${IMAGE_NAME}:ci" .
  - curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- "${IMAGE_NAME}:ci"
  - echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
  - docker tag "${IMAGE_NAME}:ci" "${IMAGE_NAME}:${IMAGE_TAG}"
  - docker push "${IMAGE_NAME}:${IMAGE_TAG}"
```

## AWS CodeBuild implementation
AWS CodeBuild supports docker command execution by default. The Anchore inline_scan script can be inserted right into your pipeline before the image is pushed to it's registry.

The `DOCKER_USER`, `DOCKER_PASS`, `IMAGE_NAME`, & `IMAGE_TAG` environment variables must be set in the CodeBuild console at `Build Projects -> <PROJECT_NAME> -> Edit Environment -> Additional Config -> Environment Variables`

#### buildspec.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/buildspec.yml)
```yaml
version: 0.2

phases:
  build:
    commands:
      - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

  post_build:
    commands:
      - curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- ${IMAGE_NAME}:${IMAGE_TAG}
      - echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
      - docker push ${IMAGE_NAME}:${IMAGE_TAG}
```

# Summary
As you can see from the above examples, the new inline scan makes it easier then ever to implement Anchore Engine image analysis in your Docker build pipeline! You can scan local images before pushing them into a registry, allowing you to inject scans directly into your current workflows. The inline_scan script makes it simple to ensure your Dockerfile meets best practices, perform fine-grained custom policy evaluations, and even pull an image directly from a remote registry for scanning. Anchore inline scan is a zero friction solution for ensuring that only secure images make it through your build pipeline and get into production. Add it to your pipeline today!

Anchore Engine is an open source project, all issues and contribution details can be found on Github. We look forward to receiving feedback and contributions from our users!

## Links
[Github - Anchore Engine](https://github.com/anchore/anchore-engine)

[Github - Inline Scan](https://github.com/anchore/ci-tools)

[Github - CI Demos](https://github.com/Btodhunter/ci-demos)