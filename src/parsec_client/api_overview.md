# API Overview

## Introduction

This document introduces the API contract that exists between the client and the service, covering
its general principles and organization. Use this document in combination with the [**wire protocol
specification**](wire_protocol.md) and the [**operation directory**](operations) as a reference
guide for the development of client libraries.

## Audience

This document details the API that is exposed directly by the service to its clients. This is the
API that is invoked over the IPC transport between the client process and the service. Client
applications do not consume this API directly. Instead, they consume a client library in their
chosen programming language. The client library will present an idiomatic and simplified view of the
API, making heavy use of argument defaulting to shield application developers from much of the
complexity of invoking cryptographic operations. Client libraries should present the API in a form
that is "easy to use and hard to get wrong". The audience for this document is the client library
developer, *not* the application developer. Application developers should consult the client library
documentation package instead.

## Opcodes and Contracts

The API is expressed as a set of individual and distinct **operations**. Each operation has a unique
numerical **opcode** to set it apart from other operations, and to allow for it to be unambiguously
selected for in a client request. (See the [**wire protocol specification**](wire_protocol.md) for
information about how the client must structure such requests). Each operation also has an **input
contract**, which defines the arguments that the client must supply when invoking it. It also has an
**output contract**, which defines the operation results that the client can expect to receive from
the service upon completion. In current manifestations of the API, all input and output contracts
are defined as protobuf messages, which provides a strong contractual definition and good
interoperability with multiple programming languages.

All operations are catalogued in the [**operation directory**](operations). There is a separate
documentation page for each operation, which will specify the correct opcode to use, and will
provide links to the input and output contracts.

In order to make an API call, the client must use the [**wire protocol
specification**](wire_protocol.md) to form a valid request to the service. The request header must
contain the opcode of the operation being performed, and the request body must contain serialized
data bytes conforming to that operation's input contract. The service will execute the operation,
and form a response according to the wire protocol. The response will contain serialized data bytes
conforming to the operation's output contract.

## Selecting Providers

All of the operations in the API are implemented by back-end modules known as **providers**. A
provider is a module that is capable of implementing operations by making use of available platform
hardware or services. For example, if cryptographic services are supplied by a hardware secure
element of some kind, then there would be a provider for that secure element. There may be a
different provider for a software-only solution. And so forth. It is valid for these different
providers to co-reside within the service. The availability of providers is governed by
configuration that is applied at service-deployment time.

While the service can be composed from multiple providers, any given API operation needs to be
implemented by a single provider. This means that client code needs to specify the target provider
when it makes an operation request. To achieve this, the service assigns an 8-bit integer value
(from 0 to 255) to each available provider. In practice, the number of providers would be very
small: probably just two or three. This integer value is the **provider identifier**. Client code
must set a single provider identifier in each API request. The [**wire protocol
specification**](wire_protocol.md) explains how a request header field is used to route the request
to the correct provider.

In order to set a provider identifier in an API request, the client must first be able to determine
what providers are available, what their identifiers are, and what their characteristics are (such
as whether they are hardware-backed or software-backed). This is done by first referencing the
**core provider**. The core provider is the only provider that is guaranteed to be available in any
deployment of the service. It has a provider identifier of zero (`0x00`, the only reserved value for
provider identifiers). The core provider is special in that it doesn't implement any security or
cryptographic operations. The core provider is used to represent the service as a whole. The
operations of the core provider can be used to gather information about the health and configuration
of the service. It can be used to ping the service to check whether it is responsive, and to check
the highest version of the wire protocol that it supports. The core provider can also be used to get
information about the cryptographic providers, their characteristics and their 8-bit identifier
values. Based on this information, the client can determine which provider is best suited to its
requirements. It can then use the integer identifier of that provider to make API requests.

The expected pattern is that a client would determine a single provider that best suits its needs,
and then use that provider exclusively for all cryptographic operations. While this usage would be
considered typical, it is certainly not enforced. There is nothing to prevent a client from using
different providers for different operations if it so desires. In many deployments, it is possible
that only a single cryptographic provider would be available anyway. To determine the best available
or most suitable provider, a client application can use the capability check mechanism, described
below.

## Open-Closed Principle

The API is designed to evolve over time. This evolution will be governed by the *open-closed
principle*. In practice, this means that each operation in the API, once introduced, will not be
contractually modified. The API can only change by introducing new operations. This preserves
backwards compatibility with client code. Any client code that makes use of any given operation will
continue to work, even if new operations are introduced.

### Non-breaking changes

In some cases, it might be helpful to update contracts in a *non-breaking way*. If multiple
operations share the same parameters (such as the PSA algorithm or key attributes), this avoids
having to duplicate all of them if an update is done to those parameters. To make sure that backward
compatibility with client code is always preserved and that the Parsec service remains stable
through updates, the allowed changes are restricted to the following:

- Adding a new *optional* field to an existing type. If not set, the contract containing the new
   field should behave exactly the same as the version without it. The new field should be ignored
   if not recognized by an old version. That means that the functionality brought by a new optional
   field might not be exercised if the service is older than the client, without an error returned.
- Adding a new variant to an existing enumerated type. If not recognized during deserialization, the
   `InvalidEncoding` error should be returned.

In the future, [capabilities discovery](https://github.com/parallaxsecond/parsec/issues/426)
operations might be added to give a way for clients to check what parameters are supported. See the
history of this feature [here](https://github.com/parallaxsecond/parsec/issues/388).

## Deprecation

While the open-closed principle dictates that operations will not be contractually changed once they
have been introduced, it may sometimes be necessary to deprecate specific operations. This will
often be to encourage the use of a new operation with an improved feature set and a different
contract. Deprecation is largely a documentation exercise: the [**operation directory**](operations)
will indicate when an operation has been deprecated. This does not mean that the operation will no
longer work. It simply means that any new use of the operation is strongly discouraged in favor of a
better alternative.

## Capability Checks

The API includes a capability check operation, which allows the client to determine the set of
operations that are available. There are two reasons why the client needs this capability check:

- The API can evolve over time, introducing new operations with new opcodes. In general, a client
   cannot know whether it is talking to a service that supports these newer operations, so it needs
   to check the level of support in advance.
- Different cryptographic providers have different capabilities. An operation that is supported in
   one provider might not be supported in another.

Refer to the [**operation directory**](operations) for information on how to perform a capability
check.

## Application Identity

Every client application that uses the API must present an **application identity**. Application
identities are arbitrary byte strings, which are used by the service to isolate the activities of
one client from those of another. The storage of secure assets such as keys is segregated on a
per-client basis: assets created by one client cannot be accessed by another. The service always
uses the client's application identity string to maintain this separation.

The means by which application identities are generated or assigned is outside of the scope of this
specification. The only requirements for application identities is that they must be **unique** and
**stable**. This means that any given application identity string can be used to identify one and
only one client application. It also means that the application identity string for any given client
should remain the same over time, even across system resets.

The granularity of application identities is not defined. In particular, there is no assumption that
a client *application* corresponds precisely with a single client *process*. A client application
might be composed of multiple processes. Conversely, a single process might contain multiple
distinct client applications. Client applications might also be organized into isolated environments
such as containers. Provided that client application is able to present a unique and stable identity
string for each API call, it does not matter how they are structured and deployed.

## Authentication and Sessions

Clients present their identity strings to the service on each API call. As set out in the [**wire
protocol specification**](wire_protocol.md), they do this using the **authentication** field of the
API request.

There are currently three ways in which the client can use the authentication field to share its
identity with the service:

- **direct authentication**.
- **authentication tokens**.
- **Unix peer credentials**.

With **direct authentication**, the client authenticates the request by directly copying the
application identity string into the **authentication** field of the request.

With **authentication tokens**, the client obtains a token from an identity provider and sends it as
the **authentication** field of the request. The token is reusable for a specified duration of time,
after which a new one must be issued. The application identity is contained in the token and can be
extracted by the service after verifying the authenticity of the token. A more detailed description
of authentication tokens and their lifecycle is present in the [**sytem architecture
specification**](../archive/system_architecture.md).

With **Unix peer credentials**, the client authenticates by self-declaring its Unix user identifier
(UID) inside the **authentication** field of the request. The Parsec service verifies that this
self-declared UID matches the actual UID of the connecting process via the Unix peer credentials
mechanism.

When it makes an API request, the client needs to tell the server which kind of authentication is
being used. This is so that the server knows how to interpret the bytes in the **authentication**
field of the request. As described in the [**wire protocol specification**](wire_protocol.md), the
client does this by setting an integer value in the **auth type** field of the request header. The
permitted numerical values for this field are given as follows:-

- A value of 0 (`0x00`) indicates that there is no authentication. The service will not expect any
   content in the **authentication** field of the request. If any authentication bytes are present,
   they will be ignored, but the request will still be considered valid. (For clients, it is
   considered bad practice to supply a non-empty **authentication** field in this case, because it
   is contradictory to supply authentication material while indicating an unauthenticated call, and
   it indicates improper coding or a possible defect on the client side). See the section below on
   unauthenticated operations.
- A value of 1 (`0x01`) indicates direct authentication. The service will expect the
   **authentication** field to contain a cleartext copy of the application identity, encoded as a
   UTF-8 string.
- A value of 2 (`0x02`) indicates authentication tokens. This authentication type is not currently
   supported. The service will expect the **authentication** field to contain a JWT token. Tokens
   must be signed with the private key of the identity provider and their validity period must cover
   the moment when the check is done.
- A value of 3 (`0x03`) indicates Unix peer credentials authentication. The service expects the
   **authentication** field to contain the Unix user identifier (UID, **not** username) of the
   connecting process as a zero-padded little-endian 32-bit unsigned integer. The Parsec service
   will verify that this self-declared UID is consistent with the UID from the Unix peer
   credentials.
- A value of 4 (`0x04`) indicates authentication through JWT SPIFFE Verifiable Identity Document.
   The service expects the **authentication** field to contain a JWT-SVID token as described in the
   [SPIFFE standard](https://github.com/spiffe/spiffe/blob/master/standards/JWT-SVID.md). The token
   must be encoded using the JWS Compact Serialization. The service will use the SPIFFE ID validated
   from the SVID as application identity.

Other values are unsupported and will be rejected by the service. See the
[authenticators](../parsec_service/authenticators.md) page for more details.

## Unauthenticated Operations

Authentication via the application identity is only needed for cryptographic operations. Core
provider operations do not require authentication. Core provider operations include those that are
used to ping the service and gather information about its capabilities. These operations neither
require nor support any notion of per-client isolation. Consequently, they can be called without any
authentication. For requests to the core provider, the **auth type** header field should always be
set to 0 (`0x00`).

## Content Type and Accept Type

As per the [**wire protocol specification**](wire_protocol.md), API request headers contain fields
for **content type** and **accept type**, which respectively indicate how the request body and
response body are encoded. Currently, the only supported value for these fields is one (`0x01`),
meaning that all request and response bodies contain serialized protobuf messages. All other values
are unsupported and will be rejected by the service.

## PSA Crypto Operations

The majority of the operations in this API are derived from the [**PSA Crypto API
Specification**](https://github.com/ARMmbed/mbed-crypto/blob/psa-crypto-api/docs/PSA_Cryptography_API_Specification.pdf).
There is a near one-to-one correspondence of functional operations between the two APIs. The main
difference is that the PSA Crypto API is defined in the C programming language, whereas the API
described here is language-agnostic. There is otherwise a close contractual equivalence between the
two, and this is intentional.

In the [**operation directory**](operations), operations derived from the PSA Crypto API have
symbolic names that start with the **Psa** prefix, and their numerical opcodes are all in the
1,000-1,999 (decimal) range. Opcode ranges are an important aspect of the API design because they
form the basis of an extensibility mechanism. In the future, it will be possible for contributors to
partition the numerical opcode space with ranges for custom operations.

## Key Names, Enumerating and Referencing

While this API is closely aligned with the PSA Crypto API, there are some differences. One important
difference is in the conventions used to name and reference cryptographic keys.

In the PSA Crypto API, every key has a 32-bit numerical identifier. This identifier is set by the
caller when the persistent key is created. Client code then uses this 32-bit identifier to refer to
the key for use in any cryptographic operation.

This API is different: the key names are not 32-bit numerical values, they are **UTF-8 strings**.
Cryptographic operations are all specified in terms of the key name string. It is important to
understand that the opacity of keys - one of the critical design characteristics of the PSA Crypto
API - is preserved here. Key names are used to reference keys for cryptographic operations, but the
actual key material is *never* exposed to the caller of the API unless an explicit operation is
invoked to export the key (and the key's usage policy permits for such an export to occur).

The use of string names offers greater flexibility in how names can be chosen and structured. It
allows for names to be readable and meaningful. It also allows for names to follow a structured
pattern with separators, similar to a file path. This allows keys to not only be named in meaningful
ways, but also for them to be organized according to a meaningful structure, just like files on a
file system. Keys with a similar purpose, for example, can be stored in the same part of the
notional "tree".

Key names adopt a path structure similar to Unix file paths, such as `/keys/rsa/my_key_1`.

This key naming convention permits for the API to support key **enumeration**, where the client is
able to determine the set of known keys according to some wildcard pattern such as `/keys/rsa/*`.

All key names are implicitly in a per-client namespace, so it is impossible for one client
application to enumerate or otherwise discover the keys that are owned by another client
application.

Providers can impose length restrictions on key names to help with internal storage and argument
validation. This API reference does not define any single fixed maximum length. Clients must
determine the maximum length at runtime using the capability checking mechanism.

## Service discovery

Establishing a connection between the client and the service requires common understanding of the
communication mechanism and the address at which the service can be found. While the default
mechanism is through a Unix domain socket found at `/run/parsec/parsec.sock`, that can be easily
changed by an admin for various reasons. Clients must therefore be made aware of such changes and
the defined mechanism for this uses environment variables.

The `PARSEC_SERVICE_ENDPOINT` environment variable is the preferred method of indicating to a
process where to find the communication channel for the Parsec service. The variable should contain
a URI as defined in [RFC3986](https://www.ietf.org/rfc/rfc3986.txt) - for example, for the default
domain socket location, the URI would be `unix:/run/parsec/parsec.sock`. Client libraries must be
capable of scanning for the environment variable, parsing it, and bootstrapping communication using
the URI, with no input from the client application. The libraries must also allow the application to
override these settings.

## Full API Reference

For the full reference guide to individual API operations, please refer to the [**operation
directory**](operations).

*Copyright 2019 Contributors to the Parsec project.*
