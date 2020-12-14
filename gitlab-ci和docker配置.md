## 需求说明
1. 实现当开发提交一个`MR`时触发`CI`任务
2. 通过`Dockerfile`创建`docker`镜像
    - 还原依赖
    - 编译`net core`程序，确定是否正常
    - 生成发布包
    - 将发布包拷贝到将要生成的`docker`镜像中
    - 完成
3. `gitlab-runner`使用`docker`启动一个该镜像的容器，并启动容器内的`net core`程序
4. 在`MR`界面生成预览地址的按钮，点击即可跳转到`net core`程序内容
5. 当`MR`归并后自动停止容器

## 创建`gitlab-ci.yml`
说明：`gitlab-runner`的`executor`类型为`shell`。并且在`gitlab-runner`所在的机器上必须安装好`git`、`docker`等必要组件。
#### 确定`CI`的执行阶段
- 创建镜像
- 启动容器
- 停止容器

大概`gitlab-ci.yml`的文件应该长这样
```yml
stages:
  - build-image
  - docker-review

build-image:
  stage: build-image
  script:
    - echo "build image"
  
docker-review:
  stage: docker-review
  script:
    - echo "review"
```
因为只有当开发提交`MR`才会触发，所以需要给相应的阶段添加`only`标记为`merge_requests`，并且需要可以手动停止。

_说明：`merge_requests`标记的使用需要用新版的`gitlab-runner`_

大概`gitlab-ci.yml`的文件应该长这样
```yml
stages:
  - build-image
  - docker-review

build-image:
  stage: build-image
  script:
    - echo "build image"
  only:
    - merge_requests
  
docker-review:
  stage: docker-review
  script:
    - echo "review"
  only:
    - merge_requests

stop-docker-review:
  stage: docker-review
  script:
    - echo "stop review"
  only:
    - merge_requests
```
可以看到`docker-review`和`stop-docker-review`都属于同一阶段`docker-review`。
查阅`gitlab`官方文档[environment:action](https://docs.gitlab.com/ee/ci/yaml/README.html#environmentaction)后，补充`gitlab-ci.yml`来实现`docker-review`和`stop-docker-review`之间的关系以及`MR`界面的预览按钮。

```yml
stages:
  - build-image
  - docker-review

build-image:
  stage: build-image
  script:
    - echo "build image"
  only:
    - merge_requests
  
docker-review:
  stage: docker-review
  script:
    - echo "review"
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: https://$CI_ENVIRONMENT_SLUG.example.com
    on_stop: stop-docker-review
    action: start
  only:
    - merge_requests

stop-docker-review:
  stage: docker-review
  dependencies: []
  script:
    - echo "stop review"
  when: manual
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  only:
    - merge_requests
```
这里`environment:url`是预览的url地址，如果有域名的话，可以通过脚本或者其他方式生成动态域名，如果没有域名，我们需要把`docker`创建容器之后分配的端口拿过来，因为端口是随机生成的，所以我们这里需要提前生成端口，然后再将端口和地址传递给`environment:url`。在`linux`中生成一个随机端口可以用下面的方法(会有碰撞可能性)
```
// 随机生成一个50000-60000之前的整数
shuf -i 50000-60000 -n 1
```
因为`environment:url`在当前`job`开始后就会初始化，所以我们需要在当前`job`的上一个阶段就知道那个随机端口号，此时我们需要用`artifacts:reports:dotenv`来继承环境变量。[artifacts:reports:dotenv](https://docs.gitlab.com/ee/ci/pipelines/job_artifacts.html#artifactsreportsdotenv)的官方文档。
所以我们的脚本就变成了下面这样
```yml
stages:
  - build-image
  - docker-review

build-image:
  stage: build-image
  script:
    - echo "build image"
  after-script:
    # 生成随机端口
    - REVIEW_PORT=$(shuf -i 50000-60000 -n 1)
    # 将端口号写入build-image.env中，并将环境变量命名为REVIEW_PORT
    - echo "REVIEW_PORT=$REVIEW_PORT" >> build-image.env
  artifacts:
    reports:
      # 上传环境变量
      dotenv: build-image.env
    when: on_success
    expire_in: 10 mins
  only:
    - merge_requests
  
docker-review:
  stage: docker-review
  script:
    - echo "review"
  environment:
    name: review/$CI_COMMIT_REF_NAME
    # 此时的REVIEW_PORT就是从上一个job中继承过来的值了。
    url: http://$CI_REVIEW_HOST:$REVIEW_PORT/swagger/index.html
    on_stop: stop-docker-review
    action: start
  only:
    - merge_requests

stop-docker-review:
  stage: docker-review
  dependencies: []
  script:
    - echo "stop review"
  when: manual
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  only:
    - merge_requests
```
#### 构建Docker镜像
接下来就是编译`docker`镜像以及发布程序了。为了能尽量减小`docker`镜像的大小，所以使用`Dockerfile`来构建镜像。
```
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim AS base-img
WORKDIR /app
EXPOSE 5000
EXPOSE 443

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build-env
WORKDIR /src
# 将所有的工程文件复制到容器当前的工作目录
COPY ./**/*.csproj ./
# 在Linux中通过命令将容器当前目录的工程文件移动到相应名字的文件夹下。
RUN for file in $(ls *.csproj); do mkdir -p ./${file%.*}/ && mv $file ./${file%.*}/; done
RUN dotnet restore "Test.Api/Test.Api.csproj"
COPY . .
WORKDIR /src/Test.Api
RUN dotnet build "Test.Api.csproj" -c Release -r ubuntu.18.04-x64 -o /app/build

FROM build-env AS publish
RUN dotnet publish "Test.Api.csproj" -c Release -r ubuntu.18.04-x64 -o /app/publish

from base-img as final
WORKDIR /app
COPY --from=publish /app/publish .
ENV ASPNETCORE_ENVIRONMENT="Development"
ENTRYPOINT ["dotnet", "Test.Api.dll"]
```
说明：项目的目录结构为
```
.sln(解决方案)
 |--Test.api 
   |-- Test.api.csproj
 |--Test.service 
   |-- Test.service.csproj
 |--Test.common
   |-- Test.common.csproj
```
  
所以在复制所有的工程文件后，需要将相应的工程文件再转移到相应的文件夹中才行。
```
# 将所有的工程文件复制到容器当前的工作目录
COPY ./**/*.csproj ./
# 在Linux中通过命令将容器当前目录的工程文件移动到相应名字的文件夹下。
RUN for file in $(ls *.csproj); do mkdir -p ./${file%.*}/ && mv $file ./${file%.*}/; done
```
所以最终的`gitlab-ci.yml`文件如下
```
variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_MERGE_REQUEST_ID
  REVIEW_NAME: $CI_PROJECT_NAME-$CI_ENVIRONMENT_SLUG
  
stages:
  - build-image
  - docker-review

build-image:
  stage: build-image
  before_script: 
    # 这里是为了清空在构建docker镜像时产生的悬空镜像。也就是<none>这种的
    - if [[ "$(docker images -q $IMAGE_NAME 2> /dev/null)" != "" ]]; then docker rmi -f $IMAGE_NAME; fi
  script:
    - docker build -t $IMAGE_NAME .
  after-script:
    # 生成随机端口
    - REVIEW_PORT=$(shuf -i 50000-60000 -n 1)
    # 将端口号写入build-image.env中，并将环境变量命名为REVIEW_PORT
    - echo "REVIEW_PORT=$REVIEW_PORT" >> build-image.env
    # 这里再清空一次
    - if [[ "$(docker images -f "dangling=true" -q)" != "" ]]; then docker rmi $(docker images -f "dangling=true" -q); fi
  artifacts:
    reports:
      # 上传环境变量
      dotenv: build-image.env
    when: on_success
    expire_in: 10 mins
  tags:
    - docker
    - linux
  only:
    - merge_requests
  
docker-review:
  stage: docker-review
  dependencies:
    # 依赖一下build-image，因为只有在创建完镜像才能相应的启动容器 
    - build-image
  before_script: 
    # 这里是停掉之前运行的容器并删除（节省空间啊）
    - docker stop $REVIEW_NAME || true && docker rm $REVIEW_NAME || true
  script:
    # 这里就不做过多解释了啊，常规操作，只不过是先切换一下docker的工作目录-w，然后后台运行
    - "docker run -di -p $REVIEW_PORT:5000 --name=$REVIEW_NAME -w /app --restart=always -e ASPNETCORE_ENVIRONMENT=Development $IMAGE_NAME nohup dotnet Test.Api.dll &"
  environment:
    name: review/$CI_COMMIT_REF_NAME
    # 此时的REVIEW_PORT就是从上一个job中继承过来的值了。
    url: http://$CI_REVIEW_HOST:$REVIEW_PORT/swagger/index.html
    on_stop: stop-docker-review
    action: start
  tags:
    - docker
    - linux
  only:
    - merge_requests

stop-docker-review:
  stage: docker-review
  # 这里不需要再依赖build-image了，所以直接清空
  dependencies: []
  script:
    # 停止并删除容器，这个地方除了手动，当MR被归并后也会执行。
    - docker stop $REVIEW_NAME || true && docker rm $REVIEW_NAME || true
  when: manual
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  tags:
    - docker
    - linux
  only:
    - merge_requests
```

到此就完成了，基本能满足日常的开发和测试了。
