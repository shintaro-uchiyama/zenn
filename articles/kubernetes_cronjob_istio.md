---
title: "Istio入ったKubernetesのCronJobがcompleteしない"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "CronJob", "Istio"]
published: false
---

# 概要

認証認可のための鍵ローテーションをKubernetesのCronJobでやろうと試みた  
ら、なんか実行されたJobが終わらない   
podがterminateされない。作成され続ける...

という事象と格闘したお話し

# 原因と対策

Istio入ってるとJobを実行するContainerが消えても  
Sidecarのcontainerが削除されなくて  
どうもPodがterminateされないくさい

結局↓この辺でなんとか乗り切った  
https://github.com/istio/istio/issues/6324#issuecomment-760156652

# Keyのローテーション

## CronJobの内容

（テスト用なんで1分毎に動き出すようになっとります

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: capital-farm
  name: secret-edit
rules:
  - apiGroups: [ "" ]
    resources: [ "secrets" ]
    verbs: [ "get", "create", "patch" ]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: secret-edit
  namespace: capital-farm
subjects:
  - kind: ServiceAccount
    name: sa-key-rotator
    namespace: capital-farm
roleRef:
  kind: Role
  name: secret-edit
  apiGroup: ""

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-key-rotator
  namespace: capital-farm

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-standard-claim
  namespace: capital-farm
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rotate-key
  namespace: capital-farm
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 0
      template:
        spec:
          serviceAccountName: sa-key-rotator
          initContainers:
            - name: create-private-key
              image: frapsoft/openssl:latest
              imagePullPolicy: IfNotPresent
              command: [ "/bin/sh" ]
              args: [ '-c', 'openssl genrsa -out /data/id_rsa 2048 -sha256' ]
              volumeMounts:
                - name: claim-volume
                  mountPath: /data
            - name: create-public-key
              image: frapsoft/openssl:latest
              imagePullPolicy: IfNotPresent
              command: [ "/bin/sh" ]
              args: [ "-c", 'openssl rsa -in /data/id_rsa -pubout -out /data/id_rsa.pub' ]
              volumeMounts:
                - name: claim-volume
                  mountPath: /data
          containers:
            - name: rotate-key
              image: bitnami/kubectl:1.23.1
              command: [ "/bin/sh", "-c" ]
              args:
                - |
                  kubectl create secret generic ssh-key-secret --save-config --dry-run=client --from-file=ssh-privatekey=/data/id_rsa --from-file=ssh-publickey=/data/id_rsa.pub -n capital-farm -o yaml | kubectl apply -f -;
              volumeMounts:
                - name: claim-volume
                  mountPath: /data
          volumes:
            - name: claim-volume
              persistentVolumeClaim:
                claimName: pv-standard-claim
          restartPolicy: Never
```

## 終わらない

この辺の説明によると  
https://kubernetes.io/ja/docs/tasks/job/_print/

jobもpodも処理が終わったら消えてくと思ってたけど  
どうも消えない

↓Jobは増え続ける

```zsh
$ kubectl get jobs -n capital-farm -w                                                                                                                                                            +[auth]
NAME                  COMPLETIONS   DURATION   AGE
rotate-key-27354273   0/1           3m24s      3m24s
rotate-key-27354274   0/1           2m24s      2m24s
rotate-key-27354275   0/1           84s        84s
rotate-key-27354276   0/1           24s        24s
rotate-key-27354277   0/1                      0s
rotate-key-27354277   0/1           0s         0s
```

↓Podも増え続けて消えない

```zsh
$ kubectl get pods -n capital-farm -w                                                                                             +[auth]
NAME                                    READY   STATUS    RESTARTS   AGE
rotate-key-27354408-t6pnr               0/2     Pending   0          0s
rotate-key-27354408-t6pnr               0/2     Pending   0          0s
rotate-key-27354408-t6pnr               0/2     Init:0/3   0          0s
rotate-key-27354408-t6pnr               0/2     Init:1/3   0          1s
rotate-key-27354408-t6pnr               0/2     Init:2/3   0          2s
rotate-key-27354408-t6pnr               0/2     PodInitializing   0          3s
rotate-key-27354408-t6pnr               0/2     Error             0          4s
rotate-key-27354408-t6pnr               1/2     Error             0          6s
rotate-key-27354409-bnlfb               0/2     Pending           0          0s
rotate-key-27354409-bnlfb               0/2     Pending           0          0s
rotate-key-27354409-bnlfb               0/2     Init:0/3          0          0s
rotate-key-27354409-bnlfb               0/2     Init:1/3          0          2s
rotate-key-27354409-bnlfb               0/2     Init:2/3          0          3s
rotate-key-27354409-bnlfb               0/2     PodInitializing   0          4s
rotate-key-27354409-bnlfb               0/2     Error             0          5s
rotate-key-27354409-bnlfb               1/2     Error             0          6s
rotate-key-27354410-kmgqb               0/2     Pending           0          0s
rotate-key-27354410-kmgqb               0/2     Pending           0          0s
rotate-key-27354410-kmgqb               0/2     Init:0/3          0          0s
rotate-key-27354410-kmgqb               0/2     Init:1/3          0          1s
rotate-key-27354410-kmgqb               0/2     Init:2/3          0          2s
```

# トライしたこと

## 修正内容

こんな感じ

```zsh
$ git diff                                                                                                                        +[auth]
diff --git a/infra/kubernetes/capital-farm/identity/rotate-key-cron.yaml b/infra/kubernetes/capital-farm/identity/rotate-key-cron.yaml
index d995648..9527cbc 100644
--- a/infra/kubernetes/capital-farm/identity/rotate-key-cron.yaml
+++ b/infra/kubernetes/capital-farm/identity/rotate-key-cron.yaml
@@ -79,10 +79,13 @@ spec:
           containers:
             - name: rotate-key
               image: bitnami/kubectl:1.23.1
               command: [ "/bin/sh", "-c" ]
               args:
                 - |
-                  kubectl create secret generic ssh-key-secret --save-config --dry-run=client --from-file=ssh-privatekey=/data/id_rsa --from-file=ssh-publickey
+                  until curl -fsI http://localhost:15021/healthz/ready; do echo \"Waiting for Sidecar...\"; sleep 3; done;
+                  echo \"Sidecar available. Running the command...\";
+                  kubectl create secret generic ssh-key-secret --save-config --dry-run=client --from-file=ssh-privatekey=/data/id_rsa --from-file=ssh-publickey
+                  x=$(echo $?); curl -fsI -X POST http://localhost:15020/quitquitquit && exit $x
               volumeMounts:
                 - name: claim-volume
                   mountPath: /data
```


## 結果

jobがCOMPLETIONSになって終わったら消えた
```zsh
 $ kubectl get jobs -n capital-farm -w                                                                                                                                                                                  [auth]
NAME                  COMPLETIONS   DURATION   AGE
rotate-key-27354441   0/1                      0s
rotate-key-27354441   0/1           0s         0s
rotate-key-27354441   1/1           13s        13s
rotate-key-27354441   1/1           13s        13s
rotate-key-27354441   1/1           13s        13s
--
rotate-key-27354442   0/1                      0s
```

podもterminatingされて終わったら消えてくれた
```zsh
$ kubectl get pods -n capital-farm -w                                                                                             +[auth]
NAME                                    READY   STATUS    RESTARTS   AGE
rotate-key-27354417-g54r4               0/2     Pending   0          0s
rotate-key-27354417-g54r4               0/2     Pending   0          0s
rotate-key-27354417-g54r4               0/2     Init:0/3   0          0s
rotate-key-27354417-g54r4               0/2     Init:0/3   0          1s
rotate-key-27354417-g54r4               0/2     Init:1/3   0          2s
rotate-key-27354417-g54r4               0/2     Init:2/3   0          3s
rotate-key-27354417-g54r4               0/2     PodInitializing   0          4s
rotate-key-27354417-g54r4               1/2     Running           0          5s
rotate-key-27354417-g54r4               2/2     Running           0          6s
rotate-key-27354417-g54r4               1/2     NotReady          0          8s
rotate-key-27354417-g54r4               0/2     Completed         0          13s
rotate-key-27354417-g54r4               0/2     Terminating       0          14s
rotate-key-27354417-g54r4               0/2     Terminating       0          14s
```

# まとめ

絶対なんか踏むよなぁ...  

何年かしたら  
IstioさんとKubernetesさんが仲良くよしなに対応してくれることを祈っとく
