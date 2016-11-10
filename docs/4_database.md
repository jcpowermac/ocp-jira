

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