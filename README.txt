Minikube. Using Keycloak operator with unsecure keycloak instance.
We need to create a nodePort service in order to have the keycloak server URL to be resolved both in the WildFly server pod
and outside of the cluster. That is required to create the SAML client from the WildFly server pod.
Ingress is not a solution there, we don't have ingress host to be resolved in pod.


* Build the server locally.

mvn clean package -Popenshift

* Build the image

docker build -t saml-app:latest .

* Start minikube

minikube start --memory='4gb'

* Load the image in local registry

minikube image load saml-app:latest

* Install keycloak operator

Follow Operator installation: https://www.keycloak.org/operator/installation

* Deploy a keycloak instance 

kubectl apply -f example-postgres.yaml

kubectl create secret generic keycloak-db-secret --from-literal=username=postgres --from-literal=password=testpassword

kubectl apply -f keycloak-instance.yaml

* Create a nodePort service for keycloak server (on port 30080)

oc apply -f keycloak-nodeport-service.yaml

* Create the Keycloak Realm and users required by this example

kubectl apply -f keycloak-saml-realm.yaml

* Create the client keystore

keytool -genkeypair -alias saml-app -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore keystore.p12 -storepass password -dname "CN=saml-basic-auth,OU=EAP SAML Client,O=Red Hat EAP QE,L=MB,S=Milan,C=IT" -ext ku:c=dig,keyEncipherment -validity 365

keytool -importkeystore -deststorepass password -destkeystore keystore.jks -srckeystore keystore.p12 -srcstoretype PKCS12 -srcstorepass password'

* Create the keystore secret that will be mounted in deployment

kubectl create secret generic saml-app-secret --from-file=keystore.jks=./keystore.jks --type=opaque

* Create the deployment environment variables secret

export MINIKUBE_IP=$(minikube ip)
cat <<EOF >> saml-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: saml-secret
type: Opaque
stringData:
  SSO_REALM: "saml-basic-auth"
  SSO_USERNAME: "client"
  SSO_PASSWORD: "creatorpassword"
  SSO_SAML_CERTIFICATE_NAME: "saml-app"
  SSO_SAML_KEYSTORE: "keystore.jks"
  SSO_SAML_KEYSTORE_PASSWORD: "password"
  SSO_SAML_KEYSTORE_DIR: "/etc/sso-saml-secret-volume"
  SSO_SAML_LOGOUT_PAGE: "/saml-app"
  SSO_DISABLE_SSL_CERTIFICATE_VALIDATION: "true"
  HOSTNAME_HTTP: "localhost:8080"
  SSO_URL: http://$MINIKUBE_IP:30080
EOF

oc apply -f saml-secret.yaml


* Deploy the application

helm install saml-app -f helm.yaml wildfly/wildfly

* Patch the deployment in order to not pull the local image

kubectl patch deployment saml-app -p '{"spec": {"template": {"spec":{"containers":[{"name":"saml-app","imagePullPolicy":"Never"}]}}}}'

kubectl port-forward deployment/saml-app 8080:8080&

In web browser http://localhost:8080/saml-app
Click on "Access Secured Servlet"
Log as user, password user
You will see the Prinicpal id displayed.
Click on logout to logout


