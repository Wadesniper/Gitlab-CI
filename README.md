# Gitlab-CI
L'objectif, c'est de mettre en place un pipeline ci/cd en respectant les phases suivantes :
-build
-Test d'acceptation
-Release
-Deploy review
-Stop review
-Deploy staging
-Test staging
-Deploy prod
-Test prod


# DockerFile

```Dockerfile
FROM nginx:1.21.1
LABEL maintainer="Ulrich MONJI"
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y curl && \
    apt-get install -y git
RUN rm -Rf /usr/share/nginx/html/*
RUN git clone https://github.com/diranetafen/static-website-example.git /usr/share/nginx/html
CMD nginx -g 'daemon off;'
```

# gitlab-ci.yml

## Déclaration des variables utilisées dans le code, dont EAZYLABS_IP que l’on va surcharger avec l’ip qu’on a généré et récupéré dans le labs d’eazytraining.

La variable INTERNAL_PORT sera le port de notre application.

NB: C’est sur la même machine que les 3 conteneurs (de staging, de production et de review ) seront lancés.
 De ce fait, le port externe sur lequel l’application sera disponible ne doit pas être le même, car il y aurait conflit.

Alors, on a choisi de mettre : 
le port de Staging sur le port 8080
le port de production sur le port 80
le port de l’environnement de review sur le port 8000
Maintenant, pour tester l’application lors de la phase CI, le port externe sera le port 90

La variable CONTAINER_NAME qui est composé d’une variable IMAGE_NAME et d’une variable appelée CI_COMMIT_REF_NAME (variable prédéfinie de jenkins qui va contenir la branche courante)

```yaml
variables:
  EAZYLABS_IP: ip10-0-0-3-ch0nmgjl13rgqen6ntj0
  APP_NAME: serigne-staticwebsite
  EAZYLABS_DOMAIN: direct.docker.labs.eazytraining.fr  
  API_PORT: "1993"
  API_ENDPOINT: ${EAZYLABS_IP}-${API_PORT}.${EAZYLABS_DOMAIN}
  INTERNAL_PORT: 80
  STG_EXTERNAL_PORT: 8080
  PROD_EXTERNAL_PORT: 80
  REVIEW_EXTERNAL_PORT: 8000
  TEST_PORT: 90
  CONTAINER_IMAGE: ${IMAGE_NAME}:${CI_COMMIT_REF_NAME}
```

## Déclaration d’une image docker et d’un service docker:dind
```yaml
image: docker:latest
services:
  - name: docker:dind
    alias: docker
```

## Ordre dans lequel nos jobs vont s'exécuter
```yaml
stages:
  - Build image
  - Test acceptation
  - Release image
  - Deploy review
  - Stop review
  - Deploy staging
  - Test staging
  - Deploy prod
  - Test prod
```

## Job de build et génération d’un artefact que l'on va garder pendant 2 jours en cas de succès
```yaml
docker-build:
  stage: Build image
  script:
    - docker build -t ${APP_NAME} .
    - docker save ${APP_NAME} > ${APP_NAME}.tar
  artifacts:
    paths:
      - "${APP_NAME}.tar"
    when: on_success
    expire_in: 2 days

```

## Ici on va charger notre artefact, notre image et ensuite on le test au niveau du curl .
 Si ça fonctionne alors on passe à la partie Release

```yaml
test acceptation:
  stage: Test acceptation
  script:
    - docker load < ${APP_NAME}.tar
    - docker run -d -p ${TEST_PORT}:${INTERNAL_PORT}  --name webapp ${APP_NAME}
    - apk --no-cache add curl
    - sleep 5
    - curl "http://docker:${TEST_PORT}" | grep -i "Dimension"
```

## Dans la partie Release, on va recharger notre application et ensuite on va tagger avec 2 tags.
Ensuite, on se log au registre de gitlab puis on va pusher les 2 tags.

```yaml
release image:
  stage: Release image
  script:
    - docker load < ${APP_NAME}.tar
    - docker tag ${APP_NAME} "${IMAGE_NAME}:${CI_COMMIT_REF_NAME}"
    - docker tag ${APP_NAME} "${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}"
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker push "${IMAGE_NAME}:${CI_COMMIT_REF_NAME}"
    - docker push "${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}"
```

## Déploiement en environnement de Review

 Au niveau du script on pourrait avoir des commandes Heroku qui permettront de bavarder avec le cloud Heroku afin de déployer notre application.
 Étant donné qu’on a choisi l’approche eazylabs, tout cela se solde par un curl avec les bonnes variables dépendant de l’environnement.
```yaml
deploy review:
  stage: Deploy review
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: http://${EAZYLABS_IP}-${REVIEW_EXTERNAL_PORT}.${EAZYLABS_DOMAIN}
    on_stop: stop review
  only: 
    - merge_requests
  script:
    - apk --no-cache add curl
    - 'curl -X POST http://${API_ENDPOINT}/review -H "Content-Type: application/json" -d "{\"your_name\":\"${APP_NAME}\",\"container_image\":\"${CONTAINER_IMAGE}\", \"external_port\":\"${REVIEW_EXTERNAL_PORT}\", \"internal_port\":\"${INTERNAL_PORT}\"}"'
```


## Stopper l’environnement de review
on utilisera un curl avec le DELETE toujours sur l’url de review

```yaml
stop review:
  stage: Stop review
  variables:
    GIT_STRATEGY: none
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  only: 
    - merge_requests
  when: manual
  script:
    - apk --no-cache add curl
    - 'curl -X DELETE http://${API_ENDPOINT}/review -H "Content-Type: application/json" -d "{\"your_name\":\"${APP_NAME}\"}"'
```

## Le job template pour permettre de tester
```yaml
.test_template: &test
  image: alpine
  script:
    - apk --no-cache add curl
    - curl "http://$DOMAIN" | grep -i "Dimension" 
```

## Le Déploiement en environnement de staging
On retrouve toujours notre curl sauf que l’url sera staging et le port sera STG_EXTERNAL_PORT

```yaml
deploy staging:
  stage: Deploy staging
  environment:
    name: staging
    url: http://${EAZYLABS_IP}-${STG_EXTERNAL_PORT}.${EAZYLABS_DOMAIN}

  script:
    - apk --no-cache add curl
    - 'curl -v -X POST http://${API_ENDPOINT}/staging -H "Content-Type: application/json" -d "{\"your_name\":\"${APP_NAME}\",\"container_image\":\"${CONTAINER_IMAGE}\", \"external_port\":\"${STG_EXTERNAL_PORT}\", \"internal_port\":\"${INTERNAL_PORT}\"}"  2>&1 | grep 200'
```

## Test en environnement de staging
```yaml
test staging:
  stage: Test staging
  <<: *test
  variables:
    DOMAIN: ${EAZYLABS_IP}-${STG_EXTERNAL_PORT}.${EAZYLABS_DOMAIN}
```

## Le Déploiement en environnement de production
exactement pareil, on retrouve toujours notre curl sauf que l’url sera prod et le port sera PROD_EXTERNAL_PORT.

Sur la prod, le déploiement et le test auront lieu uniquement si on est sur la branche main.
```yaml
deploy prod:
  stage: Deploy prod
  environment:
    name: prod
    url: http://${EAZYLABS_IP}-${PROD_EXTERNAL_PORT}.${EAZYLABS_DOMAIN}
  only: 
    - main
  script:
    - apk --no-cache add curl
    - 'curl -v -X POST http://${API_ENDPOINT}/prod -H "Content-Type: application/json" -d "{\"your_name\":\"${APP_NAME}\",\"container_image\":\"${CONTAINER_IMAGE}\", \"external_port\":\"${PROD_EXTERNAL_PORT}\", \"internal_port\":\"${INTERNAL_PORT}\"}" 2>&1 | grep 200'
```
## Test en environnement de production
```yaml
test prod:
  stage: Test prod
  <<: *test
  variables:
    DOMAIN: ${EAZYLABS_IP}-${PROD_EXTERNAL_PORT}.${EAZYLABS_DOMAIN}
  only:
    - main
```


Pour savoir d'où viennent les CURL et retrouvé les explications de ce qui est fait, je vous invite à vous rendre dans cette documentation :
## https://github.com/eazytraining/eazylabs

# Voyons le résultat !

![PipSuccess1 1](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/49606796-356c-41fc-bea4-4b7724a59d58)


## Passons maintenant aux vérifications.

1er vérification : la build et la release
Sur Gitlab, cliquons sur "Package" et puis sur "container registry"
on peut bien voir notre image avec nos tags.

![OutputBuildRelease2](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/960a7e3e-14d1-4895-990e-4db0869a864b)


2iem vérification : sur dockerps
on voit notre conteneur qui tourne , disponible sur le port 8080
![dockerps1 2](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/e5db0640-1ead-451c-a8a9-8397ce24ef92)


3iem vérification : allons dans la partie "Deployment" puis sur "environnement", est-ce qu'on aura un environnement de staging ?

![deplomentEnv](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/a1e521fd-6379-49c1-a8d9-9deb51fda7bc)

Oui, Parfait !

*Output de l'application en environnement de staging sur le port 8080 qu'on a défini

![Output4](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/19c3f19c-4678-4918-b51c-e21bef0c7ad9)

Maintenant,simulons une mise en prod en faisant une mergerequest

![Mergerequest5 1](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/3c8dfc4a-b23f-4892-9967-c96939c514bf)

Dans l'image suivante, on peut observer qu'un environnement de review se lance .
![envreview5 2](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/3e60b68f-d5f4-4a0f-8c62-371981ac06eb)

Pourquoi ce déploiement en environnement de review est fait ?

Lorsqu'on veut faire un déploiement en production, il faut qu'on se rassure que le code que l’on a actuellement est toujours fonctionnel si on le valide en production.
Cela servira à un manager/un lead technique pour valider la merge request.


Revenons dans les différents environnements , est-ce qu’il y a l'environnement de review ?, Oui

![deploymentEnv Review(6)](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/907bca6a-5c89-4eab-837b-7c28c518ee4f)

On clique sur “Open”, et on voit qu'il est bien disponible sur le port 8000 :

![depenv7](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/eb9bc08a-84e5-478d-8d95-c55de70edd35)

et si on revient sur le labs eazytraining, et qu’on fait un dockerps, on verra bien qu’il y’a un nouveau container disponible sur le port 8000 :

![docker7 2](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/4eeaeed7-434c-43ae-81b4-2a259da6fd5b)

faisons maintenant une action manuelle pour stopper la review, tel précisé dans notre code.

![stopreview8](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/f44321aa-f894-4855-b557-776d19946a9c)


nous verrons que l'environnement review sera stoppé et le pipeline complet se lance :

![Pipecomplet(9)](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/fef9f67b-f906-4e33-aab7-04fd34ef63ee)



Output de l'application en environnement de production sur le port 80 qu'on a défini

![EnvPROD10](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/1755fc6b-407c-4ae4-86f3-26ec8d0486f2)

![EnvProd10 2](https://github.com/Wadesniper/Gitlab-CI/assets/57738842/01a0e83b-ab28-4f01-9c8f-0bf71758d000)

## Conclusion
Notre chaîne ci/cd est bel et bien fonctionnel et on a pu le déployer sur l'outil eazylabs.
