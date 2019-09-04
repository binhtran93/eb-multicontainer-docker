# Giới thiệu
Trong bài này sẽ hướng dẫn cách tạo môi trường mà một con EC2 có thể chạy nhiều container cùng một lúc
 
Hình dưới là 1 ví dụ cho thấy môi trường elastic beanstalk chạy với 3 container cho mỗi instance EC2 đặt bên trong Auto Scaling group

![](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/images/aeb-multicontainer-docker-example.png)

Trong bài hướng dẫn này, chúng ta sẽ dùng elasticbeanstalk sử dụng thiết lập multicontainer docker để deploy như sau:  
- Source code dùng laravel
- Nginx làm proxy
- Sử dụng supervisor để monitor laravel worker process

Cấu trúc dự án sẽ có dạng như sau 
```
├── Dockerrun.aws.json
├── .ebxtensions
|    └── 00_test.config
├── project
│   ├── app
│   └── public 
│   └── ...
└── proxy
│    └── conf.d
│        └── default.conf
└───docker  
     └── Dockerfile       
```   

Tham khảo toàn bộ cấu trúc dự án ở [đây](https://github.com/binhtran93/eb-multicontainer-docker)

# Tạo private image trên ECR
Trước tiên ta sẽ tạo 1 private repository trên ECR sử dụng Dockerfile trong thư mục docker

Nội dung Dockerfile
```
FROM php:7.2-fpm
RUN apt-get update && apt-get install -y \
    libzip-dev \
    zip \
    unzip \
    supervisor
RUN docker-php-ext-configure zip --with-libzip
RUN docker-php-ext-install pdo_mysql zip
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

RUN chown -R www-data:www-data /var/www
```

1. Tạo 1 repository tại [đây](https://us-west-2.console.aws.amazon.com/ecr/repositories). Kết quả sẽ tạo được một repository với URI ```360118472363.dkr.ecr.us-west-2.amazonaws.com/laravel```  
2. Sử dụng AWS CLI (nếu chưa có thì cài đặt tại [đây](https://docs.aws.amazon.com/cli/latest/userguide/install-linux-al2017.html)) để thực hiện các lệnh sau
 
-  Login aws cli 
    ```
    $(aws ecr get-login --no-include-email --region us-west-2 --profile <your profile>)
    ```
-  Tạo docker image từ docker/Dockerfile với tag laravel 
    ```
    docker build -t laravel .
    ```
- Tag lại image theo URI repository mới tạo ở trên 
    ```
    docker tag laravel:latest 360118472363.dkr.ecr.us-west-2.amazonaws.com/laravel:latest
    ```
- Push lên ECR 
    ```
    docker push 360118472363.dkr.ecr.us-west-2.amazonaws.com/laravel:latest
    ```

# Config laravel và Nginx 
Với thiết lập multicontainer cho EB, file Dockerrun.aws.json sẽ là bắt buộc, bên dưới sẽ là nội dung toàn bộ của file 
```
{
  "AWSEBDockerrunVersion": 2,
  "volumes": [
    {
      "name": "project",
      "host": {
        "sourcePath": "/var/app/current/project"
      }
    },
    {
      "name": "nginx-proxy-conf",
      "host": {
        "sourcePath": "/var/app/current/proxy/conf.d"
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "project",
      "image": "360118472363.dkr.ecr.us-west-2.amazonaws.com/laravel:latest",
      "essential": true,
      "memory": 128,
      "mountPoints": [
        {
          "sourceVolume": "project",
          "containerPath": "/var/www/html"
        }
      ]
    },
    {
      "name": "nginx-proxy",
      "image": "nginx",
      "essential": true,
      "memory": 256,
      "portMappings": [
        {
          "hostPort": 80,
          "containerPort": 80
        }
      ],
      "links": [
        "project"
      ],
      "mountPoints": [
        {
          "sourceVolume": "project",
          "containerPath": "/var/www/html"
        },
        {
          "sourceVolume": "nginx-proxy-conf",
          "containerPath": "/etc/nginx/conf.d",
          "readOnly": true
        },
        {
          "sourceVolume": "awseb-logs-nginx-proxy",
          "containerPath": "/var/log/nginx"
        }
      ]
    }
  ]
}
``` 

#### Giải thích về các trường trong file config
`AWSEBDockerrunVersion`: Giá trị bắt buộc phải là 2 

`Volumes`: Config đường dẫn các folder, mà sau này sẽ sử dụng để mount vào trong các container

`containerDefinitions`: Array chứa định nghĩa các containers, chi tiết giải thích bên dưới

`name`: Tên của container
 
`image`: Docker image, trong trường hợp này, chúng ta sử dụng ECR thay vì public image trên docker hub
 
`environment`: Thiết lập biến enviroment cho container tương ứng

`memory`: Hard limit thiệt lập cho container, khi container sử dụng quá lượng memory này, system sẽ kill container

`memoryReservation`: Soft limit thiết lập cho containter, phải nhỏ hơn `memory` 

`mountPoints`: Chỉ định mount các volume vào đường dẫn trong container

`links`: Các link container sẽ tìm được với nhau

`portMappings`: Map port trong container với port của host 


File config trên 
- Tạo 2 volume lần lượt tạo 2 volume và mount vào trong 2 container project và nginx-proxy
- Export port 80 của container nginx-proxy 
- Link container project với container nginx-proxy 

# Install thư viện và các lệnh cần thiết 
Chúng ta vẫn cần phải chạy lệnh composer install, lệnh migrate và khởi chạy worker trong laravel 

Sẽ tạo 1 file `00_test.config` trong .ebxtensions
```
commands:
  create_post_dir:
    command: "mkdir /opt/elasticbeanstalk/hooks/appdeploy/post"
    ignoreErrors: true
files:
  "/opt/elasticbeanstalk/hooks/appdeploy/post/99_laravel_deps.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/usr/bin/env bash
      DOCKER_ID=`docker ps -q --filter 'ancestor=360118472363.dkr.ecr.us-west-2.amazonaws.com/laravel:latest'`
      docker exec -i $DOCKER_ID chown -R www-data:www-data ./
      docker exec -i $DOCKER_ID composer install
      docker exec -i $DOCKER_ID php artisan migrate
      docker exec -i $DOCKER_ID supervisord
      docker exec -i $DOCKER_ID supervisorctl restart all
``` 

# Deploy 
### Trên local 
<b>eb local run</b>

Câu lệnh này sẽ generate 1 file docker-compose.yaml và chạy trực tiếp trên local, với điều kiện trên máy phải cài docker và docker-compose 

### Trên EB enviroment 
<b>eb init --profile `<your profile>`</b>
```
Select a default region
1) us-east-1 : US East (N. Virginia)
2) us-west-1 : US West (N. California)
3) us-west-2 : US West (Oregon)
4) eu-west-1 : EU (Ireland)
5) eu-central-1 : EU (Frankfurt)
6) ap-south-1 : Asia Pacific (Mumbai)
7) ap-southeast-1 : Asia Pacific (Singapore)
8) ap-southeast-2 : Asia Pacific (Sydney)
9) ap-northeast-1 : Asia Pacific (Tokyo)
10) ap-northeast-2 : Asia Pacific (Seoul)
11) sa-east-1 : South America (Sao Paulo)
12) cn-north-1 : China (Beijing)
13) cn-northwest-1 : China (Ningxia)
14) us-east-2 : US East (Ohio)
15) ca-central-1 : Canada (Central)
16) eu-west-2 : EU (London)
17) eu-west-3 : EU (Paris)
```

<p>Choose application</p>

```
Select an application to use
1) ebdocker
2) [ Create new Application ]
(default is 2): 1
```
<p>Choose Yes </p>

```
It appears you are using Multi-container Docker. Is this correct?
(Y/n): 
```
<b>eb deploy `<your env>` --profile `<your profile>`</b>

```
Creating application version archive "app-6227-190813_151135".
Uploading ebdocker/app-6227-190813_151135.zip to S3. This may take a while.
Upload Complete.
2019-08-13 08:11:51    INFO    Environment update is starting.      
2019-08-13 08:11:59    INFO    Deploying new version to instance(s).
2019-08-13 08:12:05    INFO    Stopping ECS task arn:aws:ecs:us-east-2:797436216668:task/827fdde6-8c32-430b-b250-858ded2f243b.
2019-08-13 08:12:09    INFO    ECS task: arn:aws:ecs:us-east-2:797436216668:task/827fdde6-8c32-430b-b250-858ded2f243b is STOPPED.
2019-08-13 08:12:10    INFO    Starting new ECS task with awseb-Ebdocker-env-1-paskw8emm7:7.
2019-08-13 08:12:14    INFO    ECS task: arn:aws:ecs:us-east-2:797436216668:task/06c38d6d-da61-42d5-8f00-29ee7f40c933 is RUNNING.
2019-08-13 08:12:21    INFO    New application version was deployed to running EC2 instances.
2019-08-13 08:12:21    INFO    Environment update completed successfully.
```

