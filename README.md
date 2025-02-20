# triron-inference-server-tutorial

## Что потребуется установить 
- Docker: https://docs.docker.com/engine/install/
- Установить NVIDIA Container Toolkit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
- Образы triton infernece server: https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tritonserver/tags

## Что может быть полезно
- Построение ансамблей моделей: [ссылка](https://developer.nvidia.com/blog/serving-ml-model-pipelines-on-nvidia-triton-inference-server-with-ensemble-models/)
- Репозиторий с туториалами от NVIDIA: [ссылка](https://github.com/triton-inference-server/tutorials)
- Tritonserver Model Analyzer :
  1) GitHub Репозиторий Model Analyzer: [ссылка](https://github.com/triton-inference-server/model_analyzer)
  2) Пример для ансамблей: [ссылка](https://github.com/triton-inference-server/model_analyzer/blob/main/docs/ensemble_quick_start.md)
  3) Флаги для написания файла конфигурации анализатора (advanced): [ссылка](https://github.com/triton-inference-server/model_analyzer/blob/main/docs/config.md#config-options-for-profile)


## 1. Установка NVIDIA Container Toolkit
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```bash
sudo apt-get update
```

```bash
sudo apt-get install -y nvidia-container-toolkit
```
### Установка Triton Inference Server
```bash
docker pull nvcr.io/nvidia/tritonserver:24.01-py3
```
```bash
docker images
```
## 2. Базовые понятия triton infernece server
```bash
# Expected folder layout
model_repository/
├── text_detection
│   ├── 1
│   │   └── model.onnx
│   └── config.pbtxt
└── text_recognition
    ├── 1
    │   └── model.onnx
    └── config.pbtxt
```

**config.pbtxt**
``` text proto
name: "text_detection"
backend: "onnxruntime"
max_batch_size : 256
input [
  {
    name: "input_images:0"
    data_type: TYPE_FP32
    dims: [ -1, -1, -1, 3 ]
  }
]
output [
  {
    name: "feature_fusion/Conv_7/Sigmoid:0"
    data_type: TYPE_FP32
    dims: [ -1, -1, -1, 1 ]
  }
]
output [
  {
    name: "feature_fusion/concat_3:0"
    data_type: TYPE_FP32
    dims: [ -1, -1, -1, 5 ]
  }
]

dynamic_batching {
    preferred_batch_size: [ 4, 8 ]
  }
```

Разберем файл конфигов
- name - Необязательно поле, которое обозначает название модели (Note: Название должно совпадать с названием каталога!)
- backend/platform* - указывает на бэкенд, который используется для запуска модели. Бэкенды, которые поддерживает triton: [Документация](https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton_inference_server_220/user-guide/docs/backend.html), [Github](https://github.com/triton-inference-server/backend#backends)
- max_batch_size - Значение, которое указывает на максимальный размер батча, который воспринимает модель
- input and output: Конфигурация полей для  

[Все возможные поля конфигурации](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_configuration.html)

*platform Является старой нотацией, новым стандартом является использование backend


На примере выше, размерность входных данных соотвествует при max_batch_size > 0:
```
[-1] + dims = [-1, -1, -1, -1, 3]
```
Для моделей с max_batch size = 0:
```
dims = [-1, -1, -1, 3]
```
## 3. Инференс с помощью Triton Inference Server
Для начала инференса потребутеся запустить контейнер с triton inference server, запустить на нем модели и отправить запрос с клиента. 

**!! NOTE !!** - все скрипты должны выполняться в корневой директории.\
Запуск будет рассмотрен на примере simple_example.
### 3.0 Примеры представленные в репозитории 
| [Простой пример](./simple_example/README.md) | [Пример ансамбля](./ensemble_model/README.md) |
| --------------- | ------------ |
### 3.1 Загрузить onnx веса в model repository
```bash
source ./utils/yolo_export.sh
```
### 3.2 Запуск контейнера и triton
```bash
docker run --rm -it --gpus=0 -p8000:8000 -p8001:8001 -p8002:8002 -v $(pwd)/model_repository:/models nvcr.io/nvidia/tritonserver:24.01-py3

tritonserver --model-repository=/models
```

### 3.3 Отправка запроса через скрипт клиента
```bash
python client.py
```
Как результат в консоль должен вывестись JSON ответа и появиться файл `result.jpg`
```bash
{'model_name': 'detection',
 'model_version': '1',
 'outputs': [{'datatype': 'FP32',
              'name': 'output0',
              'parameters': {'binary_data_size': 2822400},
              'shape': [1, 84, 8400]}]}
```
## 4. Model Analyzer
Установка sdk для анализа проведения исследований возможностей инференса 
```bash
docker run -it --net=host -v ${PWD}:/workspace/ nvcr.io/nvidia/tritonserver:24.01-py3-sdk bash
```
### Performance Analyzer
Для использвание Perf Analyzer потребуется:
1) Запустить Trition 
```bash
docker run --rm -it --gpus=0 --net=host -p8000:8000 -p8001:8001 -p8002:8002 -v $(pwd)/model_repository:/models nvcr.io/nvidia/tritonserver:24.01-py3
```
2) Запустить SDK контейнер
```bash
docker run -it --net=host -v ${PWD}:/workspace/ nvcr.io/nvidia/tritonserver:24.01-py3-sdk bash
```
3) Запустить Perf Analyzer
```bash
# perf_analyzer -m <model name> -b <batch size> --shape <input layer>:<input shape> --concurrency-range <lower number of request>:<higher number of request>:<step>

# Query
perf_analyzer -m detection --shape images:3,640,640 --concurrency-range 2:6:2 --percentile=95
```
по результатам анализа получается:
```
*** Measurement Settings ***
  Batch size: 1
  Service Kind: Triton
  Using "time_windows" mode for stabilization
  Measurement window: 5000 msec
  Latency limit: 0 msec
  Concurrency limit: 6 concurrent requests
  Using synchronous calls for inference
  Stabilizing using p95 latency

    ...

  Request concurrency: 6
  Client: 
    Request count: 4065
    Throughput: 225.786 infer/sec
    p50 latency: 23841 usec
    p90 latency: 45345 usec
    p95 latency: 63844 usec
    p99 latency: 85863 usec
    Avg HTTP time: 26553 usec (send/recv 4594 usec + response wait 21959 usec)
  Server: 
    Inference count: 4066
    Execution count: 4066
    Successful request count: 4066
    Avg request latency: 15726 usec (overhead 24 usec + queue 7473 usec + compute input 719 usec + compute infer 6990 usec + compute output 519 usec)

  Inferences/Second vs. Client p95 Batch Latency
  Concurrency: 2, throughput: 163.593 infer/sec, latency 18085 usec
  Concurrency: 4, throughput: 209.689 infer/sec, latency 35305 usec
  Concurrency: 6, throughput: 225.786 infer/sec, latency 63844 usec

```

### Model Analyzer
1) Запустить Trition c explicit флагом
```bash
docker run --gpus=0 --shm-size=1G --rm -it -p8000:8000 -p8001:8001 -p8002:8002 -v $(pwd)/model_repository:/models nvcr.io/nvidia/tritonserver:24.01-py3

tritonserver --model-repository=/models --model-control-mode=explicit
```
2) Запустить контейнер SDK контейнер
```bash
docker run -it --gpus all -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/workspace --net=host nvcr.io/nvidia/tritonserver:24.01-py3-sdk
```
3) Запуститить Model Analyzer для модели детекции
```bash
model-analyzer profile \
    --triton-launch-mode=remote \
    --triton-docker-shm-size=1G \
    --output-model-repository-path=$(pwd)/gen_conf/output_dir \
    --export-path $(pwd)/profile_results \
    --override-output-model-repository \
    --profile-models detection\
    --triton-grpc-endpoint=0.0.0.0:8001 \
    --triton-http-endpoint=0.0.0.0:8000
```
По результатам анализатор выводит графики с метриками ля различных конфигураций развертывания модели на сервере:
![latency](./simple_example/images/latency_breakdown.png)

![latency](./simple_example/images/throughput_v_latency.png)