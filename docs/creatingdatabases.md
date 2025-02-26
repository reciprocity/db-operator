# Creating Databases

## Before start

First of all, in order to create a **Database** resource, a **DbInstance** resource is necessary. This defines the target server where the database must be created. **Database** resources require **DbInstance**. One or more **DbInstance(s)** are necessary for creating **Database(s)**

Check if DbInstance exists on cluster and running.
```
$ kubectl get dbin
```

The result should be like below.
```
NAME              PHASE      STATUS
example-generic   Running   false
```

If you get `No resources found.`, go to [how to create DbInstance](creatinginstances.md)

For more details about how it works check [here](howitworks.md)

## Next
* [Creating databases](#CreatingDatabases)
* [Connecting to the database from pods](#ConnectingToTheDatabase)
* [Checking database status](#CheckingDatabaseStatus)
* [Enabling postgreSQL extensions](#PostgreSQL)

### CreatingDatabases

Create Database custom resource

```YAML
apiVersion: "kci.rocks/v1alpha1"
kind: "Database"
metadata:
  name: "example-db"
spec:
  secretName: example-db-credentials # DB Operator will create secret with this name. it contains db name, user, password
  instance: example-gsql # This has to be match with DbInstance name
  deletionProtected: false # Protection to not delete database when custom resource is deleted
  backup:
    enable: false # turn it to true when you want to use back up feature. currently only support postgres
    cron: "0 0 * * *"
```

After successful `Database` creation, you must be able to get a secret named like `example-db-credentials`.

```
$ kubectl get secret example-db-credentials
```
It contains credentials to connect to the database generated by operator.

For postgres,
```YAML
apiVersion: v1
kind: Secret
metadata:
  labels:
    created-by: db-operator
  name: example-db-credentials
type: Opaque
data:
  POSTGRES_DB: << base64 encoded database name (generated by db operator) >>
  POSTGRES_PASSWORD: << base64 encoded password (generated by db operator) >>
  POSTGRES_USER: << base64 encoded user name (generated by db operator) >>
```

For mysql,
```YAML
apiVersion: v1
kind: Secret
metadata:
  labels:
    created-by: db-operator
  name: example-db-credentials
type: Opaque
data:
  DB: << base64 encoded database name (generated by db operator) >>
  PASSWORD: << base64 encoded password (generated by db operator) >>
  USER: << base64 encoded user name (generated by db operator) >>
```

You should be able to get configmap with same name as secret like `example-db-credentials`.
```
$ kubectl get configmap example-db-credentials
```

It contains connection information for database server access.

```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    created-by: db-operator
  name: example-db-credentials
data:
  DB_CONN: << database server address >>
  DB_PORT: << database server port >>
  DB_PUBLIC_IP: << database server public ip >>
  ...
```

### ConnectingToTheDatabase

By using the secret and the configmap created by operator after database creation, pods in Kubernetes can connect to the database.
The following deployment is an example of how application pods can connect to the database.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
      - name: example-app
        image: "appimage:latest"
        imagePullPolicy: Always
        env:
        - name: POSTGRES_PASSWORD_FILE
          value: /run/secrets/postgres/POSTGRES_PASSWORD
        - name: POSTGRES_USERNAME
          valueFrom:
            secretKeyRef:
              key: POSTGRES_USER
              name: example-db-credentials # has to be same with spec.secretName of Database custom resource
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              key: POSTGRES_DB
              name: example-db-credentials # has to be same with spec.secretName of Database custom resource
        - name: POSTGRES_HOST
          valueFrom:
            configMapKeyRef:
              key: DB_CONN
              name: example-db-credentials # has to be same with spec.secretName of Database custom resource
        volumeMounts:
        - mountPath: /run/secrets/postgres/
          name: db-secret
          readOnly: true
      volumes:
      - name: db-secret
        secret:
          defaultMode: 420
          secretName: example-db-credentials # has to be same with spec.secretName of Database custom resource
```


### CheckingDatabaseStatus

To check **Database** status
```
kubectl get db example-db
```

The output should be like
```
NAME          PHASE   STATUS   PROTECTED   DBINSTANCE         AGE
example-db    Ready   true     false       example-generic    4h39m
```

Possible phases and meanings
| Phase                 | Description                           |
|-------------------    |-----------------------                |
| `Creating`            | On going creation of database in the database server |
| `InfoConfigMapCreating` | Generating and building configmap data with database server information |
| `InstanceAccessSecretCreating`  | When instance type is `google`, it's creating access secret in the namespace where `Database` exists.  |
| `BackupJobCreating`   | Creating backup `Cronjob` when backup is enabled in the `spec` |
| `Finishing`           | Setting status of `Database` to true |
| `Ready`               | `Database` is created and all the configs are applied. Healthy status. |
| `Deleting`            | `Database` is being deleted. |

### PostgreSQL

PostgreSQL extensions listed under `spec.extensions` will be enabled by DB Operator.
DB Operator execute `CREATE EXTENSION IF NOT EXISTS` on the target database.

```YAML
apiVersion: "kci.rocks/v1alpha1"
kind: "Database"
metadata:
  name: "example-db"
spec:
  secretName: example-db-credentials
  instance: example-gsql
  deletionProtected: false
  extensions:
    - pgcrypto
    - uuid-ossp
    - plpgsql
  database: customdbname # <NAMESPACE>_<DB_INSTANCE_NAME> by default
  user: customuser # <NAMESPACE>_<DB_INSTANCE_NAME> by default
```
When monitoring is enabled on DbInstance spec, `pg_stat_statements` extension will be enabled.
If below error occurs during database creation, the module must be loaded by adding pg_stat_statements to shared_preload_libraries in postgresql.conf on the server side.
```
ERROR: pg_stat_statements must be loaded via shared_preload_libraries
```
