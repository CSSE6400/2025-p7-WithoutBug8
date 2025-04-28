## 这是week8的practice的中文文档

### 这周的主要内容

讲了消息队列的原理，如何使用，以及如何在AWS如何部署消息队列的方法

#### 详细讲一下docker-compose.yaml文件的具体内容
1. 首先是`services`这是最核心的部分,定义你要运行的每一个容器(服务)，如下所示app，database......都是服务的名字，可以随意取

   ```dockerfile
   services:
     app:
       ...
     database:
       ...
     redis:
       ...
     worker:
       ...
   ```

2. 每一个service下面，可以配置很多细节的参数，以下举例来说

   - build：指定如何从dockerfile构建镜像
   - image：使用现成的镜像，而不是自己build
   - restart：自动重启
   - ports：端口映射(本机端口：容器端口)
   - environment：设置环境变量
   - volumes：挂载本地目录到容器内部
   - env_file: 从.env文件加载变量

3. volume，networks，configs等等变量

   - 跨服务共享资源的配置

#### Terraform的一些配置讲解

1. terraform块

   - 作用：指定 Terraform 的基本配置，比如 **需要哪些 Provider**。

   - 举例来说：

     - required_providers：声明要用的 Provider，比如 aws、docker，指定来源和版本

     ```yaml
     terraform {
       required_providers {
         aws = {
           source  = "hashicorp/aws"
           version = "~> 4.0"
         }
         docker = {
           source  = "kreuzwerker/docker"
           version = "3.0.2"
         }
       }
     }
     ```

2. provider块

   - 作用：配置 **连接云平台的方式**，比如 AWS 区域、认证信息等。
   - 每个 Provider 都要有自己的配置。
   - 可以设置多个 provider，如果有多个的话需要加 alias

   ```yaml
   provider "aws" {
     region                  = "us-east-1"
     shared_credentials_files = ["./credentials"]
     default_tags {
       tags = {
         Course     = "CSSE6400"
         Name       = "CoughOverflow"
         Automation = "Terraform"
       }
     }
   }
   ```

   

3. resource块

   - 作用：真正**创建新的资源**。
   - 格式是：resource "资源类型" "资源名称" {}。
   - 每个 resource 里配置对应的参数，比如 ECR 仓库、EC2 实例、S3 桶等。
   - **resource后面跟的两个参数**，第一个是**资源类型**，第二个是**资源的本地名字**也就是在terraform其他地方引用的时候，可以使用这个名字来引用

   ```yaml
   resource "aws_ecr_repository" "csse6400-dingzhen-ecr" {
     name = "csse6400-dingzhen-ecr"
   }
   ```

4. data块

   - 作用：**读取已经存在的资源**，而不是新建。
   - 格式是：data "资源类型" "引用名称" {}。
   - 常用于查找已有的 VPC、子网、IAM 角色等信息。

   ```yaml
   data "aws_vpc" "default" {
     default = true
   }
   ```

5. locals块

   - 作用：**定义本地变量**，简化代码，避免到处复制粘贴。
   - 可以在整个 .tf 文件中引用这些变量。
   - 特别适合管理镜像地址、数据库密码、重复使用的名字等等。

   ```yaml
   locals {
     image             = "${aws_ecr_repository.csse6400-dingzhen-ecr.repository_url}:latest"
     database_username = "dingzhen"
     database_password = "postgresql123"
   }
   
   ```

   

#### 消息队列

1. 本周使用**celery**+队列实现了一个异步的任务系统，具体来说

   - celery：是一个任务队列的框架，可以把一些函数丢到后台去运行，而不是用户请求时立即完成

   - broker(队列)：是celery用于存放任务的中间人，比如Redis，RabbitMQ，SQS，Celery等等会把任务发送到队列里，worker再去队列里拿任务；举个例子来说

     - 就本次任务中，Flask是发起任务的人，Broker相当于快递中转站，worker是后台真正的工作进程，worker不断去broker中拉取任务，然后执行。

       ```python
       # 导入要用到的Celery包
       from todo.extensions import celery
       # 我们要使用Celery把要放到消息队列的函数注册成Task
       @celery.task(name="ical")
       def create_ical(todos):
           # 你的生成 ical 文件的代码
       ```

       ```python
       # Broker看到来了一个任务，要处理哪个队列queue
       # 在Celery中，routing_key 是你在发消息时附带的一串字符串，告诉broker（比如 Redis 或 RabbitMQ）这条消息应该放进哪个队列，下面的代码是默认发送给ical这个队列
       celery.conf.task_default_queue = os.environ.get("CELERY_DEFAULT_QUEUE","ical")
       
       
       # Worker负责从对应的Queue中拿去任务，然后调用Celery Task函数去执行
       # worker在docker-compose.yaml文件中，配置
       command: poetry run celery --app todo.tasks.ical worker --loglevel=info
       ```

       
