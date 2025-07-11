# Подготовка базовых образов для вывода в эксплуатацию модельных сервисов инструментом Seldon Code на базе моделей, зарегистрированных в MLflow с помощью визарда создания манифеста Seldon Deployment на платформе Neoflex Dognauts

### Оглавление

1. Пререквизиты
1. Сборка базового образа
1. Установка образа в качестве образа по-умолчанию для всей Платформы
1. Создание манифеста с помощью визарда
1. Установка отдельного образа для экземпляра модельного сервиса


### Пререквизиты
Так как Neoflex Dognauts использует MLflow в качестве реестра моделей, то в этом примере мы будем использовать MLFLOW_SERVER. Для того чтобы создать модельный сервис нужно сначала зарегистрировать в MLflow, пример можно найти в [сценарии MLFlow](0_MLflow.md) подробную информацию о моделях MLflow можно найти в [Документации](https://mlflow.org/docs/2.8.0/models.html)

### Сборка базового образа
Seldon Core поддерживает несколько видов инференс-серверов, такие как MLFLOW_SERVER, SKLEARN_SERVER, TENSORFLOW_SERVER, TRITON_SERVER и.т.д.. Так как мы используем модель из MLflow, то в этом примере будем использовать MLFLOW_SERVER.

Для запуска модельного сервиса понадобятся две сущности — это базовый образ и модельные артефакты. Базовый образ нужно либо взять готовый либо собрать свой - это сокращает время запуска модельного сервиса и уменьшает риски, связанные с установкой зависимостей во время запуска сервиса. Ниже пример `Dockerfile` для сборки такого образа:


    FROM harbor-dognauts.neoflex.ru/dognauts/mlflowserver-pyyaml:1.18.0
    COPY conda.yaml /tmp/conda.yaml
    COPY conda_env_create.py /microservice/conda_env_create.py 
    COPY run /s2i/bin/run 
    RUN /bin/sh -c conda env create -n mlflow --file /tmp/conda.yaml 
    USER root
    RUN /bin/sh -c chmod 777 -R /.cache/ 
    RUN /bin/sh -c chmod 777 -R /etc/profile.d/ 
    RUN /bin/sh -c chmod 777 -R /s2i/bin/run 
    RUN /bin/sh -c cp /microservice/MLFlowServer.py /opt/conda/envs/mlflow/lib/python3.10/MLFlowServer.py 
    USER 8888


В процессе сборки нужно организовать проверку образа на наличие известных уязвимостей, например с помощью сканера `trivy`


    trivy image registry/some_image:tag


После того, как базовый образ подготовлен, проверен и допущен к эксплуатации его можно использовать в модельном сервисе.

### Установка образа в качестве образа по-умолчанию для всей Платформы

Что бы установить образ в качестве образа по-умолчанию для всех запускаемых модельных сервисов нужно в `ConfigMap` `seldon-config` в NS `seldon-system` установить `values` релиза `seldon-core` новые образ и версию:

    predictor_servers:
    MLFLOW_SERVER:
        protocols:
        seldon:
            defaultImageVersion: "1.17.1"
            image: seldonio/mlflowserver


### Установка образа в качестве базового для определенного seldonDeployment

Что бы установить образ в качестве базового для определенного seldonDeployment нужно в манифесте деплоймена в соответствующем контейнере указать этот образ. Например:

    apiVersion: machinelearning.seldon.io/v1
    kind: SeldonDeployment
    metadata:
    name: dogs
    namespace: app-msoprod
    spec:
    predictors:
        - componentSpecs:
            - spec:
                containers:
                - env:
                    image: >-
                    harbor-dognauts.neoflex.ru/dognauts/mlflowserver-demo-mlsecops:1.18.0
                    name: dogs
        graph:
            envSecretRefName: seldon-secret
            implementation: MLFLOW_SERVER
            modelUri: s3://path_to_model/model
            name: dogs
        name: dogscurrent
        replicas: 1

