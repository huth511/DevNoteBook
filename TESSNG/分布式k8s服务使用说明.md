## 分布式k8s服务使用说明

### 基本流程

#### 1. 准备yaml文件

见文件：[tess_ds（带注释说明）.yaml](https://www.teambition.com/project/6588df9bba1c1c0c2c8ae362/works/6588df9b4135aae7213d7f56/work/66827f31157dc4272175b3d8)

- 包括：
  - 一个master pod
  - 一个master service
  - 一个worker的StatefulSet
  - 一个worker的service
- 带注释的部分，是需要关注的，一般是需要配置的
- k8s的一些基本字段没写说明，但也是需要配置的，如命名空间，服务名，pod名等等

#### 2. 修改yaml文件

根据需要，进行配置

#### 3. 启动

使用jida saas启动服务的通常方法即可

#### 4. 启动一次仿真

使用在线接口启动

#### 5. 停止一次仿真

- 使用在线接口关闭
- 并清理redis数据，key前缀为：`[TESS_DS_PROJ_ID]:[TESS_DS_PLAN_ID]`（关键字解释见yaml文件）：
  - 确保清理干净，否则可能会造成前后两次仿真信息混杂
  - 若master的快照保留未删，服务启动后，会进入恢复模式

### 具体使用

#### 常态仿真

##### 配置yaml

- 需要注意的：
  - 应该需要配置master的环境变量：`TESS_DS_MASTER_SPEC_PART_CNT`和worker的statefulset中的spec:replicas。因为分区数一般是确定的。
- 其它，见yaml文件中注释的说明

#### 临时仿真

- 使用空闲分布式服务启动仿真
- 通过在线指令，配置分区数（worker数）【待定】

#### 一些注意点

- 判断启动：
  - 返回的global数据中，`running`字段为true时，代表分布式仿真已开始。
- 启动之后：
  - 消费kafka数据，**不要提前**（或者，通过拉取kafka topic的meta数据，判断topic是否创建，并且分区数符合预期）。
- 适时重启  在一段时间内空闲的  分布式服务
- 停止和重启仿真：
  - 停止的过程可能会比单机版的慢一点，理应通过global判断`running`字段是否为false，来确定是否停止。
  - 重启：稳健的做法，最好是在仿真停止之后，再发送启动仿真指令。

