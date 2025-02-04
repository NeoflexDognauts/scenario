1. Немного определений	
    ```python
    MODEL_NAME = "default-linear-reg"
    INPUT_FILE = "airflow-batch/to_infer.parquet"
    OUTPUT_FILE = "airflow-batch/inferred-data/inferred.parquet"
    S3_OPTIONS = {
        'key': os.getenv('AWS_ACCESS_KEY_ID'),
        'secret': os.getenv('AWS_SECRET_ACCESS_KEY'),
        'client_kwargs': {'endpoint_url': os.getenv('KUBERNETES_MLFLOW_S3_ENDPOINT_URL')},
    }
    BUCKET = os.getenv('KUBERNETES_S3_BUCKET')
    ``` 
1. Получаем список зарегистрированных версий модели по имени модели
    ```python
    models = mlflow.search_registered_models(filter_string=f'name = "{MODEL_NAME}"')
    ```
    
1. Получаем последнюю версию модели в стейже Production
    ```python
    last_version = max([int(i.version) for i in models[0].latest_versions if i.current_stage == 'Production'])
    ```

    *Дело в том, что в теории у нас может быть одновременно несколько версий модели в на стейже Production, поэтому напрямую получить версию модели по стейжу не получится . На данный момент это API считается устаревшим и на смену маркеру Stage пришел маркер Alias. При использовании Alias кроме расширенного функционала по установке меток так же гарантируется уникальность — только одна версия конкретной модели может содержать определенную метку. И в этом случае ситуация с тем, что две версии модели оказываются в статусе Production исключена. В нашем примере покажем функциональность Stage а не Alias из соображений исторической совместимости.*
    
1. Получаем модель из MLfow
    ```python
    model = mlflow.pyfunc.load_model(f'models:/{MODEL_NAME}/{last_version}')
    ```
    
1. Получаем паркет для инференса из S3. В принципе здесь может быть любой источник данных)
    ```python
    data = pd.read_parquet(f's3://{BUCKET}/{INPUT_FILE}', storage_options=S3_OPTIONS)
    ```
    
1. Выполняем предсказания
    ```python
    predictions = model.predict(data)
    ```
    
1. Сохраняем ответ в S3
    ```python
    pd.DataFrame(predictions, columns=['pred']).to_parquet(
                f's3://{BUCKET}/{OUTPUT_FILE}', storage_options=S3_OPTIONS
            )
    ```
       