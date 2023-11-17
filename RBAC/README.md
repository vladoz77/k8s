Create user in kubernetes

1. Generate user.key
openssl genrsa -out user.key 2048

2. Create a certificate signing request
openssl req -new -key user.key -out user.csr -subj "/CN=user/O=support"

3. Create CSR object in k8s
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user-csr
spec:
  request: $(cat user.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000 # 1 year
  usages:
  - client auth
EOF

4. After that, approve the Certificate Signing Request using this command
kubectl certificate approve user-csr

5. Create Kubernetes Role and rolebinding

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: support-role
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: support-RoleBinding
subjects:
- kind: User
  name: user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: support-role

6. Get User Credential
kubectl get csr user-csr -o jsonpath='{.status.certificate}'| base64 -d > user.crt

7. Create new user with user.crt and user.key
kubectl config set-credentials user --client-certificate=user.crt --client-key=user.key

8. Create kube-context
kubectl config set-context user --cluster=minikube --user user


9. Use new context
k config use-context user





