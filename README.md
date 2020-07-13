# Amazon EKS

Amazon Elastic Kubernetes Service (Amazon EKS) is a fully managed Kubernetes service. Customers such as Intel, Snap, Intuit, GoDaddy, and Autodesk trust EKS to run their most sensitive and mission critical applications because of its security, reliability, and scalability.
It is managing master node for scheduling and controlling of the pods running in worker nodes. It is behind the scenes integrating the kubernetes cluster with External Load Balancer, EC2, EFS and EBS services.
Amazon Fargate is the serverless compute for containers, we dont need to manage and provision the resources for the servers. Dynamically, it will create the worker nodes in runtime and launch the pods on the top of the worker nodes.

# Deploy Multi tier Architecture on the Amazon Elastic Kubernetes Service

In this we are deploying wordpress application as a frontend on top of the EKS cluster. MySQL database we are using with this wordpress application and nobody from the public world can't access this sql server.
Here are the steps of wordpress deployment with mySQL as backend:


**Step 1:** First we need to create a EKS cluster on the top of the AWS cloud. Creation of EKS cluster will create worker nodes as EC2 instances.
            Here is the YML code:


<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/clustercode.PNG">



After running this yml file, our cluster had been launched.


<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/EKSCluster.PNG">



<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/WorkerNodes.PNG">



**Step 2:**  Now, Create an EFS storage which comes under File system storage can be used by storage class as a provisioner. EFS is preferred over EBS because in EBS we can't mount to multiple instances running in different subnets. We are mounting EFS storage to all the availability zones in Mumbai region.



<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/EFS1.PNG">




<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/EFS2.PNG">




**Step 3:**  Create an EFS provisioner with the help of above created EFS storage. I have created a separate yml file to create provisioner and  put file system ID and DNS of the same.





            kind: Deployment
            apiVersion: apps/v1
            metadata:
              name: efs-provisioner
            spec:
              selector:
                matchLabels:
                  app: efs-provisioner
              replicas: 1
              strategy:
                type: Recreate
              template:
                metadata:
                  labels:
                    app: efs-provisioner
                spec:
                  containers:
                    - name: efs-provisioner
                      image: quay.io/external_storage/efs-provisioner:v0.1.0
                      env:
                        - name: FILE_SYSTEM_ID
                          value: fs-5a18928b
                        - name: AWS_REGION
                          value: ap-south-1
                        - name: PROVISIONER_NAME
                          value: lw-course/aws-efs
                      volumeMounts:
                        - name: pv-volume
                          mountPath: /persistentvolumes
                  volumes:
                    - name: pv-volume
                      nfs:
                        server: fs-5a18928b.efs.ap-south-1.amazonaws.com
                        path: /




Install amazon-efs-utils software in all the worker nodes so that it can be compatible with the nfs server.




**Step 4:**  Now, I have created a RBAC (Role Based Access Control) file in yml to give the power or role on that particular namespace (eksns).





            apiVersion: rbac.authorization.k8s.io/v1beta1
            kind: ClusterRoleBinding
            metadata:
              name: nfs-provisioner-role-binding
            subjects:
              - kind: ServiceAccount
                name: default
                namespace: eksns
            roleRef:
              kind: ClusterRole
              name: cluster-admin
              apiGroup: rbac.authorization.k8s.io
              



**Step 5:**  Now, Storage Class will further use this EFS provisioner to create PVC and PVC will dynamically create PV. Here is the code of Storage class and PVC.




            kind: StorageClass
            apiVersion: storage.k8s.io/v1
            metadata:
              name: aws-efs
            provisioner: lw-course/aws-efs
            ---
            kind: PersistentVolumeClaim
            apiVersion: v1
            metadata:
              name: efs-wordpress
              annotations:
                volume.beta.kubernetes.io/storage-class: "aws-efs"
            spec:
              accessModes:
                - ReadWriteMany
              resources:
                requests:
                  storage: 10Gi
            ---
            kind: PersistentVolumeClaim
            apiVersion: v1
            metadata:
              name: efs-mysql
              annotations:
                volume.beta.kubernetes.io/storage-class: "aws-efs"
            spec:
              accessModes:
                - ReadWriteMany
              resources:
                requests:
                  storage: 10Gi
                  


           
Here is the command to create Storage Class and Persistent volume claims through above created yml files.




<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/efs-provisioner-1.PNG">




<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/efs-provisioner-2.PNG">





**Step 6:**  In this step, I need to create a secret file which will store the MYSQL Root Password formatted in Base 64 encoder so that noone can see mypassword 


            apiVersion: v1
              kind: Secret
              metadata:
                name: mysql-pass
              data:
                  password: cmVkaGF0
                  



**Step 7:**  We are going to launch MySQL database application from the image version mySQL 5.7. For that, we have created a deployment file which will deploy our application on the top of the container. We have created a service of type Cluster IP so that nobody from the outside world/client can access to the database server





            apiVersion: v1
            kind: Service
            metadata:
              name: wordpress-mysql
              labels:
                app: wordpress
            spec:
              ports:
                - port: 3306
              selector:
                app: wordpress
                tier: mysql
              clusterIP: None
            ---
            apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
            kind: Deployment
            metadata:
              name: wordpress-mysql
              labels:
                app: wordpress
            spec:
              selector:
                matchLabels:
                  app: wordpress
                  tier: mysql
              strategy:
                type: Recreate
              template:
                metadata:
                  labels:
                    app: wordpress
                    tier: mysql
                spec:
                  containers:
                  - image: mysql:5.6
                    name: mysql
                    env:
                    - name: MYSQL_ROOT_PASSWORD
                      valueFrom:
                        secretKeyRef:
                          name: mysql-pass
                          key: password
                    ports:
                    - containerPort: 3306
                      name: mysql
                    volumeMounts:
                    - name: mysql-persistent-storage
                      mountPath: /var/lib/mysql
                  volumes:
                  - name: mysql-persistent-storage
                    persistentVolumeClaim:
                      claimName: efs-mysql




<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/sql-deploy.PNG">




**Step 8:**  We are going to launch wordpress application as a frontend tier from the image version wordpress. For that, we have created a deployment file in yml whihc will launch application on the top of pods/container. We are going to launch MySQL database application from the image version mySQL 5.7. For that, we have created a deployment file which will deploy our application on the top of the container. We have created a service of type Cluster IP so that nobody from the outside world/client can access to the database server




            apiVersion: v1
            kind: Service
            metadata:
              name: wordpress
              labels:
                app: wordpress
            spec:
              ports:
                - port: 80
              selector:
                app: wordpress
                tier: frontend
              type: LoadBalancer
            ---
            apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
            kind: Deployment
            metadata:
              name: wordpress
              labels:
                app: wordpress
            spec:
              selector:
                matchLabels:
                  app: wordpress
                  tier: frontend
              strategy:
                type: Recreate
              template:
                metadata:
                  labels:
                    app: wordpress
                    tier: frontend
                spec:
                  containers:
                  - image: wordpress:4.8-apache
                    name: wordpress
                    env:
                    - name: WORDPRESS_DB_HOST
                      value: wordpress-mysql
                    - name: WORDPRESS_DB_PASSWORD
                      valueFrom:
                        secretKeyRef:
                          name: mysql-pass
                          key: password
                    ports:
                    - containerPort: 80
                      name: wordpress
                    volumeMounts:
                    - name: wordpress-persistent-storage
                      mountPath: /var/www/html
                  volumes:
                  - name: wordpress-persistent-storage
                    persistentVolumeClaim:
                      claimName: efs-wordpress




<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/wordpress-deploy-1.PNG">



<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/wordpress-deploy-2.PNG">



Finally our wordpress application is ready to use with the backend as MySQL. If any load comes in the future we can scale our pods to any number of replicaslet say 3.





<img src="https://github.com/jain-vaibhav2607/Amazon-EKS-Repo/blob/master/Screenshot%20(3).png">

