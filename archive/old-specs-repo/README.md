<!-- [CI](https://github.com/vacp2p/specs/workflows/CI/badge.svg) -->

This repository contains the specs for [vac](https://vac.dev), a modular peer-to-peer messaging stack, with a focus on secure messaging. A detailed explanation of the vac and its design goals can be found [here](https://vac.dev/vac-overview).

## Status

The entire vac protocol is under active development, each specification has its own `status` which is reflected through the version number at the top of every document. We use [semver](https://semver.org/) to version these specifications.

## Protocols

These protocols define various components of the [vac](https://vac.dev) stack.

### Waku

Waku is a protocol that replaces Whisper ([EIP-627](https://eips.ethereum.org/EIPS/eip-627)). Waku v2 is an upgrade of Waku v1 that is in active development and offer significant improvements. You can read more about the [plan for it](https://vac.dev/waku-v2-plan) and an [update](https://vac.dev/waku-v2-update).

#### Version 2
  - [waku/2](./specs/waku/v2/waku-v2) specs for `waku/2` version, current draft version
  - [waku/2 relay](./specs/waku/v2/waku-relay) spec for WakuRelay, current draft version
  - [waku/2 filter](./specs/waku/v2/waku-filter) spec for WakuFilter, current draft version
  - [waku/2 store](./specs/waku/v2/waku-store) spec for WakuStore, current draft version
  - [waku/2 message](./specs/waku/v2/waku-message) spec for Waku Message, current draft version
  - [waku/2 bridge](./specs/waku/v2/waku-bridge) spec for Waku bridge with v1, alpha
  - [waku/2 rln relay](./specs/waku/v2/waku-rln-relay) spec for Waku Relay with RLN, alpha
  - [waku/2 swap](./specs/waku/v2/waku-swap-accounting) spec for Waku Swap Accounting, alpha
  - [waku/2 rpc api](./specs/waku/v2/waku-v2-rpc-api) - Waku RPC API for Waku v2 nodes, alpha

#### Version 0 and 1
 - [waku/0](./specs/waku/v1/waku-0) specs for `waku/0` version, now deprecated
 - [waku/1](./specs/waku/v1/waku-1) specs for `waku/1` version, current stable version
 - [envelope data format](./specs/waku/v1/envelope-data-format) [waku](./waku/waku) envelope data field specification.
 - [mailserver](./specs/waku/v1/mailserver) - Mailserver specification for archiving and delivering historical [waku](./specs/waku/waku) envelopes on demand.
 - [rpc api](./specs/waku/v1/waku-rpc-api) - Waku RPC API for Waku v1 nodes.

### Data sync

 - [mvds](./specs/mvds) - Data Synchronization protocol for unreliable transports.
 - [remote log](./specs/remote-log) - Remote replication of local logs.
 - [mvds metadata](./specs/mvds-metadata) - Metadata field for [MVDS](./specs/mvds) messages.

## Style guide

Sequence diagrams are generated using [Mscgen](http://www.mcternan.me.uk/mscgen/) like this: `mscgen -T png -i input.msc -o output.png`. Both the source and generated image should be in source control. For ease of readability, the generated image is embedded inside the main spec document. 

Alternatively, [mscgenjs](https://github.com/mscgenjs/mscgenjs-cli) can be used to generate sequence diagrams (mscgenjs produces better quality figures especially concerning lines' spaces and figures' margins). Once installed, the following command can be used to generate the sequence diagrams `mscgenjs -T png -i input.msc -o output.png`. More details on the installation and compilation are given in [mscgenjs repository](https://github.com/mscgenjs/mscgenjs-cli). You may try the online playground https://mscgen.js.org/ as well to get a sense of the output figures. 

The lifecycle of the specs follows the [COSS Lifecycle](https://rfc.unprotocols.org/spec:2/COSS/)

## Meta

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Linting

### Spellcheck

To run the spellchecker locally, you must install [pyspelling](https://facelessuser.github.io/pyspelling/).

It can then be run with the following command:

```console
pyspelling -c .pyspelling.yml
```

Words that should be ignored or are unrecognized must be added to the [wordlist](./wordlist.txt).

### Markdown Verification

We use [remark](https://remark.js.org/) to verify our markdown. You can easily run this tool simply by using our `npm` package:

```console
npm install
npm run lint
```

### Textlint

We use [textlint](https://textlint.github.io/) for extra markdown verification. You can easily run this tool simply by using our `npm` package:

```console
npm install
npm run textlint
```