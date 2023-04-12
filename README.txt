Minikube, WildFly application secured with Keycloak SAML

* Using the Keycloak operator to deploy a secured Keycloak instance (TLS, strict hostname).
* Using ingress and ingress-dns outside and inside the cluster to resolve 'test.keycloak.org' domain.
* Using WildFly Maven plugin to provision WildFly and Keycloak SAML Galleon feature-packs, 
generates docker image and push it to minikube registry.


1- Setup Kubernetes

* Start minikube

minikube start --memory='4gb'

* Enable docker registry

minikube addons enable registry

* Allow to push to minikube docker registry

kubectl port-forward --namespace kube-system service/registry 5000:80 &

* Enable ingress and ingress-dns 

minikube addons enable ingress

minikube addons enable ingress-dns

* Configure DNS on host and inside the cluster. Make sure to use "test.keycloak.org" domain.

Follow these instructions: https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/


2- Setup Keycloak

* Install keycloak operator

Follow Operator installation: https://www.keycloak.org/operator/installation

* Deploy a keycloak instance 

Follow these instructions: https://www.keycloak.org/operator/basic-deployment

* Create the Keycloak Realm and users required by this example

kubectl apply -f keycloak-saml-realm.yaml

* Create the SAML client keystore

keytool -genkeypair -alias saml-app -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore keystore.p12 -storepass password -dname "CN=saml-basic-auth,OU=EAP SAML Client,O=Red Hat EAP QE,L=MB,S=Milan,C=IT" -ext ku:c=dig,keyEncipherment -validity 365

keytool -importkeystore -deststorepass password -destkeystore keystore.jks -srckeystore keystore.p12 -srcstoretype PKCS12 -srcstorepass password'

* Create the keystore secret that will be mounted in deployment

kubectl create secret generic saml-app-secret --from-file=keystore.jks=./keystore.jks --type=opaque


3- Build and Run the example

* Build the server, server image and push it to minikube local repository.

mvn clean package wildfly:image


* Deploy the application

helm install saml-app -f helm.yaml wildfly/wildfly

* Make the application available on localhost:8080

kubectl port-forward deployment/saml-app 8080:8080&

4- Access the application

* In web browser http://localhost:8080/saml-app
* Click on "Access Secured Servlet"
* Log as user, password user
* You will see the Prinicpal id displayed.
* Click on logout to logout
