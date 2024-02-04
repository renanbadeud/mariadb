# mariadb-galera cluster deployment

First add helm repo:

    helm repo add bitnami https://charts.bitnami.com/bitnami

Create a namespace in kubernetes:

    alias k=kubectl
    k create ns mariadb

A file [values.yaml](values.yaml) was used to customize some settings:

- Add extraEnvVars to the cluster work in a dual stack ip k8s cluster as fixed in this [issue](https://github.com/bitnami/charts/issues/17733#issuecomment-1885127124)
- Set rootUser.password to define password for the admin user
- The following paramenters was increased to wait more time for the second and third engines be added to mariadb-galera cluster
    - livenessProbe.initialDelaySeconds Delay before liveness probe is initiated
    - livenessProbe.periodSeconds How often to perform the probe
    - livenessProbe.timeoutSeconds When the probe times out
    - readinessProbe.initialDelaySeconds Delay before readiness probe is initiated
    - readinessProbe.periodSeconds How often to perform the probe
    - readinessProbe.timeoutSeconds When the probe times out

- Set wsrep_sst_method=rsync, the default one mariabackup was not working

      mariadbConfiguration:
        ...
        [galera]
        wsrep_sst_method=rsync 

Next, deploy mariadb-galera-cluster in mariadb namespace:

    helm install mariadb-galera-cluster bitnami/mariadb-galera -f values.yaml -n mariadb

Wait some time, then check that 3 pods was created with the pattern mariadb-galera-cluster-*, and take a look in the pods IPs:
    
    k get pods -n mariadb

Also check that services was created:
    
    k -n mariadb get svc

Check that the mariadb-galera-internal service endpoints points to galera pods IPs

    k -n mariadb get endpoints mariadb-galera-cluster
    
## Tests

Enter in galera mariadb engine 0 pod, in the default one database my_database:

    k -n mariadb exec -it pod/mariadb-galera-cluster-0 -- mysql -uroot -p$(kubectl get secret --namespace mariadb mariadb-galera-cluster -o jsonpath="{.data.mariadb-root-password}" | base64 -d) my_database

Inside mariadb create a table:

    CREATE TABLE Artist (Name VARCHAR(50), Song VARCHAR(50));

Insert data:

    INSERT INTO Artist (Name, Song) VALUES ("Bobby Bare", "Five Hundred Miles");
  
Check data:

    SELECT * from Artist;

Exit the pod using exit command and enter in galera mariadb engine 1 pod:

    k -n mariadb exec -it pod/mariadb-galera-cluster-1 -- mysql -uroot -p$(kubectl get secret --namespace mariadb mariadb-galera-cluster -o jsonpath="{.data.mariadb-root-password}" | base64 -d) my_database

Insert data:

    INSERT INTO Artist (Name, Song) VALUES ("John Denver", "Annie's Song");
    INSERT INTO Artist (Name, Song) VALUES ("Avicii", "The Nights");

Check that data that was inserted in engine 0 is also present with the newly inserted data (sync is working):
    
    SELECT * from Artist;

Exit the pod, and enter in galera mariadb engine 2 pod:

    k -n mariadb exec -it pod/mariadb-galera-cluster-2 -- mysql -uroot -p$(kubectl get secret --namespace mariadb mariadb-galera-cluster -o jsonpath="{.data.mariadb-root-password}" | base64 -d) my_database

Delete data:

    DELETE from Artist where Name = 'John Denver';

Check that data was removed:

    SELECT * from Artist;


You also can run a temporary client pod to make operations in the database, note that the mysql host is the service mariadb-galera-cluster, the requests made by client are forwarded to any of the galera engine pods:

     kubectl run mariadb-galera-cluster-client --rm --tty -i --restart='Never' --namespace mariadb --image docker.io/bitnami/mariadb-galera:11.2.2-debian-11-r5 --command -- mysql -h mariadb-galera-cluster -P 3306 -uroot -p$(kubectl get secret --namespace mariadb mariadb-galera-cluster -o jsonpath="{.data.mariadb-root-password}" | base64 -d) my_database

Check the data:

    SELECT * from Artist;



    


    
