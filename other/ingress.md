Kubernetes Ingress概念及部署
---
###1概述
	对于kubernetes中Service的概念，我们知道service的表现形式为IP:Port，通过kube-proxy来实现，分别有clusterIP，NodePort,LoadBalance。ClusterIP网络仅限集群内通信，NodePort可以实现暴露服务访问入口，但service的端口会作用在每个Node节点，对于端口的管理和占用带来复杂和低效。而LoadBalance通常需要第三方云提供商支持，从而带来约束性。同时，我们知道对于基于HTTP的服务来说，不同的URL地址经常需要对应到不同的后端服务或者虚拟主机，这些机制通过kubernetes的Service是无法实现的，今天我们对于Ingress的理解和部署。
	
####1.1什么是ingress
	
  	通常，Service和pod仅有集群内互相路由的IP。到流量到达边缘节点的时候将被丢弃或者转发。在概念上，这可能看起来像：
			internet
		        |
		  ------------
		  [ Services ]
		  
	Ingress主要实现集群内所有服务的入口，通过一系列规则集合来允许外部的访问。
			 internet
		        |
		   [ Ingress ]
		   --|-----|--
		   [ Services ]
		   
	Ingress可以配置为提供服务外部访问的URL，负载均衡，SSL，提供基于名称的虚拟主机等。而ingress的具体实现是由ingress Controller实现的，在定义Ingress之前，，需要先部署Ingress Controller,以实现为所有后端Service提供一个统一的入口。Ingress Controller需要实现基于不同HTTP URL 向后端转发的负载分发规则，通常应根据应用系统的需求进行自定义实现。在kubernetes中，Ingress Controller将以Pod的形式运行，监控Apiserver的/ingress接口后端的backend service，如果service发生变化，则Ingress Controll应自动更新其转发规则。
	
	
###2.部署Ingress Controller
		下面我们将部署一个Nginx 实现的Ingress Controller，需要实现的基本逻辑如下:
			1. 监听apiserver，获取全部ingress的定义
			2. 基于ingress的定义，生成Nginx所需的配置文件(/etc/nginx/nginx.conf)
			3. 执行nginx -s reload命令,重新加载nginx.conf配置文件的内容.

		下载所需镜像:
			19931028/nginx-ingress-controller:0.8.3 （ingress controller 的镜像）
			19931028/defaultbackend:1.0 （default-http-backend的servcie的镜像）
			19931028/echoserver:1.0 （用于测试ingress的service镜像）
			
		- 首先定义个默认service backend 用于为ingress controller 响应404不存在的请求。
			> cat defualt-http-backend.yaml
				apiVersion: v1
				kind: Service
				metadata:
				  labels:
				    app: default-http-backend
				  name: default-http-backend
				spec:
				  ports:
				  - port: 80
				    targetPort: 8080
				  selector:
				    app: default-http-backend
				---
				apiVersion: v1
				kind: ReplicationController
				metadata:
				  name: default-http-backend
				spec:
				  replicas: 1
				  selector:
				    app: default-http-backend
				  template:
				    metadata:
				      labels:
				        app: default-http-backend
				    spec:
				      terminationGracePeriodSeconds: 60
				      containers:
				      - name: default-http-backend
				        # Any image is permissable as long as:
				        # 1. It serves a 404 page at /
				        # 2. It serves 200 on a /healthz endpoint
				        image: 19931028/defaultbackend:1.0
				        livenessProbe:
				          httpGet:
				            path: /healthz
				            port: 8080
				            scheme: HTTP
				          initialDelaySeconds: 30
				          timeoutSeconds: 5
				        ports:
				        - containerPort: 8080
				        resources:
				          limits:
				            cpu: 10m
				            memory: 20Mi
				          requests:
				            cpu: 10m
				            memory: 20Mi
				            
		     > kubectl create -f default-http-backend.yaml
		     ##测试相应的rc以及service是否成功创建：测试能否正常访问，通过的pod的ip/healthz或者serive的ip/访问测试。
		     >curl -s 10.254.92.159
			   default backend - 404
			   
		- 配置ingress-controller
			>  cat nginx-ingress-controller.yaml 
				apiVersion: v1
				kind: ReplicationController
				metadata:
				  name: nginx-ingress
				  labels:
				    app: nginx-ingress
				spec:
				  replicas: 1
				  selector:
				    app: nginx-ingress
				  template:
				    metadata:
				      labels:
				        app: nginx-ingress
				    spec:
				      containers:
				      - image: 19931028/nginx-ingress-controller:0.8.3
				        name: nginx-ingress-controller
				        ports:
				        - containerPort: 80
				          hostPort: 80 
				        args:
				        - /nginx-ingress-controller
				        - --default-backend-service=default/default-http-backend
				        env:
				          - name: KUBERNETES_MASTER
				            value: http://internal-kubernetes-lb-1281778143.cn-north-1.elb.amazonaws.com.cn:8080 
				          - name: POD_NAMESPACE
				            value: default
				            
			  ##创建ingress-controller rc
			  > kubectl create -f ingress-nginx-controller.yaml
			
	- 配置进行测试ingress Controller功能
		##创建一个正常的服务
		> cat test.yaml 
			apiVersion: v1
			kind: ReplicationController
			metadata:
			  name: echoheaders
			spec:
			  replicas: 1
			  template:
			    metadata:
			      labels:
			        app: echoheaders
			    spec:
			      containers:
			      - name: echoheaders
			        image: 19931028/echoserver:1.0 
			        ports:
			        - containerPort: 8080
			---
			apiVersion: v1
			kind: Service
			metadata:
			  name: echoheaders-default
			  labels:
			    app: echoheaders
			spec:
			  #type: NodePort
			  ports:
			  - port: 80
			   # nodePort: 30302
			    targetPort: 8080
			    protocol: TCP
			    name: http
			  selector:
			    app: echoheaders
		##创建
		> kubectl create -f test.yaml
		
	- 配置ingress规则,通过ingress 资源对其进行配置
		> cat test-ingress.yaml
			apiVersion: extensions/v1beta1
			kind: Ingress
			metadata:
			  name: echomap
			spec:
			  rules:
			  - host: k8s.meiqia.com 
			    http:
			      paths:
			      - path: /foo
			        backend:
			          serviceName: echoheaders-default 
			          servicePort: 80
			          
		##以上配置内容:
			1-4: 和普通的pod，service字段一样
			5-7: 定义一个虚拟主机的规则.对应于nginx server{server_name k8s.meiqia.com}
			8-10: 配置一个/foo路径,对应于nginx location /foo{};
			11-13: 添加一个service名。对应于nginx location /foo{proxy_pass http://echoheaders-default}
			
		##最后测试
		> curl -v http://nodeip:80/foo -H 'host: k8s.meiqia.com'(NoeIP为ingress Controller所运行的Node)
		##其他配置说明请查阅:https://github.com/kubernetes/ingress/tree/master/controllers/nginx
		
###Question
	1.在部署ingress Controller遇到以下小问题:
		ingress内部程序默认是通过localhost:8080连接master的apiserver.如果在非单机节点部署ingress Controller时应该通过环境变量指定
		解决:在ingress-nginx-controller.yaml 中添加
			env:
			- name: KUBERNETES_MASTER
            value: http://183.131.19.231:8080
            
   
			
					
				
