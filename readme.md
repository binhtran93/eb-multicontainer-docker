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

# Tạo private image trên ECR

# Overview structure    

# Config laravel 

# Config nginx

# Config .ebextensions 



  
