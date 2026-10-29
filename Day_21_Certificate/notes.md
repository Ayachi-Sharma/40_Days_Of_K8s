# Kubernetes CertificateSigningRequest (CSR)

## 1. Introduction

Generate a private key and Certificate Signing Request (CSR), submit the CSR to Kubernetes, approve it, and retrieve the signed certificate.
A **CertificateSigningRequest (CSR)** is a Kubernetes resource used to request a signed certificate from a certificate authority (CA).

## 2. Generate the Private Key and CSR

### Step 1: Generate the private key

*Note: openssl must be installed in your PC*

Generate a 2048-bit RSA private key using OpenSSL:

```bash
openssl genrsa -out learner.key 2048
```

This creates `learner.key`, which contains the private key.

### Step 2: Generate the CSR

```bash
openssl req -new -key learner.key -out learner.csr -subj "/CN=learner"
```

This creates `learner.csr` using the private key.

- `-new` creates a new CSR.
- `-key learner.key` specifies the private key.
- `-out learner.csr` specifies the output file.
- `-subj "/CN=learner"` sets the certificate subject's Common Name to `learner`.

### Step 3: Verify the generated files

```powershell
dir learner.*
```

You should see:

- `learner.key` — private key
- `learner.csr` — certificate signing request

Inspect the CSR:

```bash
openssl req -in learner.csr -noout -text
```

## 3. Encode the CSR

### Objective

Encode the CSR using Base64 because Kubernetes expects the `spec.request` field to contain the Base64-encoded CSR data.

In PowerShell, run:

```powershell
$csr = [Convert]::ToBase64String([IO.File]::ReadAllBytes("$PWD/learner.csr"))
```

Display the encoded value:

```powershell
$csr
```

Copy the complete output. You will use it in the Kubernetes CSR manifest. Paste it at request in the yaml file.

**Important:** Encode the contents of `learner.csr`, not the filename or the private key.

## 4. Create the Kubernetes CertificateSigningRequest

### Step 1: Create the manifest

Create a file named `learner-csr.yaml` and insert the Base64-encoded CSR value into the `request` field.

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: learner
spec:
  request: <BASE64_ENCODED_CSR>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 604800
  usages:
    - client auth
    - digital signature
    - key encipherment
```

Replace `<BASE64_ENCODED_CSR>` with the complete Base64 output from the previous step, without the angle brackets.

### Explanation

- `certificates.k8s.io/v1` is the API version for Kubernetes CSR resources.
- `name: learner` names the CSR resource.
- `request` contains the Base64-encoded CSR.
- `signerName: kubernetes.io/kube-apiserver-client` requests a client certificate for Kubernetes API authentication.
- `expirationSeconds: 604800` requests a certificate lifetime of one week.
- `usages` specifies the intended uses of the certificate.

The actual certificate lifetime depends on the configured signer and cluster settings.

### Step 2: Submit the CSR

```powershell
kubectl apply -f learner-csr.yaml
```

### Step 3: Check the CSR status

```powershell
kubectl get csr
kubectl describe csr learner
```

The CSR should initially have a `Pending` approval status.

## 5. Approve the CSR

### Step 1: Approve the request

```powershell
kubectl certificate approve learner
```

### Step 2: Verify the approval

```powershell
kubectl get csr learner
```

You should see `Approved` in the CSR's conditions.

**Explanation:** Approval authorizes the certificate request. A configured signer must still issue the certificate for it to appear in the CSR's status.

## 6. Retrieve the Issued Certificate

### Step 1: Retrieve the certificate value

```powershell
kubectl get csr learner -o jsonpath='{.status.certificate}'
```

The certificate is stored in `.status.certificate` as a Base64-encoded value.

If the value is empty, inspect the CSR again:

```powershell
kubectl describe csr learner
```

Check the approval conditions and whether the signer has issued a certificate.

### Step 2: Export the certificate as YAML

```powershell
kubectl get csr learner -o yaml > learner-issued.yaml
```

This saves the full Kubernetes CSR resource, including its status and the issued certificate if available.

## 7. Decode the Certificate Into `learner.crt`

### Step 1: Decode and save the certificate

In PowerShell, run:

```powershell
$cert = kubectl get csr learner -o jsonpath='{.status.certificate}'
[IO.File]::WriteAllBytes("$PWD/learner.crt", [Convert]::FromBase64String($cert))
```

This decodes the certificate and writes the resulting PEM data to `learner.crt`.

### Step 2: Verify the certificate

```bash
openssl x509 -in learner.crt -noout -text
```

To inspect its subject and validity dates:

```bash
openssl x509 -in learner.crt -noout -subject -dates
```

You should see the certificate subject, issuer, validity period, and other certificate details.

## 8. Key Learnings

1. A private key is generated locally and must be kept secret.
2. A CSR contains the public key and requested identity information.
3. Kubernetes requires the CSR request data to be Base64-encoded.
4. `expirationSeconds: 604800` requests a one-week certificate lifetime.
5. `kubectl certificate approve` approves the CSR but does not itself sign the certificate.
6. The configured signer issues the certificate after approval.
7. `.status.certificate` contains the issued certificate in Base64-encoded form.
8. The decoded certificate is saved as `learner.crt`, while `learner.key` remains the corresponding private key.