#Add repo
helm repo add codecentric https://codecentric.github.io/helm-charts
helm  pull codecentric/keycloak  && tar -xzf keycloak-18.4.4.tgz 