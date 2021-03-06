![PipelineAI Logo](http://pipeline.ai/assets/img/logo/pipelineai-split-black-258x62.png)

# Pre-Requisites
## Install PipelineAI CLI
* Click [**HERE**](../README.md#install-pipelinecli) to install the PipelineAI CLI

## Pull PipelineAI [Sample Models](https://github.com/PipelineAI/models)
```
git clone https://github.com/PipelineAI/models
```
**Change into the new `models/` directory**
```
cd models
```

## Install Tools
* [Docker](https://www.docker.com/community-edition#/download)
* Python 2 or 3 ([Conda](https://conda.io/docs/install/quick.html) is Preferred)
* (Windows Only) [PowerShell](https://github.com/PowerShell/PowerShell/tree/master/docs/installation) 
* [Kubernetes Cluster](/docs/kube-setup/)

**Minimum Version**
* Edge (Not Stable)
* 18.02-ce
* Kubernetes v1.9.2

![Docker for Desktop Kubernetes About](http://pipeline.ai/assets/img/docker-desktop-kubernetes-about.png)

**Minimum Docker System Requirements**
* 8GB
* 4 Cores

### (Optional) Install Docker for Mac|Windows with Kubernetes!

**Enable Kubernetes Locally**

Mac

![Docker for Desktop Kubernetes Mac](https://pipeline.ai/assets/img/docker-desktop-kubernetes-about.png)

Windows

![Docker for Desktop Kubernetes Windows](http://pipeline.ai/assets/img/docker-for-desktop-windows.png)

### Configure Kubernetes CLI for Local Kubernetes Cluster
```
kubectl config use-context docker-for-desktop
```

TensorFlow + Kubernetes
* [Build and Deploy New Model Prediction Server Variant](#build-model-prediction-servers---versions-a-and-b-tensorflow-based)
* [Split Live Traffic to New Model Prediction Server Variant](#split-traffic-between-model-version-a-50-and-model-version-b-50)
* [Scale Model Prediction Servers](#scale-model-prediction-servers---version-b-to-2-replicas)
* [Shadow Live Traffic to New Model Prediction Server Variant](#shadow-traffic-from-model-version-a-100-live-to-model-version-b-0-live-only-shadow-traffic)
* [Install Model Prediction Server Dashboards](#install-dashboards)
* [Prediction with REST API](#predict-with-rest-api)
* [Distributed Model Training (CPU)](#distributed-tensorflow-training-cpu)
* [Distributed Model Training (GPU)](#distributed-tensorflow-training-gpu)

### Build Model Prediction Servers - Versions a and b (TensorFlow-based)
Notes:
* You must be in the `models/` directory created from the `git clone` above.

[**CPU (version a)**](https://github.com/PipelineAI/models/tree/master/tensorflow/mnist-cpu)
```
pipeline predict-server-build --model-name=mnist --model-tag=a --model-type=tensorflow --model-path=./tensorflow/mnist-cpu/model 
```
* For GPU-based models, make sure you specify `--model-chip=gpu`

[**CPU (version b)**](https://github.com/PipelineAI/models/tree/master/tensorflow/mnist-cpu)
```
pipeline predict-server-build --model-name=mnist --model-tag=b --model-type=tensorflow --model-path=./tensorflow/mnist-cpu/model 
```
* For GPU-based models, make sure you specify `--model-chip=gpu`

### Register Model Prediction Server with Docker Repo - Versions a and b
Notes:  
* This can be any Docker Repo including DockerHub and internal repos
* You must already be logged in to the Docker Repo using `docker login`

Defaults
* `--image-registry-url`:  docker.io
* `--image-registry-repo`:  pipelineai

[**CPU (version a)**](https://github.com/PipelineAI/models/tree/master/tensorflow/mnist-cpu)
```
pipeline predict-server-register --model-name=mnist --model-tag=a 
```

[**CPU (version b)**](https://github.com/PipelineAI/models/tree/f559987d7c889b7a2e82528cc72d003ef3a34573/tensorflow/mnist-cpu)
```
pipeline predict-server-register --model-name=mnist --model-tag=b 
```

### Install [Istio Service Mesh CLI](https://istio.io/docs/setup/kubernetes/quick-start.html#installation-steps)
**Mac**
```
curl -L https://github.com/istio/istio/releases/download/0.6.0/istio-0.6.0-osx.tar.gz | tar xz

export PATH=./istio-0.6.0/bin:$PATH
```

**Linux**
```
curl -L https://github.com/istio/istio/releases/download/0.6.0/istio-0.6.0-linux.tar.gz | tar xz

export PATH=./istio-0.6.0/bin:$PATH
```

**Windows**
```
curl -L https://github.com/istio/istio/releases/download/0.6.0/istio_0.6.0_win.zip

# Unzip and Set PATH...
```

**Verify Successful CLI Install**
```
which istioctl

### EXPECTED OUTPUT ###
./istio-0.6.0/bin/istioctl
```
Note:  You'll want to put `istioctl` on your permanent PATH - or copy to `/usr/local/bin`

### Deploy Istio to Cluster
```
kubectl apply -f ./istio-0.6.0/install/kubernetes/istio.yaml
```

**Verify Istio Components**
```
kubectl get all --namespace=istio-system

### EXPECTED OUTPUT ###
NAME                                READY     STATUS    RESTARTS   AGE
po/istio-ca-797dfb66c5-wxlbk        1/1       Running   0          10d
po/istio-ingress-67ff757554-zjzz2   1/1       Running   0          10d
po/istio-mixer-5bf5b5b94c-w5xnp     3/3       Running   0          10d
po/istio-pilot-676d495bf8-mzch5     2/2       Running   0          10d
NAME                CLUSTER-IP       EXTERNAL-IP   PORT(S)                                         
                   AGE
svc/istio-ingress   10.110.118.75    <pending>     80:31202/TCP,443:30654/TCP                      
                   10d
svc/istio-mixer     10.100.187.229   <none>        9091/TCP,15004/TCP,9093/TCP,9094/TCP,9102/TCP,91
25/UDP,42422/TCP   10d
svc/istio-pilot     10.96.104.118    <none>        15003/TCP,8080/TCP,9093/TCP,443/TCP             
                   10d
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/istio-ca        1         1         1            1           10d
deploy/istio-ingress   1         1         1            1           10d
deploy/istio-mixer     1         1         1            1           10d
deploy/istio-pilot     1         1         1            1           10d
NAME                          DESIRED   CURRENT   READY     AGE
rs/istio-ca-797dfb66c5        1         1         1         10d
rs/istio-ingress-67ff757554   1         1         1         10d
rs/istio-mixer-5bf5b5b94c     1         1         1         10d
rs/istio-pilot-676d495bf8     1         1         1         10d
```

### Deploy Model Prediction Servers - Versions a and b (TensorFlow-based)
Notes:
* Make sure you install Istio.  See above!
* Make sure nothing is running on port 80 (ie. default Web Server on your laptop).

[**CPU (version a)**](https://github.com/PipelineAI/models/tree/master/tensorflow/mnist-cpu)
```
pipeline predict-kube-start --model-name=mnist --model-tag=a
```
* For GPU-based models, make sure you specify `--model-chip=gpu`

[**CPU (version b)**](https://github.com/PipelineAI/models/tree/master/tensorflow/mnist-cpu)
```
pipeline predict-kube-start --model-name=mnist --model-tag=b
```
* For GPU-based models, make sure you specify `--model-chip=gpu`

### View Running Pods
```
kubectl get pod

### EXPECTED OUTPUT###
NAME                          READY     STATUS    RESTARTS   AGE
predict-mnist-a-...-...       2/2       Running   0          5m
predict-mnist-b-...-...       2/2       Running   0          5m
```

### Split Traffic Between Model Version a (50%) and Model Version b (50%)
```
pipeline predict-kube-route --model-name=mnist --model-split-tag-and-weight-dict='{"a":50, "b":50}' --model-shadow-tag-list='[]'
```
Notes:
* If you specify a model in `--model-shadow-tag-list`, you need to explicitly specify 0% traffic split in `--model-split-tag-and-weight-dict`
* If you see `apiVersion: Invalid value: "config.istio.io/__internal": must be config.istio.io/v1alpha2`, you need to [remove the existing route rules](#clean-up) and re-create them with this command.

### View Route Rules
```
kubectl get routerules

### EXPECTED OUTPUT ###
NAME                            KIND
predict-mnist-dashboardstream   RouteRule.v1alpha2.config.istio.io
predict-mnist-denytherest       RouteRule.v1alpha2.config.istio.io
predict-mnist-invocations       RouteRule.v1alpha2.config.istio.io
predict-mnist-metrics           RouteRule.v1alpha2.config.istio.io
predict-mnist-ping              RouteRule.v1alpha2.config.istio.io
predict-mnist-prometheus        RouteRule.v1alpha2.config.istio.io
```

### Load Test Model Prediction Servers - Versions a and b
Kube-native (Requires LoadBalancer)
```
pipeline predict-kube-test --model-name=mnist --test-request-path=./tensorflow/mnist-cpu/input/predict/test_request.json --test-request-concurrency=1000
```

Http (See Notes Below)
```
pipeline predict-http-test --endpoint-url=http://<ingress-controller-ip>:<ingress-controller-port>/predict/mnist/invocations --test-request-path=./tensorflow/mnist-cpu/input/predict/test_request.json --test-request-concurrency=1000
```
Notes:
* Use the following to get the `<ingress-controller-ip>` and `<ingress-controller-port>`:
```
# <ingress-controller-ip>
$(kubectl -n istio-system get po -l istio=ingress -o jsonpath='{.items[0].status.hostIP}')

# <ingress-controller-port>
$(kubectl get svc -n istio-system istio-ingress -o jsonpath='{.spec.ports[0].nodePort}')
```
* You need to be in the `models/` directory created when you performed the `git clone` [above](#pull-pipelineai-sample-models).
* If you see `no healthy upstream` or `502 Bad Gateway`, just wait 1-2 mins for the model servers to startup.
* If you see a `404` error related to `No message found /predict/mnist/invocations`, the route rules above were not applied.
* See [Troubleshooting](/docs/troubleshooting) for more debugging info.

**Expected Output**
* You should see a 50/50 split between Model Version a and Version b.

[**CPU (version a)**](https://github.com/PipelineAI/models/tree/f559987d7c889b7a2e82528cc72d003ef3a34573/tensorflow/a)
```
('{"variant": "mnist-a-tensorflow-tfserving-cpu", "outputs":{"outputs": '
 '[0.11128007620573044, 1.4478533557849005e-05, 0.43401220440864563, '
 '0.06995827704668045, 0.0028081508353352547, 0.27867695689201355, '
 '0.017851119861006737, 0.006651509087532759, 0.07679300010204315, '
 '0.001954273320734501]}}')
 
Request time: 36.414 milliseconds

### FORMATTED OUTPUT ###
Digit  Confidence
=====  ==========
0      0.11128007620573044
1      0.00001447853355784
2      0.43401220440864563      <-- Prediction
3      0.06995827704668045
4      0.00280815083533525
5      0.27867695689201355 
6      0.01785111986100673
7      0.00665150908753275
8      0.07679300010204315
9      0.00195427332073450
```

[**CPU (version b)**](https://github.com/PipelineAI/models/tree/master/tensorflow/mnist-cpu)
 ```
('{"variant": "mnist-b-tensorflow-tfserving-cpu", "outputs":{"outputs": '
 '[0.11128010600805283, 1.4478532648354303e-05, 0.43401211500167847, '
 '0.06995825469493866, 0.002808149205520749, 0.2786771059036255, '
 '0.01785111241042614, 0.006651511415839195, 0.07679297775030136, '
 '0.001954274717718363]}}')

Request time: 29.968 milliseconds

### FORMATTED OUTPUT ###
Digit  Confidence
=====  ==========
0      0.11128010600805283
1      0.00001447853264835
2      0.43401211500167847      <-- Prediction
3      0.06995825469493866
4      0.00280814920552074
5      0.27867710590362550 
6      0.01785111241042614
7      0.00665151141583919
8      0.07679297775030136
9      0.00195427471771836
 ```

### Scale Model Prediction Servers - Version b to 2 Replicas
Scale the Model Server
```
pipeline predict-kube-scale --model-name=mnist --model-tag=b --replicas=2
```

**Verify Scaling Event**
```
kubectl get pod

### EXPECTED OUTPUT###

NAME                          READY     STATUS    RESTARTS   AGE
predict-mnist-b-...-...       2/2       Running   0          8m
predict-mnist-b-...-...       2/2       Running   0          10m
predict-mnist-a-...-...       2/2       Running   0          10m
```

**Re-run LoadTest**
```
pipeline predict-kube-test --model-name=mnist --test-request-path=./tensorflow/mnist-cpu/input/predict/test_request.json --test-request-concurrency=1000
```
Notes:
* If you see `502 Bad Gateway`, this is OK!  You just need to wait 1-2 mins for the model servers to startup.
* You should still see a 50/50 split between Model Version a and Model Version b - even after scaling out Model Version b!

### Shadow Traffic from Model Version a (100% Live) to Model Version b (0% Live, Only Shadow Traffic)
```
pipeline predict-kube-route --model-name=mnist --model-split-tag-and-weight-dict='{"a":100, "b":0}' --model-shadow-tag-list='["b"]'
```
Notes:
* If you specify a model in `--model-shadow-tag-list`, you need to explicitly specify 0% traffic split in `--model-split-tag-and-weight-dict`
* If you see `apiVersion: Invalid value: "config.istio.io/__internal": must be config.istio.io/v1alpha2`, you need to [remove the existing route rules](#clean-up) and re-create them with this command.

### Run LoadTest on Model Prediction Servers - Versions a and b
```
pipeline predict-kube-test --model-name=mnist --test-request-path=./tensorflow/mnist-cpu/input/predict/test_request.json --test-request-concurrency=1000
```
Notes:
* You need to be in the `models/` directory created when you performed the `git clone` [above](#pull-pipelineai-sample-models).
* If you see `no healthy upstream` or `502 Bad Gateway`, just wait 1-2 mins for the model servers to startup.
* If you see a `404` error related to `No message found /predict/mnist/invocations`, the route rules above were likely not applied.
* If you see `Endpoint for model_name 'mnist' cannot be found.`, this is OK!  We will try `localhost` instead.
* See [Troubleshooting](/docs/troubleshooting) for more debugging info.

**Expected Output**
* You should see a 100% traffic to Model Version a, however Model Version b is receiving a "best effort" amount of live traffic. (See Dashboards to verify.)

[**CPU (version a)**](https://github.com/PipelineAI/models/tree/f559987d7c889b7a2e82528cc72d003ef3a34573/tensorflow/a)
```
('{"variant": "mnist-a-tensorflow-tfserving-cpu", "outputs":{"classes": [3], '
 '"probabilities": [[2.353575155211729e-06, 3.998300599050708e-06, '
 '0.00912125688046217, 0.9443341493606567, 3.8211437640711665e-06, '
 '0.0003914404078386724, 5.226673920333269e-07, 2.389515998402203e-07, '
 '0.04614224657416344, 8.35775360030766e-09]]}}')
 
Request time: 36.414 milliseconds
``` 

### Install Dashboards
**Prometheus**
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/0.6.0/install/kubernetes/addons/prometheus.yaml
```
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &   
```
Verify `prometheus-...` pod is running
```
kubectl get pod --namespace=istio-system
```
```
http://localhost:9090/graph 
```

**Grafana**
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/0.6.0/install/kubernetes/addons/grafana.yaml
```

Verify `grafana-...` pod is running
```
kubectl get pod --namespace=istio-system
```

Open localhost proxy
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

View dashboard
```
http://localhost:3000/dashboard/db/istio-dashboard
```

**Service Graph**
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/0.6.0/install/kubernetes/addons/servicegraph.yaml
```

Verify `grafana-...` pod is running
```
kubectl get pod --namespace=istio-system
```

Open localhost proxy
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 &   
```

View dashboard (60s window)
```
http://localhost:8088/dotviz?filter_empty=true&time_horizon=60s
```

**PipelineAI Real-Time Dashboard**

Deploy PipelineAI-specific [NetflixOSS Hystrix](https://github.com/netflix/hystrix)
```
kubectl create -f https://raw.githubusercontent.com/PipelineAI/dashboards/1.5.0/hystrix-deploy.yaml
```
```
kubectl create -f https://raw.githubusercontent.com/PipelineAI/dashboards/1.5.0/hystrix-svc.yaml
```
Open up permissions for turbine to talk with hystrix
```
kubectl create clusterrolebinding serviceaccounts-view \
  --clusterrole=view \
  --group=system:serviceaccounts
```
Deploy PipelineAI-specific [NetflixOSS Turbine](https://github.com/netflix/turbine)
```
kubectl create -f https://raw.githubusercontent.com/PipelineAI/dashboards/1.5.0/turbine-deploy.yaml
```
```
kubectl create -f https://raw.githubusercontent.com/PipelineAI/dashboards/1.5.0/turbine-svc.yaml
```

View the model server dashboard (60s window)
```
http://localhost:7979/hystrix-dashboard/monitor/monitor.html?streams=%5B%7B%22name%22%3A%22%22%2C%22stream%22%3A%22http%3A%2F%2Fdashboard-turbine%3A8989%2Fturbine.stream%22%2C%22auth%22%3A%22%22%2C%22delay%22%3A%22%22%7D%5D
```

### Re-Run Load Test on Model Version a and Version b
Notes:
* You may need to wait 30 secs for the 2nd replica to start up completely.
```
pipeline predict-kube-test --model-name=mnist --test-request-path=./tensorflow/mnist-cpu/input/predict/test_request.json --test-request-concurrency=1000
```

### Predict with REST API
Use the REST API to POST a JSON document representing the number 2.

![MNIST 2](http://pipeline.ai/assets/img/mnist-2-100x101.png)
```
curl -X POST -H "Content-Type: application/json" \
  -d '{"image": [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.05098039656877518, 0.529411792755127, 0.3960784673690796, 0.572549045085907, 0.572549045085907, 0.847058892250061, 0.8156863451004028, 0.9960784912109375, 1.0, 1.0, 0.9960784912109375, 0.5960784554481506, 0.027450982481241226, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.7882353663444519, 0.11764706671237946, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.988235354423523, 0.7921569347381592, 0.9450981020927429, 0.545098066329956, 0.21568629145622253, 0.3450980484485626, 0.45098042488098145, 0.125490203499794, 0.125490203499794, 0.03921568766236305, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.803921639919281, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6352941393852234, 0.9921569228172302, 0.803921639919281, 0.24705883860588074, 0.3490196168422699, 0.6509804129600525, 0.32156863808631897, 0.32156863808631897, 0.1098039299249649, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.007843137718737125, 0.7529412508010864, 0.9921569228172302, 0.9725490808486938, 0.9686275124549866, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.8274510502815247, 0.29019609093666077, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.2549019753932953, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.847058892250061, 0.027450982481241226, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.5921568870544434, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.7333333492279053, 0.44705885648727417, 0.23137256503105164, 0.23137256503105164, 0.4784314036369324, 0.9921569228172302, 0.9921569228172302, 0.03921568766236305, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.5568627715110779, 0.9568628072738647, 0.7098039388656616, 0.08235294371843338, 0.019607843831181526, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.43137258291244507, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.15294118225574493, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.1882353127002716, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6705882549285889, 0.9921569228172302, 0.9921569228172302, 0.12156863510608673, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.2392157018184662, 0.9647059440612793, 0.9921569228172302, 0.6274510025978088, 0.003921568859368563, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08235294371843338, 0.44705885648727417, 0.16470588743686676, 0.0, 0.0, 0.2549019753932953, 0.9294118285179138, 0.9921569228172302, 0.9333333969116211, 0.27450981736183167, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.4941176772117615, 0.9529412388801575, 0.0, 0.0, 0.5803921818733215, 0.9333333969116211, 0.9921569228172302, 0.9921569228172302, 0.4078431725502014, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.7411764860153198, 0.9764706492424011, 0.5529412031173706, 0.8784314393997192, 0.9921569228172302, 0.9921569228172302, 0.9490196704864502, 0.43529415130615234, 0.007843137718737125, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6235294342041016, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9764706492424011, 0.6274510025978088, 0.1882353127002716, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.18431372940540314, 0.5882353186607361, 0.729411780834198, 0.5686274766921997, 0.3529411852359772, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]}' \
  http://localhost:80/predict/mnist/invocations \
  -w "\n\n"

### Expected Output ###
{"variant": "mnist-a-tensorflow-tfserving-cpu", "outputs":{"outputs": [0.11128007620573044, 1.4478533557849005e-05, 0.43401220440864563, 0.06995827704668045, 0.0028081508353352547, 0.27867695689201355, 0.017851119861006737, 0.006651509087532759, 0.07679300010204315, 0.001954273320734501]}}

### Formatted Output
Digit  Confidence
=====  ==========
0      0.0022526539396494627
1      2.63791100074684e-10
2      0.4638307988643646      <-- Prediction
3      0.21909376978874207
4      3.2985670372909226e-07
5      0.29357224702835083 
6      0.00019597385835368186
7      5.230629176367074e-05
8      0.020996594801545143
9      5.426473762781825e-06
```
Notes:
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7.
* If you're having trouble, see our [Troubleshooting](/docs/troubleshooting) Guide.

### (Optional) GPUs
[**GPU**](https://github.com/PipelineAI/models/tree/f559987d7c889b7a2e82528cc72d003ef3a34573/tensorflow/mnist-gpu)

**Build**
```
pipeline predict-server-build --model-name=mnist --model-tag=gpu --model-type=tensorflow --model-path=./tensorflow/mnist-gpu/model --model-chip=gpu 
```
* For GPU-based models, make sure you specify `--model-chip=gpu`

**Start**
```
pipeline predict-kube-start --model-name=mnist --model-tag=gpu --model-chip=gpu
```
* For GPU-based models, make sure you specify `--model-chip=gpu`

### Distributed TensorFlow Training (CPU)
We assume you already have the following:
* Kubernetes Cluster running anywhere! 
* Access to a shared file system like S3 or GCS

[**CPU**](https://github.com/PipelineAI/models/tree/master/tensorflow/census-cpu)

**Build Training Docker Image**
```
pipeline train-server-build --model-name=census --model-tag=cpu --model-type=tensorflow --model-path=./tensorflow/census-cpu/model/ 
```
Notes:
* If you change the model (`pipeline_train.py`), you'll need to re-run `pipeline train-server-build ...`
* `--model-path` must be relative to the current ./models directory (cloned from https://github.com/PipelineAI/models)
* For GPU-based models, make sure you specify `--model-chip=gpu`

**Register Training Docker Image with Docker Repo**
* By default, we use the following public DockerHub repo `docker.io/pipelineai`
* By convention, we use `train-` to namespace our model servers (ie. `train-census`)
* To use your own defaults or conventions, specify `--image-registry-url`, `--image-registry-repo`, or `--image-registry-namespace`
```
pipeline train-server-register --model-name=census --model-tag=cpu 
```

**Start Distributed TensorFlow Training Cluster**
```
pipeline train-kube-start --model-name=census --model-tag=cpu --input-host-path=/ignore/this/for/now --output-host-path=/root/samples/models/tensorflow/census-cpu/model --master-replicas=1 --ps-replicas=1 --worker-replicas=1 --train-args="--train-files=/root/samples/models/tensorflow/census-cpu/input/training/adult.training.csv --eval-files=/root/samples/models/tensorflow/census-cpu/input/validation/adult.validation.csv --num-epochs=2 --learning-rate=0.025"
```

Notes:
* If you change the model (`pipeline_train.py`), you'll need to re-run `pipeline train-server-build ...`
* `--input-host-path` and `--output-host-path` are host paths (outside the Docker container) mapped inside the Docker container as `/opt/ml/input` (PIPELINE_INPUT_PATH) and `/opt/ml/output` (PIPELINE_OUTPUT_PATH) respectively.
* PIPELINE_INPUT_PATH and PIPELINE_OUTPUT_PATH are environment variables accesible by your model inside the Docker container. 
* PIPELINE_INPUT_PATH and PIPELINE_OUTPUT_PATH are hard-coded to `/opt/ml/input` and `/opt/ml/output`, respectively, inside the Docker conatiner .
* `--input-host-path` and `--output-host-path` should be absolute paths that are valid on the HOST Kubernetes Node
* Avoid relative paths for * `--input-host-path` and `--output-host-path` unless you're sure the same path exists on the Kubernetes Node 
* If you use `~` and `.` and other relative path specifiers, note that `--input-host-path` and `--output-host-path` will be expanded to the absolute path of the filesystem where this command is run - this is likely not the same filesystem path as the Kubernetes Node!
* `--input-host-path` and `--output-host-path` are available outside of the Docker container as Docker volumes
* `--train-args` is used to specify `--train-files` and `--eval-files` and other arguments used inside your model 
* Inside the model, you should use PIPELINE_INPUT_PATH (`/opt/ml/input`) as the base path for the subpaths defined in `--train-files` and `--eval-files`
* We automatically mount `https://github.com/PipelineAI/models` as `/root/samples/models` for your convenience
* You can use our samples by setting `--input-host-path` to anything (ignore it, basically) and using an absolute path for `--train-files`, `--eval-files`, and other args referenced by your model
* You can specify S3 buckets/paths in your `--train-args`, but the host Kubernetes Node needs to have the proper EC2 IAM Instance Profile needed to access the S3 bucket/path
* Otherwise, you can specify ACCESS_KEY_ID and SECRET_ACCESS_KEY in your model code (not recommended_
* `--train-files` and `--eval-files` can be relative to PIPELINE_INPUT_PATH (`/opt/ml/input`), but remember that PIPELINE_INPUT_PATH is mapped to PIPELINE_HOST_INPUT_PATH which must exist on the Kubernetes Node where this container is placed (anywhere)
* `--train-files` and `--eval-files` are used by the model, itself
* You can pass any parameter into `--train-args` to be used by the model (`pipeline_train.py`)
* `--train-args` is a single argument passed into the `pipeline_train.py`
* Models, logs, and event are written to `--output-host-path` (or a subdirectory within it).  These paths are available outside of the Docker container.
* To prevent overwriting the output of a previous run, you should either 1) change the `--output-host-path` between calls or 2) create a new unique subfolder within `--output-host-path` in your `pipeline_train.py` (ie. timestamp).
* Make sure you use a consistent `--output-host-path` across nodes.  If you use timestamp, for example, the nodes in your distributed training cluster will not write to the same path.  You will see weird ABORT errors from TensorFlow.
* On Windows, be sure to use the forward slash `\` for `--input-host-path` and `--output-host-path` (not the args inside of `--train-args`).
* If you see `port is already allocated` or `already in use by container`, you already have a container running.  List and remove any conflicting containers.  For example, `docker ps` and/or `docker rm -f train-mnist-cpu-tensorflow-tfserving-cpu`.
* For GPU-based models, make sure you specify `--start-cmd=nvidia-docker` - and make sure you have `nvidia-docker` installed!
* For GPU-based models, make sure you specify `--model-chip=gpu`
* If you're having trouble, see our [Troubleshooting](/docs/troubleshooting) Guide.

(_We are working on making this more intuitive._)

### Distributed TensorFlow Training (GPU)

[**GPU**](https://github.com/PipelineAI/models/tree/master/tensorflow/census-gpu)

**Build Docker Image**
```
pipeline train-server-build --model-name=census --model-tag=gpu --model-type=tensorflow --model-path=./tensorflow/census-gpu/model/ --model-chip=gpu 
```
Notes:
* If you change the model (`pipeline_train.py`), you'll need to re-run `pipeline train-server-build ...`
* For GPU-based models, make sure you specify `--model-chip=gpu`

**Register Image with Docker Repo**
* By default, we use the following public DockerHub repo `docker.io/pipelineai`
* By convention, we use `train-` to namespace our models (ie. `train-census-gpu`)
* To use your own defaults or conventions, specify `--image-registry-url`, `--image-registry-repo`, or `--image-registry-namespace`
```
pipeline train-server-register --model-name=census --model-tag=gpu 
```

**Start Distributed TensorFlow Training Cluster**
```
pipeline train-kube-start --model-name=census --model-tag=gpu --input-host-path=/ignore/this/for/now --output-host-path=/root/samples/models/tensorflow/census-gpu/model --master-replicas=1 --ps-replicas=1 --worker-replicas=1 --train-args="--train-files=/root/samples/models/tensorflow/census-gpu/input/training/adult.training.csv --eval-files=/root/samples/models/tensorflow/census-gpu/input/validation/adult.validation.csv --num-epochs=2 --learning-rate=0.025" --model-chip=gpu
```
Notes:
* If you change the model (`pipeline_train.py`), you'll need to re-run `pipeline train-server-build ...`
* For GPU-based models, make sure you specify `--model-chip=gpu`

### Clean Up

**Uninstall PipelineAI Models**

Notes:
* Each of these will remove the `predict-mnist` service and ingress - which is OK!
```
pipeline predict-kube-stop --model-name=mnist --model-tag=a
```
```
pipeline predict-kube-stop --model-name=mnist --model-tag=b
```

**Remove PipelineAI Dashboards**
```
kubectl delete deploy dashboard-turbine
kubectl delete deploy dashboard-hystrix
```
```
kubectl delete svc dashboard-turbine
kubectl delete svc dashboard-hystrix
```

**Remove PipelineAI Traffic Routes**
```
kubectl delete routerule predict-mnist-dashboardstream
kubectl delete routerule predict-mnist-denytherest
kubectl delete routerule predict-mnist-invocations
kubectl delete routerule predict-mnist-metrics
kubectl delete routerule predict-mnist-ping
kubectl delete routerule predict-mnist-prometheus
```

**Uninstall Istio Components**
```
kubectl delete namespace istio-system
```
