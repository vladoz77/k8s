# https://headlamp.dev/docs/latest/installation/in-cluster/keycloak/
# Create provider and application in authentik

# Config api-server in minikube to connect with oidc
minikube start -p=keycloak \
--extra-config=apiserver.authorization-mode=Node,RBAC \
--extra-config=apiserver.oidc-issuer-url=http://oauth.home.local/application/o/minikube/ \
--extra-config=apiserver.oidc-username-claim=email \
--extra-config=apiserver.oidc-client-id=iQKESUNGE4Y2piX6NnxeIJ6jzHvXSLvf2RM2Yic
