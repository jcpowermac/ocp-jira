##### TOC

- [Where to begin](docs/1_wheretobegin.md)
- [non-root and random uuid](docs/2_nonroot.md)
- [OpenShift and Jira](docs/3_openshift.md)
- [Jira's Database](docs/4_database.md)
- [OpenShift template](docs/5_template.md)

---

#### MySQL Changes
Now that we have an image for Jira and a container running we are still missing a critical piece, the database.  OpenShift provides out of the box an image and template for MySQL.  The only hangup with that image is that it uses MySQL's default collation and character set, Jira requires utf8 and utf8_bin.  So how to solve this wrinkle?

There maybe a more proper way but what I did is forked the GitHub project [sclorg/mysql-container](https://github.com/sclorg/mysql-container).  Then made the appropriate changes to the project to support setting collation and character set via environmental variables.  Below is the patch.
```
diff --git a/5.6/root/usr/bin/run-mysqld b/5.6/root/usr/bin/run-mysqld
index 9aef142..a77c858 100755
--- a/5.6/root/usr/bin/run-mysqld
+++ b/5.6/root/usr/bin/run-mysqld
@@ -14,6 +14,7 @@ log_info 'Processing MySQL configuration files ...'
 envsubst < ${CONTAINER_SCRIPTS_PATH}/my-base.cnf.template > /etc/my.cnf.d/base.cnf
 envsubst < ${CONTAINER_SCRIPTS_PATH}/my-paas.cnf.template > /etc/my.cnf.d/paas.cnf
 envsubst < ${CONTAINER_SCRIPTS_PATH}/my-tuning.cnf.template > /etc/my.cnf.d/tuning.cnf
+envsubst < ${CONTAINER_SCRIPTS_PATH}/my-collation.cnf.template > /etc/my.cnf.d/collation.cnf

 if [ ! -d "$MYSQL_DATADIR/mysql" ]; then
   initialize_database "$@"
diff --git a/5.6/root/usr/share/container-scripts/mysql/common.sh b/5.6/root/usr/share/container-scripts/mysql/common.sh
index 8548050..945448b 100644
--- a/5.6/root/usr/share/container-scripts/mysql/common.sh
+++ b/5.6/root/usr/share/container-scripts/mysql/common.sh
@@ -21,6 +21,9 @@ function export_setting_variables() {
   export MYSQL_MAX_ALLOWED_PACKET=${MYSQL_MAX_ALLOWED_PACKET:-200M}
   export MYSQL_TABLE_OPEN_CACHE=${MYSQL_TABLE_OPEN_CACHE:-400}
   export MYSQL_SORT_BUFFER_SIZE=${MYSQL_SORT_BUFFER_SIZE:-256K}
+  export MYSQL_CHAR_SET=${MYSQL_CHAR_SET:-utf8}
+  export MYSQL_COLLATION=${MYSQL_COLLATION:-utf8_bin}
+

   # Export memory limit variables and calculate limits
   local export_vars=$(cgroup-limits) && export $export_vars || exit 1
diff --git a/5.6/root/usr/share/container-scripts/mysql/my-collation.cnf.template b/5.6/root/usr/share/container-scripts/mysql/my-collation.cnf.template
new file mode 100644
index 0000000..655a3f0
--- /dev/null
+++ b/5.6/root/usr/share/container-scripts/mysql/my-collation.cnf.template
@@ -0,0 +1,3 @@
+[mysqld]
+character-set-server=${MYSQL_CHAR_SET}
+collation-server=${MYSQL_COLLATION}
```


#### OpenShift
Create a new-app from the GitHub repository and output as yaml
```
oc new-app -o yaml https://github.com/jcpowermac/mysql-container --strategy=docker --context-dir="./5.6/" > origin-mysql-container.yaml
```

In the project the dockerfile is named `Dockerfile.rhel` so we will need to modify the output.  Below is a diff between the original output and the modifications.

```
--- upstream-mysql-container.yaml	2016-11-16 09:57:25.173714802 -0500
+++ origin-mysql-container.yaml	2016-11-16 10:21:12.170152506 -0500
@@ -3,42 +3,12 @@
 - apiVersion: v1
   kind: ImageStream
   metadata:
-    annotations:
-      openshift.io/generated-by: OpenShiftNewApp
-    creationTimestamp: null
-    labels:
-      app: mysql-container
-    name: centos
-  spec:
-    tags:
-    - annotations:
-        openshift.io/imported-from: centos:centos7
-      from:
-        kind: DockerImage
-        name: centos:centos7
-      generation: null
-      importPolicy: {}
-      name: centos7
-  status:
-    dockerImageRepository: ""
-- apiVersion: v1
-  kind: ImageStream
-  metadata:
-    annotations:
-      openshift.io/generated-by: OpenShiftNewApp
-    creationTimestamp: null
     labels:
       app: mysql-container
     name: mysql-container
-  spec: {}
-  status:
-    dockerImageRepository: ""
 - apiVersion: v1
   kind: BuildConfig
   metadata:
-    annotations:
-      openshift.io/generated-by: OpenShiftNewApp
-    creationTimestamp: null
     labels:
       app: mysql-container
     name: mysql-container
@@ -47,37 +17,28 @@
       to:
         kind: ImageStreamTag
         name: mysql-container:latest
-    postCommit: {}
-    resources: {}
     source:
-      contextDir: ./5.6/
+      contextDir: "5.6"
       git:
         uri: https://github.com/jcpowermac/mysql-container
       type: Git
     strategy:
       dockerStrategy:
-        from:
-          kind: ImageStreamTag
-          name: centos:centos7
+        dockerfilePath: Dockerfile.rhel7
       type: Docker
     triggers:
     - github:
-        secret: Me6nkziBfxMSSc2KMqsn
+        secret: RWCc1PF61v0wJopdSszS
       type: GitHub
     - generic:
-        secret: jX8qkZcEn-qbOQ9zxFKw
+        secret: wJg6FvDCyydnn5idfZDp
       type: Generic
     - type: ConfigChange
     - imageChange: {}
       type: ImageChange
-  status:
-    lastVersion: 0
 - apiVersion: v1
   kind: DeploymentConfig
   metadata:
-    annotations:
-      openshift.io/generated-by: OpenShiftNewApp
-    creationTimestamp: null
     labels:
       app: mysql-container
     name: mysql-container
@@ -90,22 +51,26 @@
       resources: {}
     template:
       metadata:
-        annotations:
-          openshift.io/container.mysql-container.image.entrypoint: '["/bin/bash"]'
-          openshift.io/generated-by: OpenShiftNewApp
-        creationTimestamp: null
         labels:
           app: mysql-container
           deploymentconfig: mysql-container
       spec:
         containers:
-        - image: mysql-container:latest
+        - env:
+          - name: MYSQL_USER
+            value: testing
+          - name: MYSQL_PASSWORD
+            value: testingpass
+          - name: MYSQL_DATABASE
+            value: testingdb
+          - name: MYSQL_ROOT_PASSWORD
+            value: rootpassword
+          image: mysql-container:latest
           name: mysql-container
           ports:
           - containerPort: 3306
             protocol: TCP
           resources: {}
-    test: false
     triggers:
     - type: ConfigChange
     - imageChangeParams:
@@ -116,26 +81,20 @@
           kind: ImageStreamTag
           name: mysql-container:latest
       type: ImageChange
-  status: {}
 - apiVersion: v1
   kind: Service
   metadata:
-    annotations:
-      openshift.io/generated-by: OpenShiftNewApp
-    creationTimestamp: null
     labels:
       app: mysql-container
     name: mysql-container
   spec:
     ports:
-    - name: 3306-tcp
+    - name: mysql
       port: 3306
       protocol: TCP
       targetPort: 3306
     selector:
       app: mysql-container
       deploymentconfig: mysql-container
-  status:
-    loadBalancer: {}
 kind: List
 metadata: {}

```
