# **Steps to provison the kafka cluster using Github Actions Workflow**

To Provision the **coto-kafka-prod-cluster**

**Step-1 → Configure kafka cluster (Strimzi-operator,zookeeper,brokers and entity operator)**

**.github/workflows/Kafka-cluster-setup-prod.yaml**

```bash
name: Kafka Cluster Deployment (Prod)
on:
  workflow_dispatch

env:
  KAFKA_CLUSTERS_BOOTSTRAPSERVERS: "${{ secrets.KAFKA_CLUSTERS_BOOTSTRAPSERVERS }}"
  KAFKA_CLUSTERS_NAME: "${{ secrets.KAFKA_CLUSTERS_NAME }}"
  KUBE_CONFIG_DATA: "{{ secrets.KUBE_CONFIG_DATA }}"
  LINODE_ACCESS_KEY: "${{ secrets.LINODE_ACCESS_KEY }}"
  LINODE_CLI_TOKEN: "${{ secrets.LINODE_CLI_TOKEN }}"
  LINODE_SECRET_KEY: "${{ secrets.LINODE_SECRET_KEY }}"
  SPRING_SECURITY_USER_NAME: "${{ secrets.SPRING_SECURITY_USER_NAME }}"
  SPRING_SECURITY_USER_PASSWORD: "${{ secrets.SPRING_SECURITY_USER_PASSWORD }}"
  POOL_ID: "${{ vars.POOL_ID }}"

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: Prod-kafka
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
         github-token: ${{ secrets.POOL_ID }}

      - name: Configure kubectl
        uses: Azure/k8s-set-context@v1
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_DATA }}

                
      - name: Check if Kafka namespace exists
        run: |
          if kubectl get namespace kafka > /dev/null 2>&1; then
            echo "Kafka namespace already exists"
          else
            kubectl create namespace kafka
          fi     
      
      - name: Add Helm Repo
        id: add-helm-repo
        run: |
          # Attempt to add Helm repository
          HELM_REPO_OUTPUT=$(helm repo add strimzi https://strimzi.io/charts/ 2>&1)
          if echo "$HELM_REPO_OUTPUT" | grep -q "strimzi"; then
            echo "Helm repository already exists"
          else
            echo "Helm repository added successfully"
            echo "Helm repository output: $HELM_REPO_OUTPUT"
          fi
          
      - name: Check if Kafka Operator is installed and pods are running
        run: |
          if helm list -n kafka | grep 'strimzi-operator'; then
            echo "Strimzi Kafka Operator is already installed."  
            envsubst < kafka/prod/values.yaml > kafka/prod/values-substituted.yaml
            helm upgrade strimzi-operator strimzi/strimzi-kafka-operator -f kafka/prod/values-substituted.yaml -n kafka
          else
            # Substitute environment variables in the values.yaml file
            envsubst < kafka/prod/values.yaml > kafka/prod/values-substituted.yaml
            helm install strimzi-operator strimzi/strimzi-kafka-operator -f kafka/prod/values-substituted.yaml -n kafka
            retries=10  # Number of retries
            wait_time=10  # Initial wait time between retries in seconds
            sleep 10
            for ((i=0; i<$retries; i++)); do
              if [[ $(kubectl get pods -n kafka | grep 'strimzi-cluster-operator' | awk '{print $3}' | grep -v 'Running' | wc -l) -ne 0 ]]; then
                echo "Not all Strimzi Kafka Operator pods are running. Waiting for pods to be ready..."
                sleep $wait_time
              else
                kubectl get pods -n kafka -l app=strimzi-cluster-operator
                echo "Strimzi Kafka Operator pods are running."
                break
              fi
              if [ $i -eq $((retries - 1)) ]; then
                echo "Strimzi Kafka Operator pods are not running after waiting. Showing pod statuses:"
                kubectl get pods -n kafka | grep 'strimzi-cluster-operator'
                exit 1
              fi
            done
          fi
      
      - name: Check Kafka Deployment YAML and Wait for Pods
        run: |
         if kubectl get kafka -n kafka -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q True; then
           echo "Kafka cluster is ready."
         else
           # Fetching POOL_ID using envsubst
           envsubst < kafka/prod/kafka-cluster-prod.yaml > kafka/prod/kafka-cluster-prod-substituted.yaml
           sed -i "s/\$POOL_ID/${{ vars.POOL_ID }}/g" kafka/prod/kafka-cluster-prod-substituted.yaml
           kubectl apply -f kafka/prod/kafka-cluster-prod-substituted.yaml -n kafka
           
           retries=50  # Number of retries
           wait_time=20  # Initial wait time between retries in seconds
           sleep 10
           for ((i=0; i<$retries; i++)); do
             if [[ $(kubectl get pods -n kafka -l app=kafka --no-headers=true | grep 'coto-kafka-prod-cluster-zookeeper' | awk '{print $3}' | grep -v 'Running' | wc -l) -ne 0 ]]; then               
               kubectl get pods -n kafka -l app=kafka
               echo "Not all Zookeeper Deployment pods are running. Waiting for pods to be ready..."
               sleep $wait_time
               wait_time=$((wait_time * 2))  # Exponential backoff for wait time
             elif [[ $(kubectl get pods -n kafka -l app=kafka --no-headers=true | grep 'coto-kafka-prod-cluster-zookeeper' | awk '{print $2}' | grep -v '1/1' | wc -l) -ne 0 ]]; then    
               kubectl get pods -n kafka -l app=kafka
               echo "Not all Zookeeper Deployment pods are running. Waiting for pods to be ready..."
               sleep $wait_time
               wait_time=$((wait_time * 2))  # Exponential backoff for wait time
             elif [[ $(kubectl get pods -n kafka -l app=kafka --no-headers=true | grep 'coto-kafka-prod-cluster-kafka' | awk '{print $3}' | grep -v 'Running' | wc -l) -ne 0 ]]; then               
               kubectl get pods -n kafka -l app=kafka
               echo "Not all Kafka Deployment pods are running. Waiting for pods to be ready..."
               sleep $wait_time
               wait_time=$((wait_time * 2))  # Exponential backoff for wait time
             elif [[ $(kubectl get pods -n kafka -l app=kafka --no-headers=true | grep 'coto-kafka-prod-cluster-kafka' | awk '{print $2}' | grep -v '1/1' | wc -l) -ne 0 ]]; then    
               kubectl get pods -n kafka -l app=kafka
               echo "Not all Kafka Deployment pods are running. Waiting for pods to be ready..."
               sleep $wait_time
               wait_time=$((wait_time * 2))  # Exponential backoff for wait time
             elif [[ $(kubectl get pods -n kafka -l app=kafka --no-headers=true | grep 'coto-kafka-prod-cluster-entity-operator' | awk '{print $3}' | grep -v 'Running' | wc -l) -ne 0 ]]; then               
               kubectl get pods -n kafka -l app=kafka
               echo "Not all Kafka Deployment pods are running. Waiting for pods to be ready..."
               sleep $wait_time
               wait_time=$((wait_time * 2))  # Exponential backoff for wait time
             elif [[ $(kubectl get pods -n kafka -l app=kafka --no-headers=true | grep 'coto-kafka-prod-cluster-entity-operator' | awk '{print $2}' | grep -v '2/2' | wc -l) -ne 0 ]]; then    
               kubectl get pods -n kafka -l app=kafka
               echo "Not all entity operator Deployment pods are running. Waiting for pods to be ready..."
               sleep $wait_time
               wait_time=$((wait_time * 2))  # Exponential backoff for wait time
             else
               kubectl get pods -n kafka -l app=kafka
               echo "All Kafka Deployment pods are running."
               break
             fi
             
             if [ $i -eq $((retries - 1)) ]; then
               echo "Kafka Deployment pods are not running after waiting. Showing pod statuses:"
               kubectl get pods -n kafka -l app=kafka
               exit 1
             fi
           done
         fi

      - name: Apply Kafka UI YAML
        run: |
          envsubst < kafka/prod/kafka-ui-prod.yaml > kafka/prod/kafka-ui-prod-substituted.yaml
          sed -i "s/\$POOL_ID/${{ vars.POOL_ID }}/g" kafka/prod/kafka-ui-prod-substituted.yaml
          kubectl apply -f kafka/prod/kafka-ui-prod-substituted.yaml -n kafka

     
      - name: Verify all pods in namespace kafka
        run: |
          kubectl get pods -n kafka
```

**Go to Github Actions and run the workflow**

![Screenshot from 2024-04-25 21-40-07](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/8c29f635-3371-4a29-998e-3f68953f20ca)


![Screenshot from 2024-04-25 22-04-09](https://github.com/shristiopstree/Kafka/assets/112183257/7f2de741-dff4-4fc3-a3e9-8da88ac10936)


**Output** 

```bash
kubectl get pods -n kafka

NAME                                                       READY   STATUS    RESTARTS   AGE
coto-kafka-prod-cluster-entity-operator-675bbb5b7f-mkgts   2/2     Running   0          3h28m
coto-kafka-prod-cluster-kafka-0                            1/1     Running   0          3h36m
coto-kafka-prod-cluster-kafka-1                            1/1     Running   0          3h33m
coto-kafka-prod-cluster-kafka-2                            1/1     Running   0          3h31m
coto-kafka-prod-cluster-zookeeper-0                        1/1     Running   0          3h39m
coto-kafka-prod-cluster-zookeeper-1                        1/1     Running   0          3h38m
coto-kafka-prod-cluster-zookeeper-2                        1/1     Running   0          3h37m
kafka-ui-deployment-7d88649df8-vdmp7                       1/1     Running   0          3h41m
strimzi-cluster-operator-786b76ffbc-7dm6q                  1/1     Running   0          3h30m

```

**Step-2**

**.github/workflows/Mirrormaker2-setup-prod.yaml - Configure kafka cluster  ( Mirrormaker2 pod)** ( If not using mirrormaker please move to next step)

```bash
name: Mirror-maker Deployment (Prod)

on: workflow_dispatch

env:
  KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
  POOL_ID: "${{ vars.POOL_ID }}"

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: Prod-kafka

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure kubectl
        uses: Azure/k8s-set-context@v1
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_DATA }}

      - name: Check if Kafka namespace exists
        run: |
            if kubectl get namespace kafka > /dev/null 2>&1; then
              echo "Kafka namespace already exists."
            else
              kubectl create namespace kafka
            fi

      - name: Check if MirrorMaker2 deployment exists
        run: |
          if kubectl get deployment -n kafka mirrormaker2 > /dev/null 2>&1; then
            echo "MirrorMaker2 deployment already exists."
          else
            echo "MirrorMaker2 deployment does not exist. Deploying..."
            envsubst < kafka/prod/mirrormaker2-prod.yaml > kafka/prod/mirrormaker2-prod-substituted.yaml
            sed -i "s/\$POOL_ID/${{ vars.POOL_ID }}/g" kafka/prod/mirrormaker2-prod-substituted.yaml
            kubectl apply -f kafka/prod/mirrormaker2-prod-substituted.yaml -n kafka
          fi

      - name: Wait for MirrorMaker2 pods to be running
        run: |
          retries=5   # Number of retries
          wait_time=10 # Wait time between retries in seconds
          for ((i=0; i<$retries; i++)); do
            if kubectl get pods -n kafka -l app=mirrormaker2-pod --field-selector=status.phase=Running | grep -q '1/1'; then
              echo "All MirrorMaker2 pods are running."
              exit 0
            else
              echo "Waiting for MirrorMaker2 pods to be running..."
            fi
            sleep $wait_time
          done
          echo "MirrorMaker2 pods are not all running after waiting. Please check the deployment."
          exit 1
```

**Go to Github Actions and run the workflow**

![Screenshot from 2024-04-25 22-09-15](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/0ac98d19-ab3b-4ede-83dc-60e875b1565f)

Output

```bash
kubectl get pods -n kafka

mirrormaker2-5d7775f89f-nj7x2                                 1/1     Running   0            5d6h

```

step-3

**.github/workflows/Ingress-setup-pro.yaml → Configure Ingress**

```bash
name: Kafka ingress Setup(Prod)

on: workflow_dispatch

env:
  KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: Prod-kafka

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure kubectl
        uses: Azure/k8s-set-context@v1
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_DATA }}
      
      - name: Apply Ingress YAML
        run: |
          if kubectl get ingress -n kafka kafka-ingress > /dev/null 2>&1; then
            echo "Ingress already exists."
          else
            kubectl apply -f kafka/prod/ingress-prod.yaml -n kafka
          fi
```

**Go to Github Actions and run the workflow**

![Screenshot from 2024-04-25 22-10-06](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/b8108fbb-42c1-475e-b5b2-33190283442e)

![Screenshot from 2024-04-25 22-11-01](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/1bf7d620-2b7d-4ee5-be4e-aafc4b709f44)

Output

```bash
kubectl get ingress -n kafka
NAME            CLASS    HOSTS                ADDRESS          PORTS     AGE
kafka-ingress   <none>   xyz.example.com   172.232.87.254   80, 443   17d

```

step-4

**.github/workflows/Generate_secrets_ca_prod.yaml → creating kafkauers**

```bash
name: Generate_secret_Pass_kafkauser_prod

on:
  workflow_dispatch:
    inputs:
      num_users:
        description: 'Number of Kafka users to create'
        required: true
      user_names:
        description: 'Comma-separated list of Kafka user names'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: Prod-kafka
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure kubectl
        uses: Azure/k8s-set-context@v1
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_DATA }}

      - name: Set up Kafka Users and ACLs
        id: set_up_users_and_acls
        run: |
          echo "apiVersion: kafka.strimzi.io/v1beta2" > kafka_users.yaml
          echo "kind: KafkaUserList" >> kafka_users.yaml
          echo "items:" >> kafka_users.yaml
          IFS=',' read -ra USER_NAMES <<< "${{ github.event.inputs.user_names }}"
          for name in "${USER_NAMES[@]}"; do
              echo "- apiVersion: kafka.strimzi.io/v1beta2" >> kafka_users.yaml
              echo "  kind: KafkaUser" >> kafka_users.yaml
              echo "  metadata:" >> kafka_users.yaml
              echo "    name: $name" >> kafka_users.yaml
              echo "    labels:" >> kafka_users.yaml
              echo "      strimzi.io/cluster: coto-kafka-prod-cluster" >> kafka_users.yaml
              echo "  spec:" >> kafka_users.yaml
              echo "    authentication:" >> kafka_users.yaml
              echo "      type: scram-sha-512" >> kafka_users.yaml
          done
          echo "Kafka users YAML file generated successfully!"
          echo "::set-output name=kafka_users_yaml::kafka_users.yaml"
          
      - name: Apply Kafka Users YAML
        run: |
          kubectl apply -f ${{ steps.set_up_users_and_acls.outputs.kafka_users_yaml }} -n kafka

      - name: Install Linode CLI (linode-cli)
        run: |
          if ! command -v linode-cli &> /dev/null; then
            echo "Linode CLI is not installed. Installing..."
            pip3 install linode-cli
            pip3 install boto3
          else
            echo "Linode CLI is already installed."
          fi

      - name: Get secrets
        run: |
          IFS=',' read -ra USER_NAMES <<< "${{ github.event.inputs.user_names }}"
          for user_name in "${USER_NAMES[@]}"; do
          kubectl get secret "$user_name" -n kafka -o jsonpath='{.data.sasl\.jaas\.config}' | base64 -d > "${user_name}_sasl_config.txt"
          done

      - name: Upload secrets to Linode Object Storage
        env:
          LINODE_CLI_TOKEN: ${{ secrets.LINODE_CLI_TOKEN }}
          LINODE_CLI_OBJ_ACCESS_KEY: ${{ secrets.LINODE_ACCESS_KEY }}
          LINODE_CLI_OBJ_SECRET_KEY: ${{ secrets.LINODE_SECRET_KEY }}
        run: |
          IFS=',' read -ra USER_NAMES <<< "${{ github.event.inputs.user_names }}"
          for user_name in "${USER_NAMES[@]}"; do
          linode-cli obj put --cluster in-maa-1 ${user_name}_sasl_config.txt "coto-wrld/prod-kafka-details"
          done
```

**Go to Github Actions and run the workflow**

![Screenshot from 2024-04-25 22-12-41](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/860b0411-0db4-480d-ad23-047c5ea1a353)

![Screenshot from 2024-04-25 22-13-16](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/139b1dea-af52-4eed-bf19-61664a86b0e5)

![Screenshot from 2024-04-25 22-15-30](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/4de9ac94-5a2a-463d-9e49-325d337cd11d)

Output

step-5

**.github/workflows/Delete_kafka_users_prod.yaml —> delete existing kafkausers**

```bash
name: Delete_Kafka_Users_prod

on:
  workflow_dispatch:
    inputs:
      delete_users:
        description: 'Comma-separated list of Kafka users to delete'
        required: true

env:
  KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}

jobs:
  delete_users_job:
    runs-on: ubuntu-latest
    environment: Prod-kafka
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure kubectl
        uses: Azure/k8s-set-context@v1
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_DATA }}
          
      - name: Install Linode CLI (linode-cli)
        run: |
          if ! command -v linode-cli &> /dev/null; then
            echo "Linode CLI is not installed. Installing..."
            pip3 install linode-cli
            pip3 install boto3
          else
            echo "Linode CLI is already installed."
          fi
      
      - name: Delete Kafka Users
        run: |
          IFS=',' read -ra DELETE_USERS <<< "${{ github.event.inputs.delete_users }}"
          for user in "${DELETE_USERS[@]}"; do
            kubectl delete kafkauser $user -n kafka
          done
      
      - name: Delete uploaded sasl_config from Linode Object Storage
        env:
          LINODE_CLI_TOKEN: ${{ secrets.LINODE_CLI_TOKEN }}
          LINODE_CLI_OBJ_ACCESS_KEY: ${{ secrets.LINODE_ACCESS_KEY }}
          LINODE_CLI_OBJ_SECRET_KEY: ${{ secrets.LINODE_SECRET_KEY }}
        run: |
          IFS=',' read -ra DELETE_USERS <<< "${{ github.event.inputs.delete_users }}"
          for user in "${DELETE_USERS[@]}"; do
            linode-cli obj del --cluster in-maa-1 "coto-wrld" "prod-kafka-details/${user}_sasl_config.txt"
          done
```

**Go to Github Actions and run the workflow**

![Screenshot from 2024-04-25 22-18-58](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/2ed46725-22d2-4a84-b55c-fc99d0ee9c82)


Note:- 

Kindly set the Environment secrets and variables in Github settings.

![Screenshot from 2024-04-25 22-41-44](https://github.com/Eve-World-Platform/deployment-pipelines-airflow-kafka/assets/112183257/3df930d4-b080-4be3-815e-1745f2b39155)