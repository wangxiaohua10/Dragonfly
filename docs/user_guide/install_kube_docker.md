一、Install Dragonfly by Dcoker
1.install supernode 
  Please set the supernode docker instance as host net to avoid the 404 error which caused by not found the pieces.
docker run -d --net=host --name supernode --restart=always -p 8001:8001 -p 8002:8002 -v /data/dragonfly/supernode:/home/admin/supernode dragonflyoss/supernode:1.0.0 --debug  --down-limit 20  --max-bandwidth 10000 --pool-size 20 --up-limit 20
2.install dfclient
  dfclient can assign your self-registry by setting the registry parameter.
docker run -d --name dfclient --restart=always -p 65001:65001 -v /data/dragonfly/dfdaemon:/root/.small-dragonfly dragonflyoss/dfclient:1.0.0 --registry https://your-registry-domain --node "the hostips of the machine installed supernode,and seperated by comma" --ratelimit 1000M

3.How to use the dragonfly to pulling image of your registry:
  docker pull 127.0.0.1:65001/wangxiaohua10881/nginx:v1.这样就不需要修改docker的配置而重启docker

二、Install dragonfly in your kubernetes cluster by yaml
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
        image: dragonflyoss/supernode:1.0.0
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
        args: ["--registry", "https://your-registry-domain","--node","the hostips of the machine installed supernode,and seperated by comma","--ratelimit","3000M"]
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

3.Install through yaml
 kubectl create -f ds-supernode.yaml
 kubectl create -f df-daemon.yaml

4.How to use the dragonfly to pulling image of your registry: 
 docker pull 127.0.0.1:65001/wangxiaohua10881/nginx:v1
 It's not necessary to modify the docker daemon.json and no need to restart docker daemon.
