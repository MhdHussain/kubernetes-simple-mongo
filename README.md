## Simple Mongo Setup in kubernetes

This project is inspired by the Jana Janashia's popular kubernetes video on youtube
https://www.youtube.com/watch?v=X48VuDVv0do

### Structure

In this exercise we will create a simple setup of a **Mongo** database and a mongo express to browse that database in the browser.

#### Clarification Example

Lets say there is an employee **Adam**, who has a phone on his desk. He can tell other empolyees to call him on this number 27017. however if the phone does not have a service , they will not be able to call him, so the system team has to setup his phone for him so that if anyone call this number 27017 it will ring.
In order to keep things private at the workplace this number is accessible only internally, no one from outside the workplace can call the same number to talk to Adam.

**Adam** cannot serve the public directly because he is handling sensitive information and access to him has to be properly proxied. So in order for him to give you the information you need, you need to provide the password. Only internal personel has access to that password.

The Organization realized that the public need access to Adam but they cannot give direct access to his phone, he will be fludded with requests not to mention that giving public access will require exposing the password to the public. To resolve that they decided to setup another department which can be accessed by the public. The department employees will take the public requests and route them internally to Adam without revealing his password. Adam can then respond only to this department and the department can respond to the public accordingly.

In order for this Department to be accessible by the public they need a special number and a special phone service to advertise this number.

![[mongo-concept-example.png]]

#### Components

1. MongoDB: this will be a deployment which has a single replica ( to keep things simple). the container of the pod will listen on port **27017**
2. mongoDB Internal Service: this service is listening on port 27017 and forewarding to the pod IP on the same port. The service is an internal Service which can only be accessed from within the cluster, therefore it is of type **ClusterIP**
3. ConfigMap: this will act as the registry to store the URL of the database so that other services can utilize it and it can be a single source of truth. if the db url changes we can change it in one place without affecting the applications or services as they are accessing it using the key.
4. Secret: this will contain the database username and password base64 encoded
5. MongoExpress: this is the application ( department ) used to allow access to mongo db. it will need to utilize the config map and the secrets. it is exposed on port **8081**
6. Mongo Express External Service: this is an external service that allows access to the MongoExpress interface via the browser on port **30400**
   ![[Diagram.svg]]

#### Mongo DB deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mongo
  name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  strategy: {}
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - image: mongo
          name: mongo
          ports:
            - containerPort: 27017
```

to generate the above yaml we can use the following command

```bash
 kubectl create deployment mongo --image=mongo --replicas=1 --port=27017 --dry-run=client -o yaml > mongo.yaml
```

we have to create the deployment before we can expose it via a service so we have to run the following command to apply it

```bash
kubectl apply -f mongo.yaml
```

#### Mongo DB Service

now to expose the mongo db and allow access to it we have to create a service
we can generate the yaml file with the following command

```bash
kubectl expose deployment mongo --type=ClusterIP --port=27017 --dry-run=client -o yaml
```

this will generate the yaml for the service that will expose the deployment's pods and allow access to them. it is important to note that this command will not work unless the deployment has already been created.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mongo
  name: mongo
spec:
  ports:
    - port: 27017
      protocol: TCP
      targetPort: 27017
  selector:
    app: mongo
  type: ClusterIP
```

#### ConfigMap and Secret

The above deployment will not work as it does not contain the appropriate environment variables for Mongo to work. As per the dockerhub documentation we need 2 environment variables :

```console
MONGO_INITDB_ROOT_USERNAME

MONGO_INITDB_ROOT_PASSWORD
```

##### Secret

We will create the secret to containt these values and then update our deployment file to reference them.

in order to create the secret we need to run the following command

```bash
k create secret generic mongo-secret --from-literal MONGO_INITDB_ROOT_USERNAME=bW9uZ28tdXNlcg== \
--from-literal MONGO_INITDB_ROOT_PASSWORD=bW9uZ28tcGFzcw== --dry-run=client -o yaml > mongo-secret.yaml

```

as you can see the values are base64 encoded and we have to encode them before creating the secret. the above command will generate the yaml file and will not create the secret. this is ententional as we will create the deployment later.

To generate a base64 string out of your secret values , you need to use the following command on a linux like system

```bash
echo -n mongo-user | base64
echo -n mongo-pass | base64
```

The above 2 commands will generate our username and password for the secret.

Now that we have our secret, lets update our deployment to use it

We need to add an **env section under the container that needs this secret** and pull the environment variables from the secret

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:

  labels:
    app: mongo
  name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  strategy: {}
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - image: mongo
        name: mongo
        ports:
	        - containerPort: 27017
        envFrom:
		    - secretRef:
				name: mongo-secret


```

### Mongo Express

Now that the mongo db is created and the connection config is available in a secret. we can create the mongo express application which will allow users to access the database through a web interface.

The mongo express needs the same data available in the secret but with a different key. also it needs a database URL. For that reason we can do one of the following

1. create a separate secret for mongo express which hosts the same values in the mongo secret
2. utilize the same secret: this will require changing the deployment for mongo as it will not use envFrom and secretRef. The deployment config will specify the key it needs and the value will be the key within the secret.

Lets go with Option 2

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:

  labels:
    app: mongo
  name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  strategy: {}
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - image: mongo
        name: mongo
        ports:
	        - containerPort: 27017
        env:
          - name: MONGO_INITDB_ROOT_USERNAME
		    valueFrom:
		        secretKeyRef:
		           name: mongo-secret
		           key: MONGO_INITDB_ROOT_USERNAME
		  - name: MONGO_INITDB_ROOT_PASSWORD
		    valueFrom:
		        secretKeyRef:
		           name: mongo-secret
		           key: MONGO_INITDB_ROOT_PASSWORD
```

Now we can use the same secret in mongo-express just changing the name as required by mongo-express.

##### Config Map

Another requiremet for the mongo-express setup is the database URL which can be stored in a config map so lets create the config map

We can generate the config map yaml file by the following command

```bash
k create cm mongo-config-map --from-literal ME_CONFIG_SITE_BASEURL=mongo --dry-run=client -o yaml > mongo-config-map.yaml
```

This will generate the following file

```yaml
apiVersion: v1
data:
	ME_CONFIG_SITE_BASEURL: mongo
kind: ConfigMap
metadata:
	name: mongo-config-map
```

Now we can create the config map by applying the following command

```bash
k apply -f mongo-config-map.yaml
```

##### Mongo Express Deployment

We can now start creating the mongo express deployment

to generate the yaml file for the deployment we can use the following command

```bash
k create deploy mongo-express --image=mongo-express --replicas=1 --port=8081 --dry-run=client -o yaml > mongo-express.yaml
```

This will create the deployment but without any environment variables. so we need to add them

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: mongo-express
  name: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mongo-express
    spec:
      containers:
        - image: mongo-express
          name: mongo-express
          ports:
            - containerPort: 8081
          env:
            - name: ME_CONFIG_MONGODB_ADMINUSERNAME
              valueFrom:
                secretKeyRef:
                  key: MONGO_INITDB_ROOT_USERNAME
                  name: mongo-secret
            - name: ME_CONFIG_MONGODB_ADMINPASSWORD
              valueFrom:
                secretKeyRef:
                  key: MONGO_INITDB_ROOT_PASSWORD
                  name: mongo-secret
            - name: ME_CONFIG_MONGODB_SERVER
              valueFrom:
                configMapKeyRef:
                  key: ME_CONFIG_MONGODB_SERVER
                  name: mongo-config-map
```

### Mongo Express Service

Now that we have the mongo express up and running , we need to expose it to the outside world. We can achive that by creating a service of type **NodePort**

We can generate a draft of the service by using the following command

```bash
k expose deployment mongo-express --type=NodePort --port=8081  --dry-run=client -o yaml > mongo-express-service.yaml
```

This file is incomplete as it does not contain the nodePort so we have to modify it
here is the final version:

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: mongo-express
  name: mongo-express
spec:
  ports:
    - port: 8081
      protocol: TCP
      targetPort: 8081
      nodePort: 30400
  selector:
    app: mongo-express
  type: NodePort
```

After applying the service we will find out that the external IP is none.
Now the only thing that is left is to configure the External IP.
we can do this in Minikube by running the following command

```bash
minikube service mongo-express
```

it will configure the External IP and will open Mongo Express in the browser
