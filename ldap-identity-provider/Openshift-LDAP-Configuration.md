# Openshift - LDAP Identity Provider Configuration

**Prerequisites**
- Hostname and port of LDAP
- If the LDAP directory requires authentication to search `bindPassword` and `baseDn` is required.
- Define id, email, name and preferredUsername attributes configured in LDAP.
- You must be logged in to the OpenShift Container Platform cluster as an administrator.

## 1. Create the LDAP Secret
If the LDAP directory requires authentication to search, create a _secret_ that contains `bindPassword` field in the _openshift-config_ namespace.

```bash
oc create secret generic ldap-secret --from-literal=bindPassword=<secret> -n openshift-config
```

## 2. Create a Config Map
In case of a TLS connection to the server, create a _configMap_ that contains `ca.crt` field in the _openshift-config_ namespace to validate server certificates for the configured URL.

```bash
oc create configmap ca-config-map --from-file=ca.crt=/path/to/ca -n openshift-config
```

## 3. Create a Custom Resource(CR)

### 3.1 CR yaml file sample
Below is a sample CR for an LDAP identity provider. Modify it according to your environment taking into account notes below.
```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: ldapidp
    mappingMethod: claim
    type: LDAP
    ldap:
      attributes:
        id: 
        - dn
        email: 
        - mail
        name: 
        - cn
        preferredUsername: 
        - uid
      bindDN: "" 
      bindPassword: 
        name: ldap-secret
      ca: 
        name: ca-config-map
      insecure: false 
      url: "ldaps://ldaps.example.com/ou=users,dc=acme,dc=com?uid" 
```

>**Notes:**
>- `name: ldapidp`  This provider name is prefixed to the returned user ID to form an identity name.
>- `mappingMethod` Controls how mappings are established between this provider’s identities and User objects.
>
>   | mappingMethod | Description                                                                                                                                                                                                                                                                                                                               | 
>   |---------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| 
>   | claim         | The default value. Provisions a user with the identity’s preferred user name. Fails if a user with that user name is already mapped to another identity.                                                                                                                                                                                  | 
>   | lookup        | Looks up an existing identity, user identity mapping, and user, but does not automatically provision users or identities. This allows cluster administrators to set up identities and users manually, or using an external process. Using this method requires you to manually provision users.                                           | 
>   | add           | Provisions a user with the identity’s preferred user name. If a user with that user name already exists, the identity is mapped to the existing user, adding to any existing identity mappings for the user. Required when multiple identity providers are configured that identify the same set of users and map to the same user names. |
>
>- `attributes` Ldap attributes mentioned in **Prerequisites**.
>- `bindDn`  Optional DN to use to bind during the search phase. Must be set if bindPassword is defined.
>- `bindPassword` Optional reference to an OpenShift Container Platform Secret object containing the bind password, mentioned in section **1. Create the LDAP Secret** . Must be set if bindDN is defined.
>- `ca` Optional: Reference to an OpenShift Container Platform ConfigMap object containing the PEM-encoded certificate authority bundle to use in validating server certificates for the configured URL , mentioned in section **2. Create a Config Map**. Only used when insecure is false.
>- `insecure` 	When true, no TLS connection is made to the server. When false, ldaps:// URLs connect using TLS, and ldap:// URLs are upgraded to TLS. This must be set to false when ldaps:// URLs are in use, as these URLs always attempt to connect using TLS.
>- `url` Specifies the LDAP host and search parameters to use.


### 3.2 Apply your CR yaml file
When your modified custom resource yaml file is ready, run below command. 

```bash
oc apply -f </path/to/CR>
```

## 4. Confirm Identity Provider Configuration
Log in to the cluster as a user from your identity provider, entering the password when prompted. 

```bash
oc login -u <username>
```
Confirm that your user logged in successfully, and then display the user name.

```bash
oc whoami
```

## 5. Create a New User in cluster-admin Role
```bash
oc adm policy add-cluster-role-to-user cluster-admin <user>
```

## 6. Removing the kubeadmin User
After you define your LDAP identity provider and create a new cluster-admin user, you can remove the kubeadmin to improve cluster security.

> **WARNING**: This is a warning message! Be careful and ensure below items applied before removing the kubeadmin user, otherwise OpenShift Container Platform must be reinstalled. It is not possible to undo this command.
>* At least identity provider is configured successfully.
>* The `cluster-admin` role is defines for a user.
>* You are logged in as an administrator.

```bash
oc delete secrets kubeadmin -n kube-system
```
