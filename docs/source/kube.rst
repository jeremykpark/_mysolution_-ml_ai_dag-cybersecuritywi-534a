Scaling [--solutionname--] With Kubernetes
===========================

Generated On: --datetime-- UTC

You can scale your solution with Kubernetes.  To do so, will will need to apply the following YAML files to your Kubernetes cluster.

.. tip::
   Refer to TML documentation for more information on `scaling with Kubernetes <https://tml.readthedocs.io/en/latest/kube.html>`_.

   Watch the YouTube Video: `here <https://www.youtube.com/watch?v=MEbmTXIQpVo>`_.

   You can also run the YAML files locally in a 1 node kubernetes cluster called minikube.  Refer to `Installing minikube <https://tml.readthedocs.io/en/latest/kube.html#installing-minikube>`_

.. important:: 
   Below assumes you have a Kubernetes cluster and **kubectl** installed in your Linux environment.

.. attention::

   Make sure to STOP the TSS Container and other containers before running Kubernetes/Minikube.

   If you get the following WARNING from Kubernetes:

    Warning  FailedScheduling  default-scheduler  0/1 nodes are available: 1 Insufficient nvidia.com/gpu. preemption: 0/1 nodes are available: 1 No preemption victims found for 
    incoming pod.  **Make sure no other application is using the GPU.**  You can check by executing in terminal the command: **nvidia-smi**

    Oherwise, Issue the commands below:

.. code-block::

   sudo apt update && sudo apt install -y nvidia-docker2

   sudo nvidia-ctk runtime configure --runtime=docker 

   sudo systemctl restart docker


Based on your TML solution [--solutionname--] - if you want to scale your application with Kubernetes - you will need to apply the following YAML files.

.. list-table::

   * - **YML File**
     - **Description**
   * - :ref:`--solutionnamefile--`
     - This is your main solution YAML file.  
 
       It MUST be applied to your Kubernetes cluster.
   * - :ref:`secrets.yml`
     - You MUST store your passwords in base64 format 

       in this file.  For instructions on how to convert

       plain text passwords to base64 refer to `instructions <https://tml.readthedocs.io/en/latest/kube.html#how-to-store-secure-passwords-in-kubernetes>`_
   * - :ref:`mysql-storage.yml`
     - This is storage allocation for MySQL DB.
 
       It MUST be applied to your Kubernetes cluster.
   * - :ref:`mysql-db-deployment.yml`
     - This is the MySQL deployment YAML.
 
       It MUST be applied to your Kubernetes cluster.
   * - :ref:`kafka.yml`
     - This is the Kafka deployment YAML.
 
       This is MANDATORY if using kafka locally or on-premise.
   * - :ref:`privategpt.yml`
     - This is the privateGPT deployment YAML.
 
       This is OPTIONAL.  However, it must be 
 
       applied if using Step 9 DAG.
   * - :ref:`qdrant.yml`
     - This is the Qdrant deployment YAML.
 
       This is OPTIONAL.  However, it must be 
 
       applied if using Step 9 DAG.
   * - :ref:`nginx-ingress--nginxname--.yml`
     - If you are scaling your TML solution you must

       apply the nginx-ingress--nginxname--.yml; this yaml is 

       auto-generated for every TML solution.

       For more details see section 

       :ref:`Scaling with NGINX Ingress and Ingress Controller`

kubectl Apply command
-----------------

.. important::
   To apply the YAML files below to your Kubernetes cluster simply run this command:

.. code-block:: YAML

   --kubectl--

--solutionnamefile--
------------------------

.. important::
   Copy and Paste this YAML file: --solutionnamefile-- - and save it locally.

.. attention::

   MAKE SURE to update any tokens and passwords in the **secrets.yml** file:

          1. GITPASSWORD (MANDATORY)
             
          2. READTHEDOCS (MANDATORY)
             
          3. KAFKACLOUDPASSWORD (OPTIONAL)
             
          4. MQTTPASSWORD (OPTIONAL)

   For instructions on how to do this, refer to `instructions <https://tml.readthedocs.io/en/latest/kube.html#how-to-store-secure-passwords-in-kubernetes>`_

.. code-block:: YAML

   ################# --solutionnamefile--
   --solutionnamecode--

.. tip::

   In the solution YAML file above, you can adjust the **replicas** field.  Currently, **replicas: 3** for demonstration purposes. 

secrets.yml
-----------------

.. important::
   You MUST store base64 passwords in this file and apply it to the Kubernetes cluster.  

   Refer to `instructions <https://tml.readthedocs.io/en/latest/kube.html#how-to-store-secure-passwords-in-kubernetes>`_.

.. code-block:: YAML
      
      ###################secrets.yml
      apiVersion: v1
      kind: Secret
      metadata:
        name: tmlsecrets
      type: Opaque
      data:
        readthedocs: <enter your base64 password>
        githubtoken: <enter your base64 password>
        mqttpass: <enter your base64 password>
        kafkacloudpassword: <enter your base64 password>

mysql-storage.yml
------------------------

.. important::
   Copy and Paste this YAML file: mysql-storage.yml - and save it locally.

.. code-block:: YAML

      ################# mysql-storage.yml
      apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: mysql-pv-volume
        labels:
          type: local
      spec:
        storageClassName: manual
        capacity:
          storage: 20Gi
        accessModes:
          - ReadWriteMany
        hostPath:
          path: "/mnt/data"
      ---
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: mysql-pv-claim
      spec:
        storageClassName: manual
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 20Gi

mysql-db-deployment.yml
------------------------

.. important::
   Copy and Paste this YAML file: mysql-db-deployment.yml - and save it locally.

.. code-block:: YAML

      ################# mysql-db-deployment.yml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: mysql
      spec:
        selector:
          matchLabels:
            app: mysql
        strategy:
          type: Recreate
        template:
          metadata:
            labels:
              app: mysql
          spec:
            containers:
            - image: maadsdocker/mysql:latest
              name: mysql
              resources:
               limits:
                memory: "512Mi"
                cpu: "1500m"
              env:
              - name: MYSQL_ROOT_PASSWORD
                value: "raspberry"
              - name: MYSQLDB
                value: "tmlids"
              - name: MYSQLDRIVERNAME
                value: "mysql"
              - name: MYSQLHOSTNAME
                value: "mysql:3306"
              - name: MYSQLMAXCONN
                value: "4"
              - name: MYSQLMAXIDLE
                value: "10"
              - name: MYSQLPASS
                value: "raspberry"
              - name: MYSQLUSER
                value: "root"
              ports:
              - containerPort: 3306
                name: mysql
              volumeMounts:
              - name: mysql-persistent-storage
                mountPath: /var/lib/mysql
            volumes:
            - name: mysql-persistent-storage
              persistentVolumeClaim:
                claimName: mysql-pv-claim
      
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: mysql-service
      spec:
        ports:
        - port: 3306
        selector:
          app: mysql

kafka.yml
------------

This is the Kafka service needed by TML pods - if using Kafka locally or on-premise.

.. code-block:: YAML
            
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: kafka
      spec:
        selector:
          matchLabels:
            app: kafka
        replicas: 1 # tells deployment to run 1 pods matching the template
        template:
          metadata:
            labels:
              app: kafka
          spec:
            containers:
            - name: kafka
              image: maadsdocker/kafka-amd64  # IF you DO NOT have NVIDIA GPU use: maadsdocker/tml-privategpt-no-gpu-amd64
              env:
              - name: KAFKA_HEAP_OPTS
                value: "-Xmx512M -Xms512M"
              - name: PORT
                value: "9092"
              - name: TSS
                value: "0"
              - name: KUBE
                value: "1"
              - name: KUBEBROKERHOST
                value: "kafka-service:9092"      
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: kafka-service
      spec:
        ports:
        - port: 9092
        selector:
          app: kafka

privategpt.yml
---------------

.. note::
    This YAML is Optional - Use Only If Step 9 Dag is used

.. important::
   Copy and Paste this YAML file: privategpt.yml - and save it locally.

.. note::
   By default this assumes you have a Nvidia GPU in your machine and so it using the Nvidia privateGPT container:

    **image: maadsdocker/tml-privategpt-with-gpu-nvidia-amd64**

   if you DO NOT have a Nvidia GPU installed then change image to:

    **image: maadsdocker/tml-privategpt-no-gpu-amd64**

.. code-block:: YAML
            
      ################# privategpt.yml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: privategpt
      spec:
        selector:
          matchLabels:
            app: privategpt
        replicas: 1 # tells deployment to run 1 pods matching the template
        template:
          metadata:
            labels:
              app: privategpt
          spec:
            containers:
            - name: privategpt
              image: maadsdocker/tml-privategpt-with-gpu-nvidia-amd64 # IF you DO NOT have NVIDIA GPU use: maadsdocker/tml-privategpt-no-gpu-amd64
              env:
              - name: NVIDIA_VISIBLE_DEVICES
                value: all
              - name: DP_DISABLE_HEALTHCHECKS
                value: xids
              - name: WEB_CONCURRENCY
                value: "3"
              - name: GPU
                value: "1"
              - name: COLLECTION
                value: "tml"
              - name: PORT
                value: "8001"
              - name: CUDA_VISIBLE_DEVICES
                value: "0"
              - name: TSS
                value: "0"
              - name: KUBE
                value: "1"
              resources:             # REMOVE or COMMENT OUT: IF you DO NOT have NVIDIA GPU
                limits:              # REMOVE or COMMENT OUT: IF you DO NOT have NVIDIA GPU
                  nvidia.com/gpu: 1  # REMOVE or COMMENT OUT: IF you DO NOT have NVIDIA GPU
              ports:
              - containerPort: 8001
            tolerations:             # REMOVE or COMMENT OUT: IF you DO NOT have NVIDIA GPU
            - key: nvidia.com/gpu    # REMOVE or COMMENT OUT: IF you DO NOT have NVIDIA GPU
              operator: Exists       # REMOVE or COMMENT OUT: IF you DO NOT have NVIDIA GPU
              effect: NoSchedule     # REMOVE or COMMENT OUT: IF you DO NOT have NVIDIA GPU     
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: privategpt-service
        labels:
          app: privategpt-service
      spec:
        type: NodePort #Exposes the service as a node ports
        ports:
        - port: 8001
          name: p1
          protocol: TCP
          targetPort: 8001
        selector:
          app: privategpt                    
          
qdrant.yml
---------------

.. note::
    This YAML is Optional - Use Only If Step 9 Dag is used

.. important::
   Copy and Paste this YAML file: qdrant.yml - and save it locally.

.. code-block:: YAML

      ################# qdrant.yml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: qdrant
      spec:
        selector:
          matchLabels:
            app: qdrant
        replicas: 1 
        template:
          metadata:
            labels:
              app: qdrant
          spec:
            #hostNetwork: true
            containers:
            - name: qdrant
              image: qdrant/qdrant 
              ports:   
              - containerPort: 6333
              volumeMounts:
              - mountPath: /qdrant/storage
                name: qdata
            volumes:
            - name: qdata
              hostPath:
                path: /qdrant_storage          
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: qdrant-service
        labels:
          app: qdrant-service
      spec:
        type: NodePort #Exposes the service as a node ports
        ports:
        - port: 6333
          name: p1
          protocol: TCP
          targetPort: 6333
        selector:
          app: qdrant
          
.. tip::
   The number of replicas can be changed in the **cybersecuritywithprivategpt-3f10.yml** file: look for **replicas**.  You can increase or decrease the number of replicas based on the amout of real-time data you are processing.

Kubernetes Dashboard Visualization
----------------------------------

To visualize the dashboard you need to forward ports to your solution **deployment in Kubernetes**.  For this solution, the port forward command would be:

.. code-block::

   --kube-portforward--

After you forward the ports then copy/paste the viusalization URL below and run your dashboard.

.. code-block::

   --visualizationurl--

Scaling with NGINX Ingress and Ingress Controller
-------------------------------------

All TML solutions will scale with NGINX ingress to perform load-balancing.  But, before you can use ingress - ingress MUST be enabled in Kubernetes cluster.  Follow these steps:

.. important::
   **STEP 1:  To turn on ingress in minikube type:**

   .. code-block::

      minikube addons enable ingress

   .. code-block::

      minikube addons enable ingress-dns

   **STEP 2:  In Linux Add tml.tss domain name to /etc/hosts file**

    a. Edit your **/etc/hosts** file 

    b. add an entry to **/etc/hosts**: 

      .. code-block::
     
         127.0.0.1 tml.tss

    c. Save the file

   **STEP 2b:  In Windows Add tss.tml domain name to C:\\Windows\\System32\\drivers\\etc**

    a. Edit your **C:\\Windows\\System32\\drivers\\etc\\hosts** file  (Note: You may need to COPY the hosts file to another directory, then edit the file, then copy it back to 
       **C:\\Windows\\System32\\drivers\\etc\\hosts**

    b. add an entry: 

       .. code-block:: 

          127.0.0.1 tml.tss

    c. Save the file 

    d. copy it back to **C:\\Windows\\System32\\drivers\\etc\\hosts**

   **STEP 3:  In a new Linux terminal you MUST turn on minikube tunnel type:**

   .. code-block::

      minikube tunnel

   **STEP 4:  Apply nginx-ingress--nginxname--.yml to your kubernetes cluster.  First you need to save it locally then apply it:**

nginx-ingress--nginxname--.yml
-------------

   .. code-block::

      #########nginx-ingress--nginxname--.yml
      --ingress--

   .. code-block::

      kubectl apply -f nginx-ingress--nginxname--.yml

You are now ready to run the Dashboard using Ingress load balancing.

Ingress Dashboard Visualization
-------------------

Copy and paste this URL below in your browser and start streaming.  Because you are now using INGRESS, Kubernetes will perform load balancing on the streaming data.

.. code-block::

   --visualizationurling--


Kubernetes Pod Access Commands
---------------------

**To go inside the pods, you can type command:** 

.. code-block::

   kubectl exec -it <pod name> -- bash 

Note: replace **<pod name>** with actual pod name..use this command to get the pod name

.. code-block::

   kubectl get pods -A

**To list service pods type:**

.. code-block::

   kubectl get svc -A

**To list deployment pods type:**

.. code-block::

   kubectl get deployments -A

**To Horizontally AUTO-SCALE Deployments type:**

  .. code-block::

     kubectl autoscale deployment  <deployment name> --cpu-percent=50 --min=1 --max=100

.. important::

   The above command instructs Kubernetes to scale pods based on 50% CPU utilization to a minimum number of pods of 1 (small workload) to a maximum of 100 pods for large world loads.  Of 
   course, you can easily change these min and max numbers.
   
   This auto-scaling is very important to scale up and down your solution, while efficiently managing cloud computing costs.

**To list deployments being auto-scaled type:**

  .. code-block::

     kubectl get hpa -A

**To delete the pods:**

.. code-block::

   kubectl delete all --all --all-namespaces

**To get information on a pod type:**

.. code-block:: 

   kubectl describe pod <pod name>

**Start minikube with NVIDIA GPU Access:**

.. code-block::

     minikube start --driver docker --container-runtime docker --gpus all --cni calico --memory 8192

.. note::

   Note you may need to type: **./minikube**

**Start minikube with NO GPU:**

.. code-block::

   minikube start --driver docker --container-runtime docker --cni calico --memory 8192

**DELETE minikube:**

.. code-block::

   minikube delete

.. tip::

   Adjust the **\-\-memory 8192** as needed.
