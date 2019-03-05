# Running the Anchore inline-scan container as a service
For CI/CD tools that do not allow docker command execution, the anchore/inline-scan container can be utilized as a service within your pipeline infrastructure. This allows interaction with anchore engine as though it was running on premises, but without the need to manage any of the infrastructure. On certain platforms (such as GitLab) this method can offer better performance over using the inline_scan wrapper script. 

## CodeFresh implementation
This job requires that the following environment variables are set - `IMAGE_NAME, IMAGE_TAG, DOCKER_USER, DOCKER_PASS`. These variables can be setup in the CodeFresh console at `settings -> pipelines -> environment variables`. This method also requires creating a CodeFresh image registry token at `user settings -> codefresh registry -> generate`, this token can then be used in the `DOCKER_PASS` variable, which is utilized to give anchore engine pull privileges to the CodeFresh registry.

#### codefresh.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/codefresh.yml)
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
        command: bash -xc 'anchore-cli system wait && anchore-cli registry add r.cfcr.io $DOCKER_USER $DOCKER_PASS --skip-validate && anchore_ci_tools.py -a --timeout 500 --image r.cfcr.io/btodhunter/${IMAGE_NAME}:ci'
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
To speed up your image analysis time, you can setup the anchore/inline-scan on GitLab to run as a service. This also allows you to run other tests with the image in parallel with the image scan.

#### .gitlab-ci.yml - [Github Link](https://github.com/Btodhunter/ci-demos/blob/master/.gitlab-ci.yml)
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
  - anchore-cli registry add "$CI_REGISTRY" gitlab-ci-token "$CI_JOB_TOKEN" --skip-validate 
  - anchore_ci_tools.py -a -r --timeout 500 --image $IMAGE_NAME

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