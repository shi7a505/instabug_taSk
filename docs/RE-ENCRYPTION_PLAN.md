# Plan of Action for Re-encrypting SealedSecrets in Kubernetes

This document outlines a comprehensive plan for implementing the re-encryption mechanism for all SealedSecrets in a Kubernetes cluster, following the requirements and bonus features mentioned.

## 1. Identifying All Existing SealedSecrets in the Kubernetes Cluster

To implement a re-encryption mechanism, the first step is to gather all existing SealedSecrets across all namespaces within the Kubernetes cluster.

#### Steps:
1. Use `kubectl` to list all the SealedSecrets in the cluster:

```bash
kubectl get sealedsecrets --all-namespaces -o yaml > all-sealedsecrets.yaml
```
2. This YAML file will contain all the SealedSecrets across all namespaces. The next step will be to process each of these secrets individually for decryption and re-encryption.

#### Considerations:
- Ensure the kubectl user has sufficient permissions to access all namespaces and secrets.
- Make sure to handle namespaces with a large number of SealedSecrets efficiently by batching them, as detailed in the bonus section.

## 2. Fetching All Active Public Keys of the SealedSecrets Controller

To decrypt and later re-encrypt the SealedSecrets, you need access to the public keys used by the SealedSecrets controller. These public keys are essential for the encryption process.

#### Steps:
1. Fetch the latest public key used by the SealedSecrets controller using the following command:

```bash
kubeseal --fetch-cert > sealedsecrets-public-cert.pem
```
2. The `sealedsecrets-public-cert.pem` file will contain the public key that should be used for re-encryption.

#### Considerations:
- Ensure that the latest key is being used for the re-encryption process, as keys are rotated periodically (e.g., every 30 days).
- Maintain the public key securely during this process to avoid accidental exposure.

## 3. Decrypting Each SealedSecret Using Existing Private Keys

To re-encrypt the secrets, you must first decrypt them using the private key stored securely within the Kubernetes cluster.

#### Steps:
1. **Obtain the Private Key:** The private key is securely stored in the Kubernetes cluster, typically as a Kubernetes Secret. Use kubectl to fetch the private key:

```bash
kubectl get secret sealed-secrets-key -n kube-system -o jsonpath='{.data.tls\.key}' | base64 --decode > private-key.pem
```
2. **Decrypt the SealedSecrets:** Once you have the private key, use the `kubeseal` tool to decrypt each SealedSecret (Note: `kubeseal` does not support decryption directly through the CLI, this is an internal process using the SealedSecrets controller):

```bash
kubeseal --decrypt --cert private-key.pem -o yaml < sealed-secret.yaml > decrypted-secret.yaml
```
**Note:** Decrypting the SealedSecrets will be performed by the SealedSecrets controller using its internal API, not directly via `kubeseal` CLI, due to security considerations. The `kubeseal` tool is primarily used for encryption, and decryption can be managed internally within Kubernetes through the controller API.

#### Considerations:
- Ensure the private key remains secure and is not exposed to unauthorized users.
- Handle decryption errors, logging failed decryptions for auditing and troubleshooting.

## 4. Re-encrypting the Decrypted Secrets Using the Latest Public Key

Once you’ve decrypted the SealedSecrets, the next step is to re-encrypt them using the latest public key.

#### Steps:
1. **Re-encrypt with Public Key:** Use the public key fetched earlier to re-encrypt each decrypted secret:

```bash
kubeseal --cert sealedsecrets-public-cert.pem -o yaml < decrypted-secret.yaml > re-encrypted-sealed-secret.yaml
```
This command will generate a new SealedSecret object with the re-encrypted secret.

#### Considerations:
- Ensure that re-encryption is done correctly for each secret and the re-encrypted secrets are saved with the correct metadata and structure.
- Validate that the re-encrypted secrets work by applying them and confirming that the applications in the cluster can access them.

## 5. Updating the Existing SealedSecret Objects with the Re-encrypted Data

After successfully re-encrypting the secrets, the next step is to apply the updated SealedSecret objects to the Kubernetes cluster.

#### Steps:
1. **Apply the Re-encrypted SealedSecrets:** Use `kubectl apply` to update the SealedSecrets in the cluster:

```bash
kubectl apply -f re-encrypted-sealed-secret.yaml
```
This will update the existing SealedSecrets in the cluster with the newly re-encrypted data.

#### Considerations:
- Ensure that the new SealedSecrets are applied successfully without overriding any necessary configurations or causing disruptions.
- Apply changes in a controlled manner, possibly starting with a test namespace or application.

## 6. Logging or Reporting Mechanism to Track the Re-encryption Process and Errors (Bonus)

Having a logging mechanism will help track the status of the re-encryption process, making it easier to debug any issues.

#### Steps:
1. **Create a Log File:** Store logs in a file that records each step of the re-encryption process, including successes and errors:

```bash
echo "[INFO] Re-encrypting SealedSecret $SECRET_NAME in namespace $NAMESPACE" >> re-encryption.log
echo "[ERROR] Failed to decrypt SealedSecret $SECRET_NAME in namespace $NAMESPACE: $ERROR_MESSAGE" >> re-encryption-errors.log
```
2. **Log Details:** For each SealedSecret processed, log:
   - The name and namespace of the SealedSecret.
   - Whether the decryption and re-encryption were successful or if any errors occurred.

3. **Report Critical Errors:** Set up notifications (via Slack, email, etc.) to alert the team when critical errors occur (e.g., failure to decrypt a SealedSecret).

## 7. Consideration for Handling Large Numbers of SealedSecrets Efficiently (Bonus)

When dealing with large clusters, processing a large number of SealedSecrets efficiently is crucial.

#### Steps:
1. **Batch Processing:** Split the SealedSecrets into smaller chunks to avoid overloading the system:

```bash
split -l 50 all-sealedsecrets.yaml sealedsecrets_batch_
```
2. **Parallel Processing:** Use parallel processing tools like xargs or GNU Parallel to handle multiple SealedSecrets concurrently:

```bash
cat sealedsecrets_batch_1.yaml | xargs -n 1 -P 10 kubeseal --decrypt --cert private-key.pem -o yaml > re-encrypted-sealed-secrets.yaml
```
3. **Efficient Kubernetes API Usage:** Minimize calls to Kubernetes API by using direct client libraries like Go or Python, reducing overhead compared to kubectl.

## 8. Ensure Security of the Private Keys Throughout the Process (Bonus)

Security of the private key is paramount throughout the re-encryption process.

#### Steps:
1. **Store Private Key Securely:** Always ensure that the private key is stored securely as a Kubernetes Secret. It should never be exposed in plaintext or stored on disk unprotected.
2. **Use RBAC:** Limit access to the private key using Kubernetes RBAC policies, allowing only authorized services or users to access it.
3. **Audit Logs:** Enable Kubernetes audit logging to track any access to the private key and ensure it is not accessed by unauthorized parties.

## 9. Clear Documentation on How to Use the Re-encryption Feature

### SealedSecrets Re-encryption Feature Documentation

#### Overview
This document describes how to use the proposed SealedSecrets re-encryption feature integrated into the `kubeseal` CLI.  
The purpose of this feature is to automate the process of re-encrypting existing `SealedSecret` resources using the latest active public key generated by the SealedSecrets controller, especially after a key rotation event.

#### How It Works
The re-encryption process involves the following steps:
1. Identify all existing `SealedSecret` resources within the Kubernetes cluster.
2. For each `SealedSecret`, decrypt the data using the appropriate private key held by the controller.
3. Re-encrypt the decrypted data using the latest public key available.
4. Replace the old `SealedSecret` resource with a new one containing the updated encrypted data.
5. Optionally, log the results of the re-encryption operation for auditing and debugging.

#### Command Usage
The re-encryption functionality can be invoked using the following command:

```bash
kubeseal reencrypt [flags]
```

#### Flags

| Flag                | Description                                                                 |
|---------------------|-----------------------------------------------------------------------------|
| `--namespace`        | Re-encrypt SealedSecrets in the specified namespace only.                   |
| `--all-namespaces`   | Scan all namespaces for SealedSecrets.                                      |
| `--dry-run`          | Simulates the re-encryption process without applying any changes.           |
| `--log-file`         | Specifies the path to a log file to store information about the operation.  |
| `--concurrency`      | Number of concurrent workers used to process SealedSecrets (optional).      |

#### Example Usage
Re-encrypt all SealedSecrets in the cluster and store the process log:

```bash
kubeseal reencrypt --all-namespaces --log-file /tmp/reencrypt-log.txt
```

Dry-run to preview changes without applying them:

```bash
kubeseal reencrypt --namespace dev --dry-run
```

#### Security Considerations
- Private keys used for decryption remain inside the Kubernetes cluster and are never exposed externally.
- Communication with the controller is done over secure channels (HTTPS).
- The tool should be run using a Kubernetes service account with only the necessary permissions to read, decrypt, and update SealedSecrets.

#### Performance and Scalability
To efficiently handle a large number of SealedSecrets:
- Secrets are processed in batches to reduce memory usage.
- Concurrency options allow tuning for better performance.
- Optional rate-limiting can be applied to prevent API throttling.

#### Logging and Reporting
- The tool can generate a log file that includes:
  - Total number of secrets processed
  - Time taken per operation
  - Skipped resources 
  - Errors and warnings
- Logs help with troubleshooting and auditing the re-encryption process.

#### Verification and Testing
Before applying changes, users are encouraged to run the process in `--dry-run` mode.  
This ensures that the system behaves as expected and provides visibility into what changes would be made.

#### Help
For more information and a full list of supported options, run:

```bash
kubeseal reencrypt --help
```

#### Summary
This re-encryption feature enhances the security lifecycle of SealedSecrets by ensuring all secrets are encrypted with the most recent public key. It simplifies secret management in clusters where frequent key rotations are enforced and aligns with best practices for secret lifecycle automation.
