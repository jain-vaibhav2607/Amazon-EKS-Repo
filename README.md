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
