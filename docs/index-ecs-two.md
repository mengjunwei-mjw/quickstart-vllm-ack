# 基于双ECS实例的DeepSeek-R1和V3模型部署文档

## 部署说明
本服务提供了基于ECS镜像+Vllm+Ray的大模型一键部署方案，30分钟即可通过双ECS实例部署使用DeepSeek-R1满血版和DeepSeek-V3模型。

本服务通过ECS镜像打包标准环境，通过Ros模版实现云资源与大模型的一键部署，开发者无需关心模型部署运行的标准环境与底层云资源编排，仅需添加几个参数即可享受DeepSeek-R1满血版和DeepSeek-V3的推理体验。

本服务提供的方案下，以平均每次请求的token为10kb计算，采用两台H20规格的ECS实例，DeepSeek-R1满血版理论可支持的每秒并发请求数(QPS)约为75，DeepSeek-V3约为67。


## 整体架构
![arch-ecs-two.png](arch-ecs-two.png)


## 计费说明
本服务在阿里云上的费用主要涉及：
* 所选GPU云服务器的规格
* 节点数量
* 磁盘容量
* 公网带宽
计费方式：按量付费（小时）或包年包月
预估费用在创建实例时可实时看到。


## RAM账号所需权限

部署服务实例，需要对部分阿里云资源进行访问和创建操作。因此您的账号需要包含如下资源的权限。

| 权限策略名称                          | 备注                         |
|---------------------------------|----------------------------|
| AliyunECSFullAccess             | 管理云服务器服务（ECS）的权限           |
| AliyunVPCFullAccess             | 管理专有网络（VPC）的权限             |
| AliyunROSFullAccess             | 管理资源编排服务（ROS）的权限           |
| AliyunComputeNestUserFullAccess | 管理计算巢服务（ComputeNest）的用户侧权限 |

## 部署流程

1. 单击[部署链接](https://computenest.console.aliyun.com/service/instance/create/cn-hangzhou?type=user&ServiceName=LLM推理服务(ECS版))。选择双机版，并确认已申请H20实例规格。根据界面提示填写参数，可根据需求选择是否开启公网，可以看到对应询价明细，确认参数后点击**下一步：确认订单**。
    ![deploy-ecs-two-1.png](deploy-ecs-two-1.png)
    ![deploy-ecs-one-2.png](deploy-ecs-one-2.png)
2. 点击**下一步：确认订单**后可以看到价格预览，随后可点击**立即部署**，等待部署完成。(提示RAM权限不足时需要为子账号添加RAM权限)
    ![price-ecs-two.png](price-ecs-two.png)
3. 等待部署完成后，就可以开始使用服务了。点击服务实例名称，进入服务实例详情，使用Api调用示例即可访问服务。如果是内网访问，需保证ECS实例在同一个VPC下。
    ![deploying-ecs-two.png](deploying-ecs-two.png)
    ![result-ecs-two-1.png](result-ecs-two-1.png)
    ![result-ecs-two-2.png](result-ecs-two-2.png)
4. ssh访问ECS实例后，执行 docker logs vllm 即可查询模型服务部署日志。当您看到下图所示结果时，表示模型服务部署成功。模型所在路径为/root/llm_model/。
   ![deployed.png](deployed.png)

## 使用说明

### 查询模型部署参数

1. 复制服务实例名称。到[资源编排控制台](https://ros.console.aliyun.com/cn-hangzhou/stacks)查看对应的资源栈。
    ![ros-stack-1.png](ros-stack-1.png)
    ![ros-stack-2.png](ros-stack-2.png)
2. 进入服务实例对应的资源栈，可以看到所开启的全部资源，并查看到模型部署过程中执行的全部脚本。
    ![ros-stack-3.png](ros-stack-3.png)

### 内网API访问
复制Api调用示例，在资源标签页的ECS实例中粘贴Api调用示例即可。也可在同一VPC内的其他ECS中访问。
    ![result-ecs-two-2.png](result-ecs-two-2.png)
    ![private-ip-ecs-two-1.png](private-ip-ecs-two-1.png)
    ![private-ip-ecs-two-2.png](private-ip-ecs-two-2.png)
### 公网API访问
复制Api调用示例，在本地终端中粘贴Api调用示例即可。
    ![result-ecs-two-2.png](result-ecs-two-2.png)
    ![public-ip-ecs-two-1.png](public-ip-ecs-two-1.png)

## 性能测试
本服务方案下，针对Deepseek-R1和V3，分别测试QPS为75和60情况下模型服务的推理响应性能，压测持续时间均为20s。

### Deepseek-R1
#### QPS为75
![qps75-r1-ecs-two.png](qps75-r1-ecs-two.png)

### Deepseek-V3
#### QPS为60
![qps60-v3-ecs-two.png](qps60-v3-ecs-two.png)

### 压测过程(供参考)
>**前提条件：** 1. 无法直接测试带api-key的模型服务，需要修改benchmark_serving.py使其允许传入api-key；2. 需要公网。
>

以Deepseek-R1为例，模型服务部署完成后，ssh登录ECS实例。执行下面的命令，即可得到模型服务性能测试结果。
    
    yum install -y git-lfs
    git lfs install
    git lfs clone https://www.modelscope.cn/datasets/gliang1001/ShareGPT_V3_unfiltered_cleaned_split.git
    git lfs clone https://github.com/vllm-project/vllm.git
    
    docker exec vllm bash -c "
    pip install pandas datasets &&
    python3 /root/vllm/benchmarks/benchmark_serving.py \
    --backend vllm \
    --model /root/llm-model/deepseek-ai/DeepSeek-R1 \
    --served-model-name deepseek-ai/DeepSeek-R1 \
    --sonnet-input-len 1024 \
    --sonnet-output-len 6 \
    --sonnet-prefix-len 50 \
    --num-prompts 1500 \
    --request-rate 75 \
    --port 8080 \
    --trust-remote-code \
    --dataset-name sharegpt \
    --save-result \
    --dataset-path /root/ShareGPT_V3_unfiltered_cleaned_split/ShareGPT_V3_unfiltered_cleaned_split.json
    "