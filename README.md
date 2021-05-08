# prometheus-operator-ocp3.11
在ocp3.11安装k8s的prometheus-operator v0.34.0
## 说明
1. ocp3.11环境是固定的，为了使用prometheus-operator的remote write功能
2. 默认部署在openshift-monitoring这个ns下
3. 使用VictoriaMetrics作为prometheus remote write方式，因此需要替换manifest/prometheus-prometheus.yaml中的<vminsert ip>
4. 使用prometheus-adapter作为hpa组件。因此在执行oc top pod/node的时候则是使用prometheus-adapter服务
5. 我不需要使用默认的prometheus-rules.yaml中定义的告警规则，因此修改了prometheus-rules.yaml中与alert相关的内容，只保留了与record相关的内容，并整理为prometheus-records.yaml。里面一些record与oc adm top指令相关。
   如果没有部署prometheus-records，可能会导致在执行oc adm top pod/nodes的时候报错没有值。
5. alertmanager并不对接告警接收端点用户，而是使用webhook，由负责webhook的服务实现终端用户通知。
6. alertmanager里面区分了容器告警和节点告警，分别对应不同的webhook服务，如果不需要区分，可以合并。
7. grafana支持匿名登录，grafana配置文件使用configmap方式挂载到grafana-deployment中，通过NodePort方式暴露30000端口。
## 部署
### 第一步：创建ns、crd、prometheus-operator、node-exporter sa等
    ````
    cd manifest/setup
    oc apply -f ./
    ````
### 第二步：禁用ocp默认的node selector(为了调度daemonset到master节点)
   ````
   oc patch namespace openshift-monitoring -p '{"metadata": {"annotations": {"openshift.io/node-selector": ""}}}'
   ```` 
### 第三步：给ds授权，以便能使用host网络模式
   ````
   oc adm policy add-scc-to-user privileged -n openshift-monitoring -z node-exporter
   ````   
## 第四步：修改prometheus-prometheus.yaml，配置remote write中的替换<vminsert-ip>
   ````
   cd manifest
   vim prometheus-prometheus.yaml
   ````
### 第五步：grafana支持匿名登录
   ````
   cd grafana
   oc create cm grafana-config --from-file=`pwd`/grafana.ini -n openshift-monitoring
   ````
### 第六步：部署prometheus-operator中的其他组件
   ````
   cd manifest
   oc apply -f ./
   ````
### 第七步：修改alertmanager配置，替换alertmanager/alertmanager.yaml中的<容器告警webhook地址>和<节点告警webhook地址>
   ````
   cd alertmanager
   vim alertmanager.yaml

   oc delete secret alertmanager-main -n openshift-monitoring

   oc create secret generic alertmanager-main --from-file=`pwd`/alertmanager.yaml -n openshift-monitoring
   ````
### 至此，所有prometheus-operator监控告警组件已经配置部署完成
