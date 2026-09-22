# agnostic-signature

A TypeScript prototype for sending signed, encrypted payloads to a backend. The payload doesn't say which service it is meant for; the backend works that out by trying each service's key.

```mermaid
flowchart LR
    A[Client signs data with its private key] --> B[Encrypts with the chosen service's public key]
    B --> C[Opaque payload, no service identifier]
    C --> D[Backend tries each service's private key]
    D --> E[Verifies signature with client's public key]
```

When one backend handles credentials for several services (in the example, three banks), the client has to say which service a message is for. The usual approach puts that information outside the ciphertext: a key ID in the envelope, a header, or a separate endpoint for each service. The data is encrypted, but anyone who can see the traffic still learns which service the user is working with.

Standard encryption envelopes can't prevent this. To pick the right key, the receiver needs a label it can read, so the label travels in the clear. This project tries the other way around: send no label, and let the receiver find the key by trial.

## Example

Taken from `index.ts`:

```ts
const keys = await generateServicesKeys(['bancolombia', 'bbva', 'nequi']);
const clientWallet = await generateRSAKeyPair();

const signature = signData(Buffer.from(JSON.stringify(message.data)), clientWallet.privateKey);
const data = rsaEncrypt(signature, keys['nequi'].publicKey);

const decryptedData = findCorrespondingServiceKey(keys, data);
const verified = verifySignature(decryptedData.key, message.publicKey);
```

Run it with `npm start`.

## How it works

The client signs its data with SHA-256 and RSA, then puts the 256-byte signature in front of the data. The signed buffer is encrypted with a random AES-256-CBC key, and that AES key is wrapped with the service's RSA public key using OAEP. Before sending, the client removes the first byte of the wrapped key. The receiver loops over every service's private key and, for each one, tries all 256 possible values of the missing byte. The first combination that passes OAEP decoding identifies the service. The backend then checks the signature against the public key the client sent alongside the payload.

## What it doesn't do

This is a demo script, not a library. Service keys live in memory and are regenerated on every run. Nothing covers key storage, rotation, transport, or replay protection. Key and signature sizes are hardcoded for 2048-bit RSA.

A valid signature only proves the sender holds the private key matching the public key they sent. It says nothing about who that sender is. Removing one byte hides nothing cryptographically; it only forces the trial loop. In the worst case that loop runs 256 RSA decryptions per registered service, so cost grows linearly with the number of services. Don't use this where you need authenticated identities, high throughput, or a reviewed protocol.
