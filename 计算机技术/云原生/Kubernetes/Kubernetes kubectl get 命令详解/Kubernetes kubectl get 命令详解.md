# Kubernetes kubectl get 命令详解

### kubectl get

获取列出一个或多个资源的信息。
可以使用的资源包括：

*   all

*   certificatesigningrequests (aka 'csr')

*   clusterrolebindings

*   clusterroles

*   clusters (valid only for federation apiservers)

*   componentstatuses (aka 'cs')

*   configmaps (aka 'cm')

*   controllerrevisions

*   cronjobs

*   daemonsets (aka 'ds')

*   deployments (aka 'deploy')

*   endpoints (aka 'ep')

*   events (aka 'ev')

*   horizontalpodautoscalers (aka 'hpa')

*   ingresses (aka 'ing')

*   jobs

*   limitranges (aka 'limits')

*   namespaces (aka 'ns')

*   networkpolicies (aka 'netpol')

*   nodes (aka 'no')

*   persistentvolumeclaims (aka 'pvc')

*   persistentvolumes (aka 'pv')

*   poddisruptionbudgets (aka 'pdb')

*   podpreset

*   pods (aka 'po')

*   podsecuritypolicies (aka 'psp')

*   podtemplates

*   replicasets (aka 'rs')

*   replicationcontrollers (aka 'rc')

*   resourcequotas (aka 'quota')

*   rolebindings

*   roles

*   secrets

*   serviceaccounts (aka 'sa')

*   services (aka 'svc')

*   statefulsets

*   storageclasses

*   thirdpartyresources

## 语法

```纯文本
$ get [(-o|--output=)json|yaml|wide|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...] (TYPE [NAME | -l label] | TYPE/NAME ...) [flags]

```

### 示例

列出所有运行的Pod信息。
`kubectl get pods`
列出Pod以及运行Pod节点信息。
`kubectl get pods -o wide`
列出指定NAME的 replication controller信息。
`kubectl get replicationcontroller web`
以JSON格式输出一个pod信息。
`kubectl get -o json pod web-pod-13je7`
以“pod.yaml”配置文件中指定资源对象和名称输出JSON格式的Pod信息。
`kubectl get -f pod.yaml -o json`
返回指定pod的相位值。
`kubectl get -o template pod/web-pod-13je7 --template={{.status.phase}}`
列出所有replication controllers和service信息。
`kubectl get rc,services`
按其资源和名称列出相应信息。
`kubectl get rc/web service/frontend pods/web-pod-13je7`
列出所有不同的资源对象。
`kubectl get all`
