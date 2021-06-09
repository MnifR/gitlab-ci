# Collection of GitLab Pipelines

## laravel
```yaml
stages:
  - pull_code_test
  - pull_code_production
  - install_deps
  - test
  - build
  - deploy_test
  - deploy_production
variables:
  PHP_FPM_CONTAINER: lnmp-php-fpm
  WORK_DIR: /usr/share/nginx/html/
  PROJECT: laravel-demo
  GIT_DIR: /mnt/lnmp-docker

pull_code_test: 
  stage: pull_code_test
  only: 
    - develop
  script: 
     - cd ${GIT_DIR}/${PROJECT}
     - git pull origin develop
pull_code_production:
  stage: pull_code_production
  only:
    - master
  script: 
    - cd ${GIT_DIR}/${PROJECT}
    - git pull origin master

install_deps:
  stage: install_deps
  script: 
    - docker exec -w ${WORK_DIR}/${PROJECT} ${PHP_FPM_CONTAINER} composer install
build: 
  stage: build
  script: 
    # Run migrations
    - docker exec -w ${WORK_DIR}/${PROJECT} ${PHP_FPM_CONTAINER} php artisan migrate
    # Cache clearing
    - docker exec -w ${WORK_DIR}/${PROJECT} ${PHP_FPM_CONTAINER} php artisan cache:clear
    # Create a cache file for faster configuration loading
    - docker exec -w ${WORK_DIR}/${PROJECT} ${PHP_FPM_CONTAINER} php artisan config:cache
    # Create a route cache file for faster route registration
    - docker exec -w ${WORK_DIR}/${PROJECT} ${PHP_FPM_CONTAINER} php artisan route:clear
deploy_test: 
  stage: deploy_test
  script:
    - cd ${GIT_DIR}
    - docker-compose down && docker-compose build && docker-compose up -d
deploy_production: 
  stage: deploy_production
  script:
    - cd ${GIT_DIR}
    - docker-compose restart
```
## Building Docker Image With Kaniko

```yaml
build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
  rules:
    - if: $CI_COMMIT_TAG

```


## Building Node.js

```yaml
backend-server-image-build:
  image: docker:19.03.1
  stage: build
  variables:
    CI_CONTAINER_IMAGE: <username>/<image-name>:$CI_COMMIT_SHORT_SHA
  before_script:
    - docker login -u <username> -p <password>
  script:
    - docker build -t "$CI_CONTAINER_IMAGE" .
    - docker tag "$CI_CONTAINER_IMAGE" "$CI_CONTAINER_IMAGE"
    - docker tag "$CI_CONTAINER_IMAGE" <username>/<image-name>:latest
    - docker push "$CI_CONTAINER_IMAGE"
  after_script:
    - docker login -u <username> -p <password>
    - docker image rm "<username>/<image-name>:$CI_COMMIT_SHORT_SHA" -f
    - docker system prune -f
```

## Testing Node.js

```yaml
backend-server-image-test:
  image: <username>/<image-name>:$CI_COMMIT_SHORT_SHA
  stage: test
  script:
    - npm --prefix /usr/src/app test
    - exit
```

## Cleanup Artifacts (unwanted docker/image/container/volume)

```yaml
cleanup:
  stage: .post
  image: docker:19.03.1
  script:
    - docker login -u <username> -p <password>
    - docker image rm "<username>/<image-name>:$CI_COMMIT_SHORT_SHA" -f
    - docker system prune -f
```

## Image Delivery To Test Kubernetes (Non Master Branch)

```yaml
backend-server-image-deliver-test-environment:
  environment:
    name: k8s-test
    url: 172.31.52.51:30012
  image: <username>/ubuntu-for-ssh:20200514-stable-1
  stage: deploy
  variables:
    CI_CONTAINER_IMAGE: <username>\/<image-name>:$CI_COMMIT_SHORT_SHA
  before_script:
    - echo "172.31.52.51 ecdsa-sha2-nistp256 <ssh-key>" > /root/.ssh/known_hosts
  script:
    - sshpass -p macroview ssh 172.31.52.51 << EOL
    - cd /root/billy/web-based-defacement-detection/deployment/test
    - sed -i -E "s/(image:\s).*/\1$CI_CONTAINER_IMAGE/" backend.yaml
    - kubectl apply -f backend.yaml
    - >
      if ! kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>; then
          kubectl rollout undo deployment <k8s-deployment-name> -n <k8s-namespace>
          kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>
          exit 1
      fi
    - EOL
  except:
    - master
```

## Image Delivery To Production Kubernetes (Master Branch)

```yaml
backend-server-image-deliver-test-environment:
  environment:
    name: k8s-test
    url: 172.31.52.51:30012
  image: <username>/ubuntu-for-ssh:20200514-stable-1
  stage: deploy
  variables:
    CI_CONTAINER_IMAGE: <username>\/<image-name>:$CI_COMMIT_SHORT_SHA
  before_script:
    - echo "172.31.52.51 ecdsa-sha2-nistp256 <ssh-key>" > /root/.ssh/known_hosts
  script:
    - sshpass -p macroview ssh 172.31.52.51 << EOL
    - cd /root/billy/web-based-defacement-detection/deployment/test
    - sed -i -E "s/(image:\s).*/\1$CI_CONTAINER_IMAGE/" backend.yaml
    - kubectl apply -f backend.yaml
    - >
      if ! kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>; then
          kubectl rollout undo deployment <k8s-deployment-name> -n <k8s-namespace>
          kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>
          exit 1
      fi
    - EOL
```
