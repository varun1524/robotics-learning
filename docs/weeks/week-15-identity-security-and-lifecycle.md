# Week 15 — Identity, Security, and Lifecycle

[← Week 14: Calibration and navigation tuning](week-14-calibration-and-navigation-tuning.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 16: Capstone cloud-edge inspection →](week-16-capstone-cloud-edge-inspection.md)

**Estimated effort:** 8–10 hours

**Lab mode:** localhost PKI and broker; physical robot motion remains disabled

## Outcomes

By the end of this week you will be able to:

- separate bootstrap authorization from long-lived operational identity;
- generate the robot’s private key locally and enroll only its CSR;
- bind an mTLS connection to one stable robot identity and least-privilege topic scope;
- implement retry-safe enrollment, certificate renewal, suspension, revocation, recovery, and retirement;
- reject payload-supplied identity when it disagrees with authenticated context; and
- create audit evidence without logging grants, private keys, tokens, or arbitrary payload content.

## Prerequisites

- Week 14 complete, with a stable `robot-lab-01` identity in your learning registry.
- Docker, OpenSSL 3.x, Python 3.11+, `jq`, `sqlite3`, and MQTT clients.
- Motion disabled or the robot on blocks. This week tests trust and lifecycle, not movement.
- Basic understanding of public/private keys, hashes, TLS, and file permissions.

## Public readings

1. [OpenSSL certificate authority command](https://docs.openssl.org/3.0/man1/openssl-ca/)
2. [OpenSSL certificate request command](https://docs.openssl.org/3.0/man1/openssl-req/)
3. [Mosquitto TLS configuration](https://mosquitto.org/man/mosquitto-conf-5.html)
4. [MQTT 5 specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
5. [NISTIR 8259A: IoT device cybersecurity capability core baseline](https://csrc.nist.gov/pubs/ir/8259/a/final)
6. [OWASP IoT Security Testing Guide](https://owasp.org/owasp-istg/)

## Concepts

| Concept | Meaning in this lab |
| --- | --- |
| Registration profile | Admission policy that limits identity, model, count, lifetime, and initial-trust method. |
| Bootstrap grant | Short-lived bearer secret that authorizes one enrollment attempt; it is not a runtime identity. |
| CSR | Certificate signing request containing a public key and proof that the requester has the corresponding private key. |
| Operational certificate | Short-lived client certificate uniquely bound to one robot and used for normal connections. |
| mTLS | TLS in which both server and client authenticate with certificates. |
| Authorization context | Robot identity and permitted scope derived from the authenticated certificate and registry. |
| Renewal | Planned replacement while the current operational identity is still valid. |
| Recovery | Operator-authorized restoration when ordinary renewal is impossible. |
| Revocation | Invalidation of a certificate before its natural expiry. |
| Retirement | Terminal lifecycle state; the identity and audit history remain, but runtime operations are prohibited. |

The trust chain is deliberately narrow:

```text
bootstrap grant -> may attempt enrollment
device claim    -> claimed model/serial, not cryptographic chassis proof
robot CSR       -> proof of possession of a new robot-owned key
issued cert     -> operational identity for exactly one robot
registry        -> lifecycle, authorization version, and allowed scope
```

## Environment and packages

```bash
sudo apt update
sudo apt install -y openssl mosquitto-clients jq sqlite3
mkdir -p ~/robotics-lab/week15/{ca,broker,robot,cloud,registry,evidence}
cd ~/robotics-lab/week15
umask 077
openssl version | tee evidence/versions.txt
mosquitto_pub --help 2>&1 | head -1 | tee -a evidence/versions.txt
docker version --format '{{.Server.Version}}' | tee -a evidence/versions.txt
```

Never place this lab’s `ca/private`, robot private keys, plaintext grants, or active tokens in source control. Verify your ignore rules before continuing.

## Lab: enroll and operate one robot with mTLS

### 1. Create the durable resource and lifecycle record

Create `registry/robot-lab-01.json`:

```json
{
  "robotId": "robot-lab-01",
  "displayName": "Learning Robot",
  "deviceIdentity": {
    "manufacturer": "lab",
    "model": "mobile-base",
    "serialNumber": "LAB-0001"
  },
  "lifecycle": "ENROLLED",
  "authorizationVersion": 1,
  "certificateGeneration": 0,
  "enabledCapabilities": ["observe_state.v1", "halt.v1"],
  "timeCreated": "<RFC3339 timestamp>"
}
```

The stable robot record exists before operational credentials. Treat the physical identity tuple as immutable. A copied serial number is not hardware attestation; document that residual risk.

Initialize append-only audit storage:

```bash
touch registry/audit.jsonl
chmod 600 registry/audit.jsonl
```

Every audit record must contain event ID, event type, actor, robot ID, certificate serial or digest when applicable, outcome, reason, request/correlation ID, and timestamp. Do not include private keys, plaintext grants, or arbitrary command payloads.

### 2. Build a local certificate authority

Create the CA database:

```bash
cd ~/robotics-lab/week15/ca
mkdir -p certs crl newcerts private
chmod 700 private
touch index.txt
printf '1000\n' > serial
printf '1000\n' > crlnumber
```

Create `ca/openssl.cnf`:

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = .
database          = $dir/index.txt
new_certs_dir     = $dir/newcerts
certificate       = $dir/certs/ca.crt
private_key       = $dir/private/ca.key
serial            = $dir/serial
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl
default_md        = sha256
default_days      = 7
default_crl_days  = 1
policy            = policy_loose
unique_subject    = no
copy_extensions   = none

[ policy_loose ]
commonName = supplied

[ req ]
distinguished_name = req_dn
prompt = no
x509_extensions = ca_ext

[ req_dn ]
CN = Robotics Learning Lab CA

[ ca_ext ]
basicConstraints = critical,CA:true,pathlen:0
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always

[ server_cert ]
basicConstraints = critical,CA:false
keyUsage = critical,digitalSignature,keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:localhost,IP:127.0.0.1
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer

[ client_cert ]
basicConstraints = critical,CA:false
keyUsage = critical,digitalSignature
extendedKeyUsage = clientAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
```

Generate the CA key and certificate:

```bash
openssl genpkey -algorithm EC \
  -pkeyopt ec_paramgen_curve:P-256 \
  -out private/ca.key
openssl req -new -x509 -days 3650 -sha256 \
  -config openssl.cnf \
  -key private/ca.key \
  -out certs/ca.crt
openssl x509 -in certs/ca.crt -noout -subject -fingerprint -sha256 \
  | tee ../evidence/ca-identity.txt
```

For a production system, an offline or managed CA, protected signing keys, formal policy, rotation, and independent review are required. This file-backed CA is only a local exercise.

### 3. Issue the broker’s server identity

```bash
cd ~/robotics-lab/week15
openssl genpkey -algorithm EC \
  -pkeyopt ec_paramgen_curve:P-256 \
  -out broker/server.key
openssl req -new -sha256 \
  -key broker/server.key \
  -subj '/CN=localhost' \
  -out broker/server.csr
cd ca
openssl ca -batch -config openssl.cnf \
  -extensions server_cert \
  -in ../broker/server.csr \
  -out ../broker/server.crt
openssl ca -config openssl.cnf -gencrl -out crl/ca.crl
```

Verify hostname, chain, and purpose:

```bash
openssl verify -CAfile ca/certs/ca.crt -purpose sslserver broker/server.crt
openssl x509 -in broker/server.crt -noout -text \
  | grep -A2 'Subject Alternative Name'
```

### 4. Issue a one-use bootstrap grant

Create a short-lived grant without shell tracing:

```bash
set +x
openssl rand -hex 32 > registry/bootstrap-grant.secret
openssl dgst -sha256 -r registry/bootstrap-grant.secret \
  | awk '{print $1}' > registry/bootstrap-grant.digest
chmod 600 registry/bootstrap-grant.secret registry/bootstrap-grant.digest
```

Add profile metadata containing only the digest, expected robot ID/identity, issued time, expiry, and `ACTIVE` status. The plaintext grant is returned once through a secure operator channel and deleted after successful enrollment.

Retry safety rule:

- same retry token plus the same normalized CSR/identity returns the original result;
- the same retry token with different content returns a conflict;
- a consumed or expired grant cannot enroll a second key.

### 5. Generate the robot key and CSR on the robot side

The private key is generated in `robot/` and never copied into `ca/`:

```bash
openssl genpkey -algorithm EC \
  -pkeyopt ec_paramgen_curve:P-256 \
  -out robot/operational-g1.key
chmod 600 robot/operational-g1.key
openssl req -new -sha256 \
  -key robot/operational-g1.key \
  -subj '/CN=robot-lab-01' \
  -out robot/operational-g1.csr
openssl req -in robot/operational-g1.csr -noout -verify -subject
```

Create `robot/robot-lab-01.ext`:

```ini
[ client_cert ]
basicConstraints = critical,CA:false
keyUsage = critical,digitalSignature
extendedKeyUsage = clientAuth
subjectAltName = URI:urn:robot:robot-lab-01
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
```

The enrollment service, represented manually here, must verify:

1. grant hash, status, and expiry;
2. exact expected `robotId` and device claim;
3. lifecycle is `ENROLLED`;
4. CSR signature and allowed algorithm;
5. public key is not already bound to another robot; and
6. idempotency token/fingerprint.

Only after all pass, sign the CSR:

```bash
cd ~/robotics-lab/week15/ca
openssl ca -batch -config openssl.cnf \
  -extfile ../robot/robot-lab-01.ext \
  -extensions client_cert \
  -in ../robot/operational-g1.csr \
  -out ../robot/operational-g1.crt
```

Verify the identity:

```bash
openssl verify -CAfile ca/certs/ca.crt -purpose sslclient \
  robot/operational-g1.crt
openssl x509 -in robot/operational-g1.crt -noout \
  -subject -serial -dates -fingerprint -sha256
openssl x509 -in robot/operational-g1.crt -noout -text \
  | grep -A2 'Subject Alternative Name'
```

Update the registry atomically to `REGISTERED`, generation `1`, certificate serial/fingerprint/not-after, and authorization version `2`. Mark the grant `CONSUMED`, append an audit event, and securely remove the plaintext grant:

```bash
rm registry/bootstrap-grant.secret
```

### 6. Create the cloud client identity

Generate a separate client certificate with common name `cloud-service` and URI SAN `urn:service:cloud-command`. Do not reuse the broker key, CA key, or robot key.

Repeat the CSR/signing flow using `cloud/cloud.key`, `cloud/cloud.csr`, `cloud/cloud.crt`, and a service-specific extension file.

### 7. Configure least-privilege mTLS authorization

Create `broker/acl`:

```text
user robot-lab-01
topic read robots/robot-lab-01/cloud-to-robot
topic write robots/robot-lab-01/robot-to-cloud

user cloud-service
topic write robots/robot-lab-01/cloud-to-robot
topic read robots/robot-lab-01/robot-to-cloud
```

Create `broker/mosquitto.conf`:

```text
persistence false
allow_anonymous false
listener 8883
cafile /mosquitto/config/ca.crt
certfile /mosquitto/config/server.crt
keyfile /mosquitto/config/server.key
crlfile /mosquitto/config/ca.crl
require_certificate true
use_identity_as_username true
acl_file /mosquitto/config/acl
log_type all
```

Copy only public trust material and the broker key/cert into the mounted configuration directory, then start the broker:

```bash
cp ca/certs/ca.crt broker/ca.crt
cp ca/crl/ca.crl broker/ca.crl
docker run --rm --name week15-broker -p 8883:8883 \
  -v "$HOME/robotics-lab/week15/broker:/mosquitto/config:ro" \
  eclipse-mosquitto:2
```

The CA private key is not mounted into the broker.

### 8. Prove authenticated routing

In one terminal, subscribe as the robot:

```bash
mosquitto_sub -h localhost -p 8883 \
  --cafile ca/certs/ca.crt \
  --cert robot/operational-g1.crt \
  --key robot/operational-g1.key \
  -t robots/robot-lab-01/cloud-to-robot -d
```

Publish as the cloud service:

```bash
mosquitto_pub -h localhost -p 8883 \
  --cafile ca/certs/ca.crt \
  --cert cloud/cloud.crt --key cloud/cloud.key \
  -t robots/robot-lab-01/cloud-to-robot \
  -m '{"messageId":"identity-test-01","type":"observe_state.v1"}' -d
```

Negative tests:

```bash
# Robot must not publish on the cloud-to-robot topic.
mosquitto_pub -h localhost -p 8883 \
  --cafile ca/certs/ca.crt \
  --cert robot/operational-g1.crt \
  --key robot/operational-g1.key \
  -t robots/robot-lab-01/cloud-to-robot -m forbidden -d

# Robot must not access another robot's topic.
mosquitto_sub -h localhost -p 8883 \
  --cafile ca/certs/ca.crt \
  --cert robot/operational-g1.crt \
  --key robot/operational-g1.key \
  -t robots/robot-lab-02/cloud-to-robot -d
```

Pass only if both are denied. A payload containing another `robotId` must also be rejected by the application if it conflicts with the certificate-derived identity.

### 9. Renew with a new key and bounded overlap

Before generation 1 expires, generate `operational-g2.key` and a new CSR. Authenticate the renewal request with generation 1, validate lifecycle and current generation, then sign generation 2.

During a predeclared five-minute lab overlap, both generations may authenticate. Record first successful use of generation 2, then revoke generation 1. Never use the bootstrap grant for renewal.

### 10. Suspend and revoke generation 1

Update robot lifecycle to `SUSPENDED`, increment `authorizationVersion`, and append an audit record. Revoke the old certificate and generate a CRL:

```bash
cd ~/robotics-lab/week15/ca
openssl ca -config openssl.cnf \
  -revoke ../robot/operational-g1.crt \
  -crl_reason superseded
openssl ca -config openssl.cnf -gencrl -out crl/ca.crl
cp crl/ca.crl ../broker/ca.crl
```

Restart the local broker so it reloads the CRL. Generation 1 must fail a new connection; generation 2 must also be denied by the application while the Robot resource is suspended, even if its TLS certificate is cryptographically valid. TLS validity and resource authorization are separate checks.

### 11. Recover the same robot identity

Issue a random, short-lived, one-use recovery grant bound to:

- `robot-lab-01`;
- its immutable device identity;
- suspended lifecycle state;
- expiry; and
- a new retry token.

Generate a third key/CSR on the robot. Verify recovery proof, sign generation 3, consume the recovery grant, transition the same Robot resource back to `ACTIVE`, and increment authorization version. Recovery must not create `robot-lab-02`, alter the immutable identity, or reuse registration capacity.

### 12. Retire the identity

Transition the resource to `RETIRED`, revoke all active generations, deny new sessions, and retain public certificate metadata plus audit history. Retirement is terminal in this lab; editing JSON back to `ACTIVE` is not a valid recovery path.

## Deliberate failure injection

Run and document these tests:

1. **Grant replay:** reuse the consumed bootstrap grant with a different CSR. Expect rejection and no new certificate.
2. **Retry-token conflict:** use the enrollment retry token with different normalized identity content. Expect a conflict, not a second robot.
3. **Wrong scope:** use the valid robot certificate on another robot’s topic. Expect broker denial.
4. **Payload spoofing:** send `robotId=robot-lab-02` over the authenticated `robot-lab-01` connection. Expect application rejection and audit.
5. **Revoked certificate:** reconnect with generation 1 after CRL update. Expect TLS rejection.
6. **Suspended resource:** use a cryptographically valid generation while the registry says `SUSPENDED`. Expect application authorization denial.
7. **Expired recovery grant:** wait beyond expiry, then attempt recovery. Expect rejection without resource mutation.
8. **Second live connection:** connect twice with the same robot identity and different client IDs. Your gateway/session layer must retain one authoritative connection and reject or fence the other deterministically.

## Assignment

Automate the manual enrollment workflow as a small service with `create-profile`, `issue-grant`, `enroll`, `renew`, `suspend`, `revoke`, `issue-recovery`, `recover`, and `retire` operations. Store the registry and idempotency records in SQLite. Store no plaintext grant or private key. Add integration tests for every lifecycle transition and negative case above.

## Measurements

| Measurement | Result |
| --- | ---: |
| Enrollment latency, median/p95 over 20 runs | |
| mTLS connection latency, median/p95 | |
| Revocation propagation time | |
| Renewal overlap duration | |
| Audit append latency | |
| Duplicate enrollment count | Must be 0 |
| Unauthorized cross-topic successes | Must be 0 |
| Secrets found by repository scan | Must be 0 |

Scan the learning workspace before completion:

```bash
rg -n --hidden \
  'BEGIN (EC |RSA )?PRIVATE KEY|bootstrap-grant\.secret|labadmin-change-me' \
  ~/robotics-learning ~/robot_ws || true
```

Interpret matches; do not merely suppress them.

## Evidence and deliverables

- public CA and certificate metadata; never deliver private keys
- redacted registration profile and Robot lifecycle records
- CSR verification and certificate-chain verification output
- broker configuration and least-privilege ACL
- successful allowed publish/subscribe transcript
- negative authorization, replay, suspension, revocation, recovery, and retirement results
- CRL metadata and measured revocation propagation
- append-only, redacted audit log
- automated lifecycle/idempotency tests and secrets-scan report
- threat note explaining why a shared bootstrap grant does not prove chassis authenticity

## Objective exit criteria

- [ ] The operational private key was generated robot-side and never entered CA, broker, or application storage.
- [ ] Bootstrap grant authorizes enrollment only and is stored server-side only as a verifier/digest.
- [ ] Consumed, expired, or conflicting grant use is rejected.
- [ ] The operational certificate binds exactly one stable robot identity.
- [ ] Server, robot, and cloud client use separate keys and extended-key usages.
- [ ] Broker ACL permits only the exact two directional topics for the robot.
- [ ] Authenticated payload identity mismatches are rejected.
- [ ] Enrollment and retry behavior never creates a duplicate Robot identity.
- [ ] Renewal creates a new key and generation with bounded overlap.
- [ ] Revocation and suspended lifecycle both block new operational sessions.
- [ ] Recovery preserves the same Robot resource and immutable device identity.
- [ ] Retirement is terminal and preserves audit evidence.
- [ ] Logs, repository, and delivered evidence contain no plaintext grants, private keys, or live tokens.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `certificate verify failed` | CA file, validity dates, system clock, key usage, SAN, and complete chain. |
| Broker reports unknown user | Client certificate common name and `use_identity_as_username`; do not confuse SAN with username mapping. |
| ACL appears ignored | Confirm anonymous access is false, ACL path is mounted, and broker reloaded configuration. |
| Revoked certificate still connects | Regenerate/copy the CRL and restart or reload the broker; measure propagation explicitly. |
| New certificate has the old public key | Confirm a new key generated the CSR; compare SHA-256 public-key fingerprints. |
| Same grant creates two certificates | Make grant consumption, idempotency reservation, and certificate state one atomic transaction. |
| Valid certificate works while suspended | Add registry authorization after TLS authentication and fence active sessions on authorization-version change. |
| Secrets appear in logs | Disable shell tracing, redact headers/bodies, log digests/IDs only, rotate exposed credentials, and repeat the scan. |

## Next step

Week 16 combines the canonical adapter, durable commands, pinned maps, offline mission execution, telemetry replay, WebRTC, evidence upload, identity, and audit into one objective capstone acceptance test.
