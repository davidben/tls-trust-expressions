# Trust Anchor Identifier Assignments
The following table tracks the list of OIDs assigned as Trust Anchor Identifiers.

## Steps to assign new TAIs:
1. Obtain a Private Enterprise Number OID from IANA ([Request Form](https://www.iana.org/assignments/enterprise-numbers/assignment/apply/))
2. For each participating trust anchor, identified by their (subjectName, public key) tuple, assign a unique OID under the PEN.
3. Submit a pull request that adds a new row to the below table for each TAI assignment.

### Notes:
* TAI OIDs should be represented in dot decimal notation.
* Public Keys should be PEM-encoded and include the `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` blocks. 
* The following command can be used to easily extract public keys in the correct format:
    * `$ openssl x509 -in certificate.pem -noout -pubkey`
* The following command can be used to easily extract subjectNames in the correct format:
    * `$ openssl x509 -in gts-root-4.crt -noout -subject | sed -e "s/^subject= //"`

## List of assigned TAIs
|TAI OID|SubjectName|Public Key|
|-------|-----------|----------|
||||
