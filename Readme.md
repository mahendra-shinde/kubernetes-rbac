# Hands On Lab : Kubernetes RBAC 

## Pre-requisites:

* Docker desktop with kubernetes 
* kubectl (pre-installed / bundled with docker-desktop)
* WSL (Windows Subsystem for Linux ) 
* VS Code (Text editor)

## Part 1 : New User Certificate

> Use `Windows Subsystem for Linux` as all of the utilities (sed, base64 and openssl) are pre-installed on WSL.

1. Create a new certificate for new user `mahendra` 

	```bash
	# RSA key with 2KB size with name `mahendra`
	openssl genrsa -out mahendra.key 2048
	# Create Certificate Signing request with Subject line "/CN=mahendra" which is MANDATORY for kubernetes authentication 
	openssl req -new -key mahendra.key  -out mahendra.csr -subj="/CN=mahendra"
	# list all the files
	# expected set of files : mahendra.key and mahendra.csr
	ls
	```

1. Convert the CSR into `base64` encoded environment variable

	```bash
	CSR=$(cat mahendra.csr | base64 --wrap 0 )
	# Just verify if variable contains the base64 encoded cert
	echo $CSR
	```

1. Now, create or use following YAML file for `certificate signing request` the file-name should be `sign-request.yml`

	> you can download my [sample file](./manifests/sign-request.yml)

	```yaml
	apiVersion: certificates.k8s.io/v1
	kind: CertificateSigningRequest
	metadata:
	# specify user-name
	name: mahendra
	  spec:
        request: $CSR
  	    signerName: kubernetes.io/kube-apiserver-client
  	    expirationSeconds: 86400  # one day
  	    usages:
  	    - client auth
	```


1.	The file `sign-request.yml` contains one environment variable $CSR which needs to be replaced with its value. 

	```bash
	sed -i "s/\$CSR/$CSR/g" sign-request.yml
	```

1. Use following command to issue request to `kubernetes certificate signing controller` 
	
	> You must be logged in with default admin user 
	> In docker-desktop, the default user already has admin privileges.

	```bash
	kubectl apply -f sign-request.yml
	 kubectl certificate approve mahendra
	```

1.  Extract the signed certificate from kubernetes and store it on local system as `*.crt` file.

	> This step is needed for the next phase `Authentication`


	```bash
	kubectl get csr mahendra -o jsonpath='{.status.certificate}'| base64 -d > mahendra.crt
	## Verify if CRT file exists
	cat mahendra.crt
	```


## Part 2 : Authentication with new user

1. Update your kube-config file with new-user and his/her credentials.

	```bash
	kubectl config set-credentials mahendra --client-key=mahendra.key --client-certificate=mahendra.crt --embed-certs=true
	
	```

1. Define a new context that binds your `docker-desktop` cluster with new user `mahendra`

	```bash
	kubectl config set-context reader --cluster=docker-desktop --user=mahendra
	```

1.	Check the modified configuration

	```bash
	kubectl config view
	kubectl config get-contexts
	```

1.  Switch to new `reader` context and test performing few operations.

	```bash
	kubect config use-context reader
	kubectl auth can-i get pod
	kubectl auth can-i create pod
	## You must get "no" response for both tests !
	```



1.	Kindly switch back to original context `docker-desktop` which has admin role.

	```
	kubectl config use-context docker-desktop
	kubectl auth can-i get pod
	kubectl auth can-i create pod
	## You must get "yes" response for both tests !
	```

### Part 3 : RBAC Allow GET, List and Watch resources in all namespaces.

1. Create a new file `reader-cluster-role.yaml` with following lines :

	> you can download my [sample file](./manifests/reader-cluster-role.yaml)

	```yml
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRole
	metadata:
  	  name: cluster-reader
	rules:
	- apiGroups: [""]
      resources:
      - "*"
      verbs:
      - "get"
      - "list"
      - "watch"
	```

1. Create another file `reader-cluster-role-binding.yml` for assigning the reader role to mahendra user.

	> you can download my [sample file](./manifests/reader-cluster-role-binding.yml)

	```yaml
	apiVersion: rbac.authorization.k8s.io/v1
	kind: RoleBinding
	metadata:
      name: reader
	subjects:
	- kind: User
	  name: mahendra
	roleRef:
  	  apiGroup: rbac.authorization.k8s.io
  	  kind: ClusterRole
  	  name: cluster-reader
	```

1. Apply both the role and role-bindings

	```bash
	kubectl apply -f reader-cluster-role.yaml
	kubectl apply -f reader-cluster-role-binding.yml
	```

1.	Verify if default `docker-desktop` user can view all pods

	```
	kubectl config use-context docker-desktop
	kubectl auth can-i get pod 
	```

1.	Verify if new `mahendra` user in `reader` context can get all pods.

	```
	kubectl config use-context reader
	kubectl auth can-i get pod 
	```

1.	Verify if default `docker-desktop` user can create new pods

	```
	kubectl config use-context docker-desktop
	kubectl auth can-i create pod 
	```

1.	Verify if new `mahendra` user can create new pods

	```
	kubectl config use-context reader
	kubectl auth can-i create pod 
	```

---

### Sources 
- [Kubeernets Docs : Certificate Signing](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
- [Kubeernets Docs : Authorization (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)