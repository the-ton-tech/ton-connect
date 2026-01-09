# Object Storage API

Object Storage is a service used to temporarily store objects. It allows clients to store data and retrieve it later using a unique URL.

* **Stored objects have TTL (Time-To-Live).** Objects are automatically removed after the TTL expires.

## HTTP Object Storage

Client uploads an object to the object storage.

```tsx
request
    POST /store?ttl=<ttl_in_seconds>
```

Body:
```json
{
  "object": "base64_encoded_object"
}
```

**ttl** – Time-to-live in seconds. The object will be automatically removed after this duration. Object Storage services should support at least 300 seconds TTL.

If the TTL exceeds the hard limit of the storage server, it should respond with HTTP 400.

When the object storage receives a `base64_encoded_object`, it stores it and generates a response `StorageResponse`:

```json
{
  "get_url": "<url_to_retrieve_stored_object>"
}
```

The `get_url` is a unique URL that can be used to retrieve the stored object.

Client retrieves the stored object using the `get_url`.

```tsx
request
    GET <get_url>
```

The object storage responds with the stored object:

```js
{
  "object": <base64_encoded_object>
}
```

Object Storage removes the object as soon as it is retrieved, or when the TTL expires, whichever comes first.

If the object was not found (expired or never existed), the storage should respond with HTTP 404.
