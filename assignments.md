# Trust Anchor Identifier Assignments

To facilitate early experiments with Trust Anchor Identifiers, the following table tracks the list of allocated trust anchor IDs. This table tracks the current OID-based allocation scheme. The final standard may ultimately use a different mechanism.

## Steps to assign new IDs:
1. If needed, obtain a Private Enterprise Number OID from IANA ([Request Form](https://www.iana.org/assignments/enterprise-numbers/assignment/apply/))
2. For each participating trust anchor, identified by their (subjectName, public key) tuple, assign a unique OID under the PEN.
3. Submit a pull request that adds a new row to the below table for each trust anchor ID assignment.

### Notes:
* Trust Anchor IDs should use the ASCII (dotted decimal) notation, e.g. `32473.1`.
* Public Keys should be PEM-encoded and include the `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` blocks. 
* The following command can be used to easily extract public keys in the correct format:
    * `$ openssl x509 -in certificate.pem -noout -pubkey`
* The following command can be used to easily extract subjectNames in the correct format:
    * `$ openssl x509 -in certificate.pem -noout -subject | sed -e "s/^subject= //"`

## List of assigned IDs
|Trust Anchor ID|Subject Name|Public Key|
|---------------|------------|----------|
||||
