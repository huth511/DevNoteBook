## 参考

### MapReduce

> [Development of a distributed computing system based on MapReduce and Kubernetes | by Digital Wing | Digital Wing | digitalwing.co | Medium](https://medium.com/digitalwing/development-of-a-distributed-computing-system-based-on-mapreduce-and-kubernetes-837fc7f112f9)

#### Master

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mapreduce-master
spec:
  selector:
    app: mapreduce-master
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: mapreduce-master
  labels:
    app: mapreduce-master
spec:
  containers:
  - name: mapreduce-master
    image: temakozyrev/mapreduce:1.14
    ports:
    - containerPort: 8080
      name: mapper
    env:
    - name: TYPE
      value: "MASTER"
    - name: MAPPER_HOST
      value: "mappers"
    - name: REDUCER_HOST
      value: "reducers"
    - name: MAPPER_PORT
      value: "8080"
    - name: REDUCER_PORT
      value: "8080"
```

#### Worker

```yaml
apiVersion: v1
kind: Service
metadata:
  name: reducers
  labels:
    app: reducers
spec:
  ports:
  - port: 8080
    name: reducer
  clusterIP: None
  selector:
    app: reducers
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: reducer
spec:
  serviceName: "reducers"
  replicas: 3
  selector:
    matchLabels:
      app: reducers
  template:
    metadata:
      labels:
        app: reducers
    spec:
      containers:
      - name: mapreduce
        image: temakozyrev/mapreduce:1.14
        ports:
        - containerPort: 8080
          name: reducer
        env:
        - name: TYPE
          value: "REDUCE"
```

## Tess

### V1

```yaml
# master
apiVersion: v1
kind: Service
metadata:
  name: tess-ds-master-svc
  namespace: tess
spec:
  selector:
    app: tess-ds-master-app
  ports:
  - protocol: TCP
    port: 31100
    targetPort: 31100
  - protocol: TCP
  	port: 7788
    targetPort: 31200
---
apiVersion: v1
kind: Pod
metadata:
  name: tess-ds-master
  namespace: tess
  labels:
    app: tess-ds-master-app
spec:
  containers:
  - name: tess-ds-master-container
    image: tessng:ds-test
    ports:
    - containerPort: 31100
    - containerPort: 7788
    env:
    - name: TESS_DS_STATUS
      value: "master"
    - name: TESS_DS_MASTER_PORT
      value: "31100"
    - name: TESS_DS_WORKER_PORT
      value: "31101"
    - name: TESS_DS_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: TESS_DS_MASTER_ASSIGN_NODE_ID
      value: 1
    - name: TESS_DS_MASTER_PART_CNT
      value: 3
    - name: TESS_DS_MASTER_CUT_METHOD
      value: "metis"
    - name: TESS_DS_KAFKA_ADDR
      value: "192.168.1.49:9092"
    - name: TESS_DS_KAFKA_TOPIC
      value: "tess-ds-trace"
    - name: TESS_DS_TRACE_FORMAT
      value: "default"
      
  - name: tess-ds-combined-container
    image: tessng:ds-client
    env:
    - name: TESS_DS_MASTER_PORT
      value: "31100"
    - name: TESS_DS_COMBINED_KAFKA_ADDR
      value: "192.168.1.49:9092"
    - name: TESS_DS_COMBINED_KAFKA_TOPIC
      value: "tess-trace"
    - name: TESS_DS_COMBINED_MSG_TYPE
      value: "json"
      
---
# worker x 3
apiVersion: v1
kind: Service
metadata:
  name: tess-ds-worker-svc
  namespace: tess
  labels:
    app: tess-ds-svc
spec:
  ports:
  - port: 31101
  clusterIP: None
  selector:
    app: tess-ds-workers-app
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tess-ds-workers
  namespace: tess
  labels:
    app: tess-ds-workers-app
spec:
  serviceName: "tess-ds-worker-svc"
  replicas: 3
  selector:
    matchLabels:
      app: tess-ds-workers-app
  template:
    metadata:
      labels:
        app: tess-ds-workers-app
    spec:
      containers:
      - name: tess-ds-workers-container
        image: tessng:ds-test
        ports:
        - containerPort: 31101
        env:
        - name: TESS_DS_STATUS
          value: "worker"
        - name: TESS_DS_MASTER_PORT
          value: "31100"
        - name: TESS_DS_WORKER_PORT
          value: "31101"
        - name: TESS_DS_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: TESS_DS_MASTER_SVC_NAME
          value: "TESS_DS_MASTER_SVC"
        - name: TESS_DS_MASTER_ASSIGN_NODE_ID
          value: 1
        - name: TESS_DS_KAFKA_ADDR
          value: "192.168.1.49:9092"
        - name: TESS_DS_KAFKA_TOPIC
          value: "tess-ds-trace"
        - name: TESS_DS_TRACE_FORMAT
          value: "default"
```

