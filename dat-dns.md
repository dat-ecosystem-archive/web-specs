# The `dat://` DNS-over-HTTPS protocol

Browsers needs a mechanism by which users can securely deploy a site at a domain, eg `dat://example.com`. Ideally, this mechanism will allow discovery and automatic redirection; for instance, if the user browses to `https://example.com`, they should be prompted to redirect to `dat://example.com`.

## Requirements

1. **MUST** provide a single canonical Dat URL for the domain. It should not be possible for multiple Dats to be specified within a domain. This means the Dat must be specified by a single fixed file, or by DNS.
2. **MUST** not be controllable by non-owners of the domain. It should not be possible for user-input or injections to set the Dat URL.
3. **MUST** cryptographically authenticate the validity of the entry.
4. **SHOULD** be accessible to as many users as possible (eg response headers are frequently unsettable).

## Options

### DNS

Satisfies requirement 1 and 2, fails requirement 3, not ideal for requirement 4. DNSSEC could be used to satisfy requirement 3, but support for DNSSEC by gTLDs is limited.

### HTML tags (eg meta)

Satisfies requirements 3 (with HTTPS) and 4, fails requirements 1 and 2.

### .Well-known file over TLS

Satisfies requirements 1, 2, 3, and 4.

## Discussion

https://github.com/beakerbrowser/beaker/issues/227

https://github.com/beakerbrowser/beaker/issues/228

## Specification

Place a file at `/.well-known/dat` with the following schema:

```
{dat-url}
TTL={time in seconds}
```

TTL is optional and will default to `3600` (one hour). If set to `0`, the entry is not cached.

### Dat-name Resolution

Resolution of a site at `dat://hostname` will occur with the following process:

 - Browser checks its dat names cache. If a non-expired entry is found, return with the entry.
 - Browser issues a GET request to `https://hostname/.well-known/dat`.
 - If the server responds with a `404 Not Found` status, store a null entry in the cache with a TTL of `3600` and return a failed lookup.
 - If the server responds with anything other than a `200 OK` status, return a failed lookup.
 - If the server responds with a malformed file, return a failed lookup.
 - If the response includes no TTL, set to default `3600`.
 - If the response includes a non-zero TTL, store the entry in the dat-name cache.
 - Return the entry.

### Discovery

Visits to sites served over HTTPS should trigger Dat-name Resolution. If an entry is found, the browser UI will present the user with an option to redirect to the Dat site. Discovery traffic will be throttled by the Dat-name caching.
