一、通过docker安装dragonfly服务
1.安装supernode 
   注意要使用主机网络，这样是保证下载piecese的时候不会出现404.
docker run -d --net=host --name supernode --restart=always -p 8001:8001 -p 8002:8002 -v /data/dragonfly/supernode:/home/admin/supernode dragonflyoss/supernode:1.0.0 --debug  --down-limit 20  --max-bandwidth 10000 --pool-size 20 --up-limit 20
2.安装dfclient
  dfclient 在实际使用中需要指定自己的镜像仓库，
docker run -d --name dfclient --restart=always -p 65001:65001 -v /data/dragonfly/dfdaemon:/root/.small-dragonfly dragonflyoss/dfclient:1.0.0 --registry https://自己的镜像仓库 --node supernode所在节点的IP,多个IP用逗号分隔 --ratelimit 1000M

3.使用: docker pull 127.0.0.1:65001/wangxiaohua10881/nginx:v1.这样就不需要修改docker的配置而重启docker


二、通过yaml在k8s集群中部署dragonfly服务
1.ds-supernode.yaml
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    k8s-app: supernode
  name: supernode
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: supernode
  template:
    metadata:
      labels:
        k8s-app: supernode
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: "node-role.kubernetes.io/master"
                  operator: Exists
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: supernode
        #image: dragonflyoss/supernode:0.4.2
        image: dragonflyoss/supernode:1.0.0
        #image: thub.autohome.com.cn/wangxiaohua10881/supernode:v0.4.3-4
        #image: thub.autohome.com.cn/wangxiaohua10881/supernode:v1.0.0-20
        #image: hub.c.163.com/hzlilanqing/supernode:0.3.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          hostPort: 8080
          name: tomcat
          protocol: TCP
        - containerPort: 8001
          hostPort: 8001
          name: register
          protocol: TCP
        - containerPort: 8002
          hostPort: 8002
          name: download
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/localtime
          name: ltime
        - mountPath: /home/admin/supernode/
          name: data
        - mountPath: /etc/dragonfly/supernode.yml
          name: config
          readOnly: true
          subPath: supernode.yml
      dnsPolicy: ClusterFirstWithHostNet
      hostNetwork: true
      restartPolicy: Always
      volumes:
      - hostPath:
          path: /etc/localtime
          type: ""
        name: ltime
      - name: data
        hostPath:
          path: /data/dragonfly/supernode/
          type: DirectoryOrCreate
      - name: config
        configMap:
          name: supernode-config
          items:
          - key: supernode.yml
            path: supernode.yml
---
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    k8s-app: supernode
  name: supernode-config
  namespace: kube-system
data:
  supernode.yml: |-
    base:
      listenPort: 8002
      downloadPort: 8001
      debug: true
      maxBandwidth: 20000M
      peerUpLimit: 200
      peerDownLimit: 200
      eliminationLimit: 200
      failureCountLimit: 200
      linkLimit: 2000M

2.df-daemon.yaml

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    k8s-app: df-daemon
  name: df-daemon
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: df-daemon
  template:
    metadata:
      labels:
        k8s-app: df-daemon
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "external"
        operator: "Equal"
        value: "ingress"
        effect: "NoSchedule"
      containers:
      - name: df-daemon
        image: dragonflyoss/dfclient:1.0.0
        command: ["/opt/dragonfly/df-client/dfdaemon"]
        args: ["--registry", "https://thub.autohome.com.cn","--node","10.27.12.100,10.27.12.101,10.27.12.102","--ratelimit","3000M"]
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 65001
          hostPort: 65001
          name: dfget
        volumeMounts:
        - mountPath: /etc/localtime
          name: ltime
        - mountPath: /root/.small-dragonfly/
          name: data
      dnsPolicy: ClusterFirstWithHostNet
      restartPolicy: Always
      volumes:
      - hostPath:
          path: /data/dragonfly/dfdaemon
        name: data
      - hostPath:
          path: /etc/localtime
          type: ""
        name: ltime

3.安装
 kubectl create -f ds-supernode.yaml
 kubectl create -f df-daemon.yaml

4.使用: docker pull 127.0.0.1:65001/wangxiaohua10881/nginx:v1.这样就不需要修改docker的配置而重启docker
