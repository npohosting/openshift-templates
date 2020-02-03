# Keycloak setup guide

## Think of a realm name
We are setting a rewrite rule in nginx to direct users to the default keycloak realm. \
You need the name of this realm before deploying the nginx. \
This can be edited in the configmap later.

## Create mariadb database
    $ oc new-app -n <namespace> mariadb-persistent -p MYSQL_DATABASE=keycloakdb

## Rollout nginx
    $ oc create -n <namespace> -f keycloak-nginx.yaml -p APP_HOSTNAME=<keycloak hostname> -p NAMESPACE=<namespace> -p KEYCLOAK_REALM=<default realm>

## Rollout keycloak
    $ oc create -n <namespace> -f keycloak-app.yaml -p NAMESPACE=<namespace> 

## Log in to the keycloak instance and set your admin password
Login on `https://<keycloak hostname>/auth/admin`

## Remove admin default password from deployment environments
    $ oc env -n <namespace> dc/keycloak KEYCLOAK_USER="" KEYCLOAK_PASSWORD="" 