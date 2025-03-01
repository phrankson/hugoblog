+++
title = 'Encrypting Kubernetes Secrets with SOPS: My Homelab Setup'
date = 2025-02-20T17:58:37+01:00
draft = false
+++


While managing my homelab, I quickly ran into a major challenge: handling secrets securely in a GitOps workflow. With Kubernetes and GitOps tools like Flux, all configurations—including secrets—are stored in a Git repository. However, committing secrets in plaintext is a security risk, and even base64-encoded secrets (as Kubernetes does by default) can be easily decoded.

After some research online, I quickly settled on  **SOPS (Secrets OPerationS)** with **Age** for encryption. This happened to be the best place to start for me. I plan to explore using Hashicorp Vault and Azure Secrets in the near future.

# Getting Started: Install SOPS and Age

First, install **SOPS** and **Age**, the lightweight encryption tool we’ll use.

For Debian-based systems:

```sh
sudo apt install sops
```

For macOS (with Homebrew):

```sh
brew install sops age
```

# Generating an Age Key Pair

Next, generate an Age key pair:

```sh
age-keygen -o age.agekey
```

To view the key:

```sh
cat age.agekey
```

This outputs both the public and private keys. The public key is used for encryption, while the private key is required for decryption.

> **Warning:** Store the private key securely in a password manager or a secrets vault.

# Encrypting Kubernetes Secrets

Let’s create a Kubernetes secret:

```sh
kubectl create secret generic test-secret \
  --from-literal=user=admin \
  --from-literal=password=mischa \
  --dry-run=client -o yaml > test-secret.yaml
```

If you check the file:

```sh
cat test-secret.yaml
```

You’ll see that Kubernetes encodes values using base64, which can be easily decoded:

```sh
echo 'encodedValueHere' | base64 -d
```

If we commit this secret to Git, it can be easily accessed and decrypted by anyone. To prevent this risk, we encrypt the secret using SOPS and Age.

# Encrypting with Age

First, export the public key:

```sh
export AGE_PUBLIC=<public-key-value>
```

Verify:

```sh
echo $AGE_PUBLIC
#check that the public key shown matches what is in they age.agekey file.
```

Now, encrypt the file:

```sh
sops --age=$AGE_PUBLIC \
  --encrypt --encrypted-regex '^(data|stringData)$' --in-place test-secret.yaml

#this command selectively encrypts sensitive data fields in `test-secret.yaml`while leaving the rest of the file structure readable.
```

If you check the file again:

```sh
cat test-secret.yaml
```

You’ll see the contents are now encrypted.

# Automating Secrets Decryption with Flux

Now the key to be committed to Git has been encrypted. but we need to allow Flux to decrypt the secrets to be used within the cluster. 
To allow Flux to decrypt secrets, we need to store the private Age key in a Kubernetes secret, this secret will only exist in the kubernetes cluster and not commited to Git. 

```sh
cat age.agekey | \
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

# Configuring SOPS for Flux

To enable Flux to find and decrypt secrets, you need to place a `.sops.yaml` configuration file in your Git repository.  This file MUST be stored in the Path where Flux will store cluster-specific manifests. 
This is same path that was used when bootstrapping FluxCD for your cluster. 

```yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: <public key>
```

Replace the `age:` value with your actual public key.

# Updating Flux Kustomization

In your flux (Flux Kustomization file), add:

```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

This tells Flux to use SOPS for decryption, leveraging the private key stored in the `sops-age` secret.


# Uploading to Git

Now that everything is set up, commit and push the encrypted secret and configuration files to your Git repository:

```
git add test-secret.yaml .sops.yaml

git commit -m "Added encrypted secret and SOPS configuration"

git push origin main
```

Once pushed, Flux will detect the changes and apply the decrypted secrets to the cluster automatically.


Using SOPS with Age provides a simple yet powerful way to manage secrets in a GitOps workflow. This setup ensures secrets remain encrypted in Git while being seamlessly decrypted within the Kubernetes cluster. 
