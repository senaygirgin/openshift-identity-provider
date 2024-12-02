# Openshift - htpasswd Identity Provider Configuration on Linux

**Prerequisites**
- Have access to `htpasswd` utility, install httpd-tools package on On Red Hat Enterprise Linux.
  ```bash
  yum install httpd-tools
  ```
- You must be logged in to the OpenShift Container Platform cluster as an administrator.

## 1. Create the htpasswd file 
Create your flat file with username and hashed password using below command.

```bash
htpasswd -c -B -b </path/to/users.htpasswd> <username> <password>
```

Verify that your user is added to the htpasswd file. 

```bash
htpasswd -c -B -b users.htpasswd senay senay
cat users.htpasswd
senay:$2y$05$CIJ2/lfZGMNxd/4gmwd6muM1cDAaCwyGnv//q/lJOWiQAfvzHEZ4e
```

## 2. Create the htpasswd secret
When creating the secret, from file argument must be named `htpasswd` .

```bash
oc create secret generic htpass-secret --from-file=htpasswd=<path_to_users.htpasswd> -n openshift-config 
```

```bash
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n openshift-config 
oc get secret htpass-secret -n openshift-config
NAME            TYPE     DATA   AGE
htpass-secret   Opaque   1      1d
```

## 3. Adding htpasswd Identity Provider to your cluster

### 3.1 Create htpasswd Custom Resource(CR)
Below is a sample yaml file for an htpasswd identity provider. Modify it according to your environment taking into account notes below.
```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: my_htpasswd_provider
      mappingMethod: claim
      type: HTPasswd
      htpasswd:
        fileData:
          name: htpass-secret 
```

>**Notes:**
>- `name: my_htpasswd_provider`  This provider name is prefixed to the returned user ID to form an identity name.
>- `mappingMethod` Controls how mappings are established between this provider’s identities and User objects.
>
>   | mappingMethod | Description                                                                                                                                                                                                                                                                                                                               | 
>   |---------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| 
>   | claim         | The default value. Provisions a user with the identity’s preferred user name. Fails if a user with that user name is already mapped to another identity.                                                                                                                                                                                  | 
>   | lookup        | Looks up an existing identity, user identity mapping, and user, but does not automatically provision users or identities. This allows cluster administrators to set up identities and users manually, or using an external process. Using this method requires you to manually provision users.                                           | 
>   | add           | Provisions a user with the identity’s preferred user name. If a user with that user name already exists, the identity is mapped to the existing user, adding to any existing identity mappings for the user. Required when multiple identity providers are configured that identify the same set of users and map to the same user names. |
>
>- `type` It should be HTPasswd for htpasswd identity provider.
>- `name: htpass-secret` The secret name created at step 2.


### 3.2 Apply your CR yaml file
When your modified custom resource yaml file is ready, run below command. 

```bash
oc apply -f ./htpasswd-identity-provider.yaml
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

## 5. Add a New User for htpasswd Identitiy Provider
### 5.1 Retrieve Users from htpasswd provider.
Use your secret configured to hold users for htpasswd identitiy provider.
```bash
oc get secret <secret-name> -ojsonpath={.data.htpasswd} -n openshift-config | base64 --decode > users .htpasswd
```

### 5.2 To add a new user `senay` with password `senay` into htpasswd file.
Ensure that your user is added into the file in a new line. 
```bash
htpasswd -bB users.htpasswd senay senay
```
>sample htpasswd file;
> 
> kubeadmin:$2a$10$QfDj4.jw/OgNJzkBLOiny.rLfz1lV1OeFvC3Sk46KNLsJJ0cElCuu
> senay:$2y$05$ILmCySL9P7aFQgNIoqA9TeP4eB/Dj4mopaaKFHkPVLZ9R/e9zngAy

### 5.3 Replace secret object with the updated user file.
```bash
oc create secret generic <secret-name> --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml -n openshift-config | oc replace -f -
```
### 5.4 Confirm new user

```bash
oc login -u senay
oc whoami 
# senay
```

## 6. Remove Existing User from htpasswd Identitiy Provider
### 6.1 Retrieve Users from htpasswd provider.
Use your secret configured to hold users for htpasswd identitiy provider.
```bash
oc get secret <secret-name> -ojsonpath={.data.htpasswd} -n openshift-config | base64 --decode > users .htpasswd
```

### 6.2 To remove existing user `senay` from htpasswd file.
Ensure that your user is removed from the file users.htpasswd file.
```bash
htpasswd -D users.htpasswd senay
```

### 6.3 Replace secret object with the updated user file.
```bash
oc create secret generic <secret-name> --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml -n openshift-config | oc replace -f -
```
### 6.4 Remove Existing Resources 
Run below commands to remove existing resources for each user you removed.

```bash
oc delete user senay
# oc delete identity <idp name>:<username>
oc delete identity my_htpasswd_provider:senay
```