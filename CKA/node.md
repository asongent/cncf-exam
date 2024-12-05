## ETCD Installation

```
wget https://github.com/etcd-io/etcd/releases/download/v3.5.16/etcd-v3.5.16-linux-amd64.tar.gz
tar xzvf etcd-v3.5.16-linux-amd64.tar.gz
sudo mv etcd-v3.5.16-linux-amd64/etcdctl /usr/local/bin
sudo mv etcd-v3.5.16-linux-amd64/etcdutl /usr/local/bin
```

### How to issue a certificate for a user
- Grant access to user called honor to kubernetes cluster
- 1) Create private key

```bash
openssl genrsa -out honor.key 2048
```
```bash
openssl req -new -key honor.key -out honor.csr -subj "/CN=honor"
```
- Get Base64 encoded value of `honor.csr`

 ```bash
 cat honor.csr| base64 | tr -d "\n"
 ```
- Create a CertificateSigningRequest
 NB: 

  ```bash
  .
  .
  spec:
    request: cat honor.csr| base64 | tr -d "\n"
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: honor
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZUQ0NBVDBDQVFBd0VERU9NQXdHQTFVRUF3d0ZhRzl1YjNJd2dnRWlNQTBHQ1NxR1NJYjNEUUVCQVFVQQpBNElCRHdBd2dnRUtBb0lCQVFDeDhLTnVQVjN5VVZ1Q0ZGUEZmYlJMNlNRVjlwUHpaRUd5RUF0djBYWW50bmw5Cmtad29RM0lPNXF5NGtGZDY1QXNUMDErQjdXMmlKQkdPVDZpSjlNdnlodDlCZVY2MFpJQ3hiWmJid21NazdCbzIKcEQ3Mlo5T095cEtvdTBhS2hwVUs4QWxIUGVaUkZwNGxCcmdsY2pIRHZ4d0pwN0ttSGx2RStSOSt1eHJVMDVpWApUdDA4bDhHS0FaV2RGSk1NTDdVeXJvOFA5Mlh1MXViYmVDU3ZLSkdEY2VBWURTbGtuRzNHc0lXRld4T2c2Vi9YCmgxeGFOeFJGL3dIYW01blNjYmtTUEp4TnVuTGkwZ2hMeTlnR3pGNS9iTkY1MkJmeU1vWlhncTlKcC96OTZ1U00KcHJ2eHYwaFNWS2hMTzY3T0QvNFp5NXFMblhUQTR5U2JhYlprbVBLNUFnTUJBQUdnQURBTkJna3Foa2lHOXcwQgpBUXNGQUFPQ0FRRUFBM3Vka09tOFpHaWJ0QlN5bnB5Mjc2SG91SmtzWHJGNXBDcHhBRjdscXFVcTkwZDk3TWYwClpKUzlkM21GcGEzVWdLVFdPMUJ1ZWhHM0dNejcvSXdMUS9GdWtMb0lEUVIraVBGVGQ1S3NWMDZvb3REWFdxUngKUEIwOERYaEhnWDlyTGJ2czE4bGNrUzN2ZlVsK0Z2TUp6Wmxxd3pVVDBMdC9HS0JJVnJoRTI1QjVQbU4yNHdxRApZYmNCZzhKQjgxMmVQaXphVlhSYk9QUVU2alFhUVF5M1RSMmgwQ0xySmUrdXdEL2IvL3NzT0Q4UXZuUUFtUXRYCmpIZ2dXZjJjTDlVWXVYWC9vMElDVitFZFlTQndYZmNvL3p6NlNtSnd5STMrN2hIeVgyWFZlNmh5QUl3MTJPRmcKV1BhOEVLbDB0ejdTRG5oTlM5UWpFVXJieEg1d2dnY3Y5UT09Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  #expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
```

- Get the certificate

```bash
kubectl get csr honor -o yaml
```
- Export the issued certificate from the CertificateSigningRequest

```bash
kubectl get csr honor -o jsonpath='{.status.certificate}'| base64 -d > honor-ca.crt
```
- Create `namespace` and `service account` 
```bash
kubectl create ns developers
kubectl create sa developers-sa -n developers
```
- Create Role for user honor

```bash
kubectl -n developers create role developer \
--verb=create --verb=get --verb=list --verb=update --verb=delete  \
--resource=pods --resource=deployment --resource=secret --resource=service \
--resource=serviceaccount --resource=statefulset --resource=configmap \
--resource=persistentvolumeclaim --resource=persistentvolume --dry-run=client -oyaml 
```
- create a RoleBinding for this new user honor

```bash
kubectl -n developers create rolebinding developer-binding-honor \
--role=developer --user=honor --serviceaccount=developers:developers-sa --dry-run=client -oyaml  
```
- Add to kubeconfig

```bash
kubectl config set-credentials honor --client-key=honor.key --client-certificate=honor-ca.crt --embed-certs=true
```
- Add to the context

```bash
kubectl config set-context developers --cluster=kubernetes --user=honor
kubectl config use-context developers
```
