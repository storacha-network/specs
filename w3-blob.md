# W3 Blob Protocol

![status:wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)

- [Irakli Gozalishvili](https://github.com/gozala)

## Authors

- [Irakli Gozalishvili](https://github.com/gozala)

## Abstract

W3 blob protocol allows authorized agents to store arbitrary content blobs with a storage provider.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction

W3 blob protocol provides core building block for storing content and sharing access to it through UCAN authorization system. It is successor to the [store protocol] which no longer requires use of [Content Archive][CAR]s even if in practice clients can continue to use it for storing shards of large DAGs.

## Concepts

### Roles

There are several distinct roles that [principal]s may assume in this specification:

| Name        | Description                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Principal | The general class of entities that interact with a UCAN. Identified by a DID that can be used in the `iss` or `aud` field of a UCAN. |
| Agent       | A [Principal] identified by [`did:key`] identifier, representing a user in an application. |
| Issuer | A [principal] delegating capabilities to another [principal]. It is the signer of the [UCAN]. Specified in the `iss` field of a UCAN. |
| Audience | Principal access is shared with. Specified in the `aud` field of a UCAN. |

### Space

A namespace, often referred as a "space", is an owned resource that can be shared. It corresponds to a unique asymmetric cryptographic keypair and is identified by a [`did:key`] URI.

### Blob

Blob is a fixed size byte array addressed by the [multihash]. Usually blobs are used to represent set of IPLD blocks at different byte ranges.

# Capabilities

## Add Blob

Authorized agent MAY invoke `/space/content/add/blob` capability on the [space] subject to store specific byte array.

> Note that storing a blob does not imply advertising it on the network or making it publicly available.

### Add Blob Diagram

Following diagram illustrates execution workflow.

```mermaid
flowchart TB
Add("▶️ /space/content/add/blob 👩‍💻 🤖")

Added("🧾 { ok: { claim 🚦 } }")

Allocate("⏭️ /service/blob/allocate 🤖")

Accept("⏭️ /service/blob/accept 🤖")

Accepted("🧾 { ok: { claim 🎫 } }")


Claim("🎫 /assert/location 🤖👩‍💻")

Add --o Added
Added -.-> Allocate
Added -.-> Accept
Allocate --> Accept
Accept --o Accepted
Accepted -- claim --> Claim
Added -- claim --> Accept
```

#### Iconography

- Icon on the left describes type of the node
- Icon on the right describes issuer / audience pair if only one audience is the issuer
- ▶️ Task
- ⏭️ Next Task (a.k.a Effect)
- 🧾 Receipt
- 🎫 Delegation / Commitment
- 🚦 Await
- 👩‍💻 Alice
- 🤖 Service

### Add Blob Invocation Example

Shown Invocation example illustrates Alice requesting to add 2MiB blob to her space.

```js
{
  "cmd": "/space/content/add/blob",
  "sub": "did:key:zAlice",
  "iss": "did:key:zAlice",
  "aud": "did:web:web3.storage",
  "args": {
    "blob": {
      // multihash of the blob as byte array
      "content": { "/": { "bytes": "mEi...sfKg" } },
      // size of the blob in bytes
      "size": 2_097_152,
    }
  }
}
```

### Add Blob Receipt Example

Shows an example receipt for the above `/space/content/add/blob` capability invocation.

> ℹ️ We use `// "/": "bafy..` comments to denote CID of the parent object.

```js
{ // "/": "bafy..add",
  "iss": "did:web:web3.storage",
  "aud": "did:key:zAlice",
  "cmd": "/ucan/assert/result"
  "sub": "did:web:web3.storage",
  "args": {
    // refers to the invocation from the example
    "ran": { "/": "bafy..add" },
    "out": {
      "ok": {
        // result of the add is the content (location) claim
        // that is produced as result of "bafy..accept"
        "claim": { "await/ok": { "/": "bafy...accept" } }
      }
    },
    // Previously `next` was known as `fx` instead, which is
    // set of tasks to be scheduled.
    "next": [
      // 1. System attempts to allocate memory in user space for the blob.
      { // "/": "bafy...alloc",
        "cmd": "/service/blob/allocate",
        "sub": "did:web:web3.storage",
        "args": {
          // space where memory is allocated
          "space": "did:key:zAlice",
          "blob": {
            // multihash of the blob as byte array
            "content": { "/": { "bytes": "mEi...sfKg" } },
            // size of the blob in bytes
            "size": 2_097_152,
          }
        }
      },
      // 2. System will attempt to accept received content
      // if matches blob multihash and size.
      { // "/": "bafy...accept",
        "cmd": "/service/blob/accept",
        "sub": "did:web:web3.storage",
        "args": {
          "space": "did:key:zAlice",
          "blob": {
            // multihash of the blob as byte array
            "content": { "/": { "bytes": "mEi...sfKg" } },
            "size": 2_097_152,
          },
          exp: 1711122994101,
          // This task is blocked on allocation
          _: { "await/ok": { "/": "bafy...alloc" } }
        }
      }
    ]
  }
}
```

### Add Blob Capability

#### Add Blob Capability Schema

```ts
type AddBlob = {
  cmd: "/space/content/add/blob"
  sub: SpaceDID
  args: {
    blob: Blob
  }
}

type Blob = {
  content: Multihash
  size:    int
}

type Multihash = bytes
type SpaceDID = string
```

#### Blob Content

The `args.blob.content` field MUST be a [multihash] digest of the blob payload bytes. Implementation SHOULD support SHA2-256 algorithm. Implementation MAY in addition support other hashing algorithms.

#### Blob Size

Blob `args.blob.size` field MUST be set to the number of bytes in the blob content.

### Add Blob Receipt

#### Add Blob Receipt Schema

```ts
// Only operation specific fields are covered the
// rest are implied
type AddBlobReceipt = {
  out: Result<AddBlobOk, AddBlobError>
  next: [
    AllocateBlob,
    AcceptBlob,
  ]
}

type AddBlobOk = {
  claim: {
    "await/ok": Link<AcceptBlob>
  }
}

type AddBlobError = {
  message: string
}
```

#### Add Blob Result

Invocation MUST fail if any of the following is true

1. Provided **sub**ject space is not provisioned with a provider.
1. Provided `blob.size` is outside of supported range.
1. Provided `blob.content` is not a valid [multihash].
1. Provided `blob.content` [multihash] hashing algorithm is not supported.

Invocation MUST succeed if non of the above is true. Success value MUST be an object with a `claim` field set to [await/ok] of the task that produces [location claim].

Task linked from the `claim` of the success value MUST be present in the receipt effects _(`next` field)_.

#### Add Blob Effects

Successful invocation MUST start a workflow consisting of following tasks, that MUST be set in receipt effects (`next` field) in the following order.

1. [Allocate Blob]
1. [Accept Blob]

## Allocate Blob

Authorized agent MAY invoke `/service/blob/allocate` capability on the [provider] subject to create a memory address where `blob` content can be written via HTTP `PUT` request.

### Allocate Blob Capability

#### Allocate Blob Capability Schema

```ts
type BlobAllocate = {
  cmd:  "/service/blob/allocate"
  sub:  ProviderDID
  args: {
    space:  SpaceDID
    blob:   Blob
    cause:  Link<AddBlob>
  }
}
```

#### Allocation Space

The `args.space` field MUST be set to the [DID] of the user space where allocation takes place.

#### Allocation Blob

The `args.blob` field MUST be set to the `Blob` the space is allocated for.

#### Allocation Cause

The `args.cause` field MUST be set to the [Link] for an [Add Blob] task, that caused an allocation.

### Allocate Blob Receipt

Allocations MUST fail if `space` does not have enough capacity for the `blob` and succeed otherwise.

#### Allocate Blob Receipt Schema

```ts
type BlobAllocateReceipt = {
  ran:  Link<BlobAllocate>
  out:  Result<BlobAllocateOk, BlobAllocateError>
  next: []
}

type BlobAllocateOk = {
  size: int
  address?: BlobAddress
}

type BlobAddress = {
  url:     string
  headers: {[key:string]: string}
}
```

### Allocation Size

The `out.ok.size` MUST be set to the number of bytes that were allocated for the `Blob`. It MUST be equal to either:

1. The `args.blob.size` of the invocation.
2. `0` if space already has memory allocated for the `args.blob`.

### Allocation Address

The optional `out.ok.address` SHOULD be omitted when content for the allocated is already available on site. Otherwise it MUST be set to the `BlobAddress` that can receive a blob content.

The `url` of the `BlobAddress` MUST be an HTTP(S) location that can receive blob content via HTTP `PUT` request, as long as HTTP headers from `headers` dictionary are set on the request.

It is RECOMMENDED that issued `BlobAddress` only accept `PUT` payload that matches requested `blob`, both content [multihash] and size.

> ℹ️ If enforcing above recommendation is not possible implementation MUST enforce in [Accept Blob] invocation.

### Allocation Effects

Allocation MUST have no effects.

## Accept Blob

Authorized agent MAY invoke `/service/blob/accept` capability on the [provider] subject. Invocation MUST either succeed when content is delivered on allocated `address` or fail if no content is allocation expires without content being delivered.

ℹ️ Implementation that is unable to enforce to reject HTTP PUT request that do not match blob [multihash] or `size` SHOULD enforce that invariant in this invocation by failing task if no valid content has been delivered.

### Accept Blob Capability

#### Accept Blob Capability Schema

```ts
type BlobAccept = {
  cmd: "/service/blob/accept"
  sub: ProviderDID
  args: {
    blob: Blob
    exp: int
  }
}
```

### Accept Blob Receipt

#### Accept Blob Receipt Schema

```ts
type BlobAcceptReceipt = {
  ran: Link<BlobAccept>
  out: Result<BlobAcceptOk, BlobAcceptError>
  next: []
}

type BlobAcceptOk = {
  claim: Link<LocationClaim>
}

type BlobAcceptError = {
  message: string
}
```

#### Blob Accept Effects

Receipt MUST not have any effects.

## Location Claim

Location claim represents commitment from the issuer to the audience that
content matching the `content` [multihash] can be read via HTTP [range request]

### Location Claim Capability

#### Location Claim Capability Schema

```ts
type LocationClaim = {
  cmd: "assert/location"
  sub: ProviderDID
  args: {
    content: Multihash
    url: string
    range: [start:int, end:int, ...int[]]
  }
}
```

# Coordination

## Accept Content

[Accept Blob] invocation will block until content is delivered, however some implementations may not be able to observe when content was received. Those implementations can await for subsequent [Add Blob] invocations and re-check whether content has been received.

Note that implementation MUST be idempotent and same receipts MUST be returned to the caller, yet pending tasks could be updated.

## Publishing Blob

Blob can be published by authorizing read interface (e.g. IPFS gateway) by delegating it [Location Claim] that has been obtained from the provider.

> Note that same applies to publishing blob on [IPNI], new capability is not necessary, user simply needs to re-delegate `LocationClaim` to the DID representing [IPNI] publisher. [IPNI] publisher in turn may publish delegation to DID with publicly known private key allowing anyone to perform the reads.

[store protocol]:./w3-store.md
[CAR]:https://ipld.io/specs/transport/car/
[multihash]:https://github.com/multiformats/multihash
[space]:#space
[IPNI]:https://github.com/ipni/specs/blob/main/IPNI.md
[await/ok]:https://github.com/ucan-wg/invocation?tab=readme-ov-file#await
[location claim]:#location-claim
[Add Blob]:#add-blob
[Allocate Blob]:#allocate-blob
[Accept Blob]:#accept-blob
[DID]:https://www.w3.org/TR/did-core/
[Link]:https://ipld.io/docs/schemas/features/links/
[range request]:https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests
