# Object Storage API

Object Storage is a service used to temporarily store objects. It allows clients to store data and retrieve it later using a unique URL.

* **Object Storage is maintained by the wallet provider**. App developers do not have to choose or build an **Object Storage**. Each wallet’s **Object Storage** is listed in the [wallets-list](https://github.com/ton-blockchain/wallets-list) config.
* **Stored objects have TTL (Time-To-Live).** Objects are automatically removed after the TTL expires.

## HTTP Object Storage

### Store

Client uploads an object to the object storage.

```tsx
request
    POST /objects?ttl=<ttl_in_seconds>

    Content-Type: <content_type>
    body: <object>
```

**ttl** – Time-to-live in seconds. The object will be automatically removed after this duration. Object Storage services should support at least 300 seconds TTL.

If the TTL exceeds the hard limit of the storage server, it should respond with HTTP 400.

Response:

```json
{
  "get_url": "<base_url>/objects/<hash>"
}
```

The `get_url` MUST follow the format `/objects/:hash`, where `:hash` is the SHA256 hash of the stored object (hex-encoded).

**Hash calculation:** `hash = SHA256(byte(content) + byte(content_type))`, where `content` is the raw object bytes and `content_type` is the value of the `Content-Type` header. The hash is hex-encoded in the URL path.

### Retrieve

Client retrieves the stored object using the `get_url`.

```tsx
request
    GET /objects/:hash
```

Response: the stored object.

Object Storage removes the object when the TTL expires.

If the object was not found (expired or never existed), the storage should respond with HTTP 404.
