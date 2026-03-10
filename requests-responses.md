# Requests and Responses

App sends requests to the wallet. Wallet sends responses and events to the app.

```tsx
type AppMessage = ConnectRequest | AppRequest;

type WalletMessage = WalletResponse | WalletEvent;
```

### App manifest
App needs to have its manifest to pass meta information to the wallet. Manifest is a JSON file named as `tonconnect-manifest.json` following format:

```json
{
  "url": "<app-url>",                        // required
  "name": "<app-name>",                      // required
  "iconUrl": "<app-icon-url>",               // required
  "termsOfUseUrl": "<terms-of-use-url>",     // optional
  "privacyPolicyUrl": "<privacy-policy-url>" // optional
}
```

Best practice is to place the manifest in the root of your app, e.g. `https://myapp.com/tonconnect-manifest.json`. It allows the wallet to handle your app better and improve the UX connected to your app.
Make sure that manifest is available to GET by its URL.

#### Fields description
- `url` -- app URL. Will be used as the dapp identifier. Will be used to open the dapp after click to its icon in the wallet. It is recommended to pass url without closing slash, e.g. 'https://mydapp.com' instead of 'https://mydapp.com/'.
- `name` -- app name. Might be simple, will not be used as identifier.
- `iconUrl` -- Url to the app icon. Must be PNG, ICO, ... format. SVG icons are not supported. Perfectly pass url to a 180x180px PNG icon.
- `termsOfUseUrl` -- (optional) url to the Terms Of Use document. Optional for usual apps, but required for the apps which is placed in the Tonkeeper recommended apps list.
- `privacyPolicyUrl` -- (optional) url to the Privacy Policy document. Optional for usual apps, but required for the apps which is placed in the Tonkeeper recommended apps list.

### Initiating connection

App’s request message is **InitialRequest**.

```tsx
type ConnectRequest = {
  manifestUrl: string;
  items: ConnectItem[], // data items to share with the app
}

// In the future we may add other personal items.
// Or, instead of the wallet address we may ask for per-service ID.
type ConnectItem = TonAddressItem | TonProofItem | ...;

type TonAddressItem = {
  name: "ton_addr";
}
type TonProofItem = {
  name: "ton_proof";
  payload: string; // arbitrary payload, e.g. nonce + expiration timestamp.
}
```

ConnectRequest description:
- `manifestUrl`: link to the app's tonconnect-manifest.json
- `items`: data items to share with the app.

Wallet responds with **ConnectEvent** message if the user approves the request. 

```tsx
type ConnectEvent = ConnectEventSuccess | ConnectEventError;

type ConnectEventSuccess = {
  event: "connect";
  id: number; // increasing event counter
  payload: {
      items: ConnectItemReply[];
      device: DeviceInfo;   
  }
}
type ConnectEventError = {
  event: "connect_error",
  id: number; // increasing event counter
  payload: {
      code: number;
      message: string;
  }
}

type DeviceInfo = {
  platform: "iphone" | "ipad" | "android" | "windows" | "mac" | "linux";
  appName:      string; // e.g. "Tonkeeper"  
  appVersion:  string; // e.g. "2.3.367"
  maxProtocolVersion: number;
  features: Feature[]; // list of supported features and methods in RPC
}

type Feature =
  | {
      name: 'SendTransaction';
      maxMessages: number; // maximum number of messages in one `SendTransaction` that the wallet supports
      extraCurrencySupported?: boolean; // indicates if the wallet supports extra currencies
    }
  | {
      name: 'SignData';
      types: ('text' | 'binary' | 'cell')[]; // array of supported data types for signing
    }
  | {
      name: 'SignMessage';
    }
  | {
      name: 'SendTransactionDraft';
      types: ('ton' | 'jetton' | 'nft')[];
    }
  | {
      name: 'SignMessageDraft';
      types: ('ton' | 'jetton' | 'nft')[];
    }
  | {
      name: 'ActionDraft';
    }
  | {
      name: 'Intents';
      types: ('txDraft' | 'signMsgDraft' | 'actionDraft' | 'signData')[]; // requests supported via intent URL
    };

type ConnectItemReply = TonAddressItemReply | TonProofItemReply ...;

// Untrusted data returned by the wallet. 
// If you need a guarantee that the user owns this address and public key, you need to additionally request a ton_proof.
type TonAddressItemReply = {
  name: "ton_addr";
  address: string; // TON address raw (`0:<hex>`)
  network: NETWORK; // network global_id
  publicKey: string; // HEX string without 0x
  walletStateInit: string; // Base64 (not url safe) encoded stateinit cell for the wallet contract
}

type TonProofItemReply = TonProofItemReplySuccess | TonProofItemReplyError;

type TonProofItemReplySuccess = {
  name: "ton_proof";
  proof: {
    timestamp: string; // 64-bit unix epoch time of the signing operation (seconds)
    domain: {
      lengthBytes: number; // AppDomain Length
      value: string;  // app domain name (as url part, without encoding) 
    };
    signature: string; // base64-encoded signature
    payload: string; // payload from the request
  }
}

type TonProofItemReplyError = {
  name: "ton_addr";
  error: {
      code: ConnectItemErrorCode;
      message?: string;
  }
}

enum NETWORK {
  MAINNET = '-239',
  TESTNET = '-3'
}
```

**Connect event error codes:**

| code | description                  |
|------|------------------------------|
| 0    | Unknown error                |
| 1    | Bad request                  |
| 2    | App manifest not found       |
| 3    | App manifest content error   |
| 100  | Unknown app                  |
| 300  | User declined the connection |

**Connect item error codes:**

| code | description                  |
|------|------------------------------|
| 0    | Unknown error                |
| 400  | Method is not supported      |

If the wallet doesn't support the requested `ConnectItem` (e.g. "ton_proof"), it must send reply with the following ConnectItemReply corresponding to the requested item.
with following structure: 
```ts
type ConnectItemReplyError = {
  name: "<requested-connect-item-name>";
  error: {
      code: 400;
      message?: string;
  }
}
```

### Address proof signature (`ton_proof`)

If `TonProofItem` is requested, wallet proves ownership of the selected account’s key. The signed message is bound to:

- Unique prefix to separate messages from on-chain messages. (`ton-connect`)
- Wallet address.
- App domain
- Signing timestamp
- App’s custom payload (where server may put its nonce, cookie id, expiration time).

```
message = utf8_encode("ton-proof-item-v2/") ++ 
          Address ++
          AppDomain ++
          Timestamp ++  
          Payload 
signature = Ed25519Sign(privkey, sha256(0xffff ++ utf8_encode("ton-connect") ++ sha256(message)))
```

where:

* `Address` is the wallet address encoded as a sequence: 
   * `workchain`: 32-bit signed integer big endian;
   * `hash`: 256-bit unsigned integer big endian;
* `AppDomain` is Length ++ EncodedDomainName
  - `Length` is 32-bit value of utf-8 encoded app domain name length in bytes
  - `EncodedDomainName` id `Length`-byte  utf-8 encoded app domain name
  - For standard dApp connections, the domain name MUST contain at least one dot (.) character with valid characters on both sides. Domains without proper dot-separation (e.g., "tonkeeper", "tonhub") are reserved for native wallet integrations only and MUST NOT be accepted from external dApps.
* `Timestamp` 64-bit unix epoch time of the signing operation 
* `Payload` is a variable-length binary string.

Note: payload is variable-length untrusted data. To avoid using unnecessary length prefixes we simply put it last in the message.

The signature must be verified by public key:

1. First, try to obtain public key via `get_public_key` get-method on smart contract deployed at `Address`.

2. If the smart contract is not deployed yet, or the get-method is missing, you need:

   2.1. Parse `TonAddressItemReply.walletStateInit` and get public key from stateInit. You can compare the `walletStateInit.code` with the code of standard wallets contracts and parse the data according to the found wallet version.

   2.2. Check that `TonAddressItemReply.publicKey` equals to obtained public key
   
   2.3. Check that `TonAddressItemReply.walletStateInit.hash()` equals to `TonAddressItemReply.address`. `.hash()` means BoC hash.

## Messages

- All messages from the app to the wallet are requests for an operation.
- Messages from the wallet to the application can be either responses to app requests or events triggered by user actions on the side of the wallet.

**Available operations:**

- sendTransaction
- signData
- signMessage
- disconnect

**Available events:**

- connect
- connect_error
- disconnect

### Structure

**All app requests have the following structure (like json-rpc 2.0)**
```tsx
interface AppRequest {
	method: string;
	params: string[];
	id: string;
}
```
Where 
- `method`: name of the operation ('sendTransaction', 'signMessage', ...)
- `params`: array of the operation specific parameters
- `id`: increasing identifier that allows to match requests and responses


**Wallet messages are responses or events.**

Response is an object formatted as a json-rpc 2.0 response. Response `id` must match request's id. 

Wallet doesn't accept any request with an id that does not greater the last processed request id of that session.

```tsx
type WalletResponse = WalletResponseSuccess | WalletResponseError;

interface WalletResponseSuccess {
    result: string;
    id: string;
}

interface WalletResponseError {
    error: { code: number; message: string; data?: unknown };
    id: string;
}
```

Event is an object with property `event` that is equal to event's name, `id` that is increasing event counter (**not** related to `request.id` because there is no request for an event), and `payload` that contains event additional data. 
```tsx
interface WalletEvent {
    event: WalletEventName;
    id: number; // increasing event counter
    payload: <event-payload>; // specific payload for each event
}

type WalletEventName = 'connect' | 'connect_error' | 'disconnect';
```

Wallet must increase `id` while generating a new event. (Every next event must have `id` > previous event `id`)

DApp doesn't accept any event with an id that does not greater the last processed event id of that session. 

### Methods

#### Sign and send transaction

App sends **SendTransactionRequest**:

```tsx
interface SendTransactionRequest {
	method: 'sendTransaction';
	params: [<transaction-payload>];
	id: string;
}
```

Where `<transaction-payload>` is JSON string with following properties:

* `valid_until` (integer, optional): unix timestamp. after th moment transaction will be invalid.
* `network` (NETWORK, optional): The network (mainnet or testnet) where DApp intends to send the transaction. If not set, the transaction is sent to the network currently set in the wallet, but this is not safe and DApp should always strive to set the network. If the `network` parameter is set, but the wallet has a different network set, the wallet should show an alert and DO NOT ALLOW TO SEND this transaction.
* `from` (string in <wc>:<hex> format, optional) - The sender address from which DApp intends to send the transaction. If not set, wallet allows user to select the sender's address at the moment of transaction approval. If `from` parameter is set, the wallet should DO NOT ALLOW user to select the sender's address; If sending from the specified address is impossible, the wallet should show an alert and DO NOT ALLOW TO SEND this transaction.
* `messages` (array of messages): 1-4 outgoing messages from the wallet contract to other accounts. All messages are sent out in order, however **the wallet cannot guarantee that messages will be delivered and executed in same order**.

Message structure:
* `address` (string): message destination in user-friendly format
* `amount` (decimal string): number of nanocoins to send.
* `payload` (string base64, optional): raw one-cell BoC encoded in Base64.
* `stateInit` (string base64, optional): raw once-cell BoC encoded in Base64.
* `extra_currency` (object, optional): extra currency to send with the message. 


<details>
<summary>Common cases</summary>
1. No payload, no stateInit: simple transfer without a message.
2. payload is prefixed with 32 zero bits, no stateInit: simple transfer with a text message.
3. No payload or prefixed with 32 zero bits; stateInit is present: deployment of the contract.
</details>

<details>
<summary>Example</summary>

```json5
{
  "valid_until": 1658253458,
  "network": "-239", // enum NETWORK { MAINNET = '-239', TESTNET = '-3'}
  "from": "0:348bcf827469c5fc38541c77fdd91d4e347eac200f6f2d9fd62dc08885f0415f",
  "messages": [
    {
      "address": "EQBBJBB3HagsujBqVfqeDUPJ0kXjgTPLWPFFffuNXNiJL0aA",
      "amount": "20000000",
      "stateInit": "base64bocblahblahblah==", //deploy contract
      "extra_currency": {
        "239": "1000000000"
      }
    },{
      "address": "EQDmnxDMhId6v1Ofg_h5KR5coWlFG6e86Ro3pc7Tq4CA0-Jn",
      "amount": "60000000",
      "payload": "base64bocblahblahblah==" //transfer nft to new deployed account 0:412410771DA82CBA306A55FA9E0D43C9D245E38133CB58F1457DFB8D5CD8892F
    }
  ]
}
```
</details>


Wallet replies with **SendTransactionResponse**:

```tsx
type SendTransactionResponse = SendTransactionResponseSuccess | SendTransactionResponseError; 

interface SendTransactionResponseSuccess {
    result: <boc>;
    id: string;
	
}

interface SendTransactionResponseError {
   error: { code: number; message: string };
   id: string;
}
```

**Error codes:**

| code | description                   |
|------|-------------------------------|
| 0    | Unknown error                 |
| 1    | Bad request                   |
| 100  | Unknown app                   |
| 300  | User declined the transaction |
| 400  | Method not supported       |


#### Sign Data

App sends **SignDataRequest**:

```tsx
interface SignDataRequest {
	method: 'signData';
	params: [<sign-data-payload>];
	id: string;
}
```

Where `<sign-data-payload>` is JSON string with one of the 3 types of payload:

- **Text**. JSON object with following properties:
  - **type** (string): 'text'
  - **network** (NETWORK, optional): The network (mainnet or testnet) where DApp intends to sign data. If not set, the data is signed with the network currently set in the wallet, but this is not safe and DApp should always strive to set the network. If the `network` parameter is set, but the wallet has a different network set, the wallet should show an alert and DO NOT ALLOW TO SIGN data.
  - **from** (string in raw format, optional): The signer address from which DApp intends to sign data. If not set, wallet allows user to select the signer's address at the moment of signing data. If `from` parameter is set, the wallet should DO NOT ALLOW user to select the signer's address; If signing from the specified address is impossible, the wallet should show an alert and DO NOT ALLOW TO SIGN data.
  - **text** (string): arbitrary UTF-8 text to sign.
  
- **Binary**. JSON object with following properties:
  - **type** (string): 'binary'
  - **network** (NETWORK, optional): The network (mainnet or testnet) where DApp intends to sign data. If not set, the data is signed with the network currently set in the wallet, but this is not safe and DApp should always strive to set the network. If the `network` parameter is set, but the wallet has a different network set, the wallet should show an alert and DO NOT ALLOW TO SIGN data.
  - **from** (string in raw format, optional): The signer address from which DApp intends to sign data. If not set, wallet allows user to select the signer's address at the moment of signing data. If `from` parameter is set, the wallet should DO NOT ALLOW user to select the signer's address; If signing from the specified address is impossible, the wallet should show an alert and DO NOT ALLOW TO SIGN data.
  - **bytes** (string): base64 (not url safe) encoded arbitrary bytes array to sign.

- **Cell**. JSON object with following properties:
  - **type** (string): 'cell'
  - **network** (NETWORK, optional): The network (mainnet or testnet) where DApp intends to sign data. If not set, the data is signed with the network currently set in the wallet, but this is not safe and DApp should always strive to set the network. If the `network` parameter is set, but the wallet has a different network set, the wallet should show an alert and DO NOT ALLOW TO SIGN data.
  - **from** (string in raw format, optional): The signer address from which DApp intends to sign data. If not set, wallet allows user to select the signer's address at the moment of signing data. If `from` parameter is set, the wallet should DO NOT ALLOW user to select the signer's address; If signing from the specified address is impossible, the wallet should show an alert and DO NOT ALLOW TO SIGN data.
  - **schema** (string): TL-B schema of the cell payload as an UTF-8 string.  
  *If the schema contains several type definitions, the **last** declared type is treated as the root during serialization and deserialization.*
  - **cell** (string): base64 (not url safe) encoded BoC (single-root) with arbitrary cell to sign.

**Examples:**

```json5
{
  "type": "text",
  "text": "Confirm new 2fa number:\n+1 234 567 8901",
  "network": "-239", // enum NETWORK { MAINNET = '-239', TESTNET = '-3'}
  "from": "0:348bcf827469c5fc38541c77fdd91d4e347eac200f6f2d9fd62dc08885f0415f"
}
```

```json5
{
  "type": "binary",
  "bytes": "1Z/SGh+3HFMKlVHSkN91DpcCzT4C5jzHT3sA/24C5A==",
  "network": "-239", // enum NETWORK { MAINNET = '-239', TESTNET = '-3'}
  "from": "0:348bcf827469c5fc38541c77fdd91d4e347eac200f6f2d9fd62dc08885f0415f"
}
```

```json5
{
  "type": "cell",
  "schema": "transfer#0f8a7ea5 query_id:uint64 amount:(VarUInteger 16) destination:MsgAddress response_destination:MsgAddress custom_payload:(Maybe ^Cell) forward_ton_amount:(VarUInteger 16) forward_payload:(Either Cell ^Cell) = InternalMsgBody;",
  "cell": "te6ccgEBAQEAVwAAqg+KfqVUbeTvKqB4h0AcnDgIAZucsOi6TLrfP6FcuPKEeTI6oB3fF/NBjyqtdov/KtutACCLqvfmyV9kH+Pyo5lcsrJzJDzjBJK6fd+ZnbFQe4+XggI=",
  "network": "-239", // enum NETWORK { MAINNET = '-239', TESTNET = '-3'}
  "from": "0:348bcf827469c5fc38541c77fdd91d4e347eac200f6f2d9fd62dc08885f0415f"
}
```

**Wallet behaviour:**

- If the payload is in the Text format, Wallet MUST show the content from the `text` field to the user with monospace font, without any formatting and with forced line breaks. User SHOULD scroll long message before signing.
- If the payload is in the Binary format, Wallet MUST display a warning message informing that the content being signed is unknown.
- If the payload is in the Cell format, Wallet CAN parse TL-B scheme from the `schema` field and verify payload using this scheme. Otherwise, Wallet MUST display a warning message informing that the content being signed is unknown.
    - If the `schema` contains unparseable data, Wallet MUST display a warning message informing that the content being signed is unknown.
    - If the data in the `cell` field cannot be parsed using the provided TL-B sheme, Wallet MUST display a warning message informing that the content being signed is unknown.
    - If the data is parsed successfully, Wallet MUST display the parsed data to the user in any readable format.

Wallet replies with **SignDataResponse**:

```tsx
type SignDataResponse = SignDataResponseSuccess | SignDataResponseError;

interface SignDataResponseSuccess {
    result: {
        signature: string; // base64 encoded signature
        address: string;   // wallet address in raw format
        timestamp: number; // UNIX timestamp in seconds (UTC) at the moment on creating the signature.
        domain: string;  // app domain name (as url part, without encoding) 
        payload: <sign-data-payload>; // payload received from the request in the `params` field
    };
    id: string;
}

interface SignDataResponseError {
    error: { code: number; message: string };
    id: string;
}
```

**Error codes:**

| code | description               |
|------|---------------------------|
| 0    | Unknown error             |
| 1    | Bad request               |
| 100  | Unknown app               |
| 300  | User declined the request |
| 400  | Method not supported      |

**Signature**

If the payload is in the Text or Binary format, then signature computes as follows:

```
message = 0xffff ++
          utf8_encode("ton-connect/sign-data/") ++
          Address ++
          AppDomain ++
          Timestamp ++
          Payload
signature = Ed25519Sign(privkey, sha256(message))
```

where:

- `Address` is the wallet address encoded as a sequence:
  - `workchain`: 32-bit signed integer big endian;
  - `hash`: 256-bit unsigned integer big endian;
- `AppDomain` is Length ++ EncodedDomainName from dApp manifest without scheme (same as in `connect` event)
  - `Length` is 32-bit value of utf-8 encoded app domain name length in bytes;
  - `EncodedDomainName` is `Length`byte utf-8 encoded app domain name;
- `Timestamp` 64-bit unix epoch time of the signing operation
- `Payload` is Prefix ++ PayloadLength ++ PayloadData:
  - `Prefix` is `utf8_encode("txt")` for text data or `utf8_encode("bin")` for binary data;
  - `PayloadLength` is 32-bit value of PayloadData length in bytes;
  - `PayloadData` is `PayloadLength` byte payload data, which is `utf8_encode(text)` for text data or bytes array for binary data;

If the payload is in the Cell format, then signature computes as follows:

```tsx
let payload = beginCell()
	.storeUint(0x75569022, 32)
	.storeUint(crc32(schema), 32)
	.storeUint(timestamp, 64)
	.storeAddress(userWalletAddress)
	.storeStringRefTail(appDomain)
	.storeRef(cell);

let signature = Ed25519Sign(payload.hash(), privkey);
```

TL-B:

```tsx
message#75569022 schema_hash:uint32 timestamp:uint64 userAddress:MsgAddress 
					{n:#} appDomain:^(SnakeData ~n) payload:^Cell = Message;
```

where:

- `schema` is the TL-B scheme of the cell payload in the utf-8 encoded string.
- `timestamp` is 64-bit unix epoch time of the signing operation.
- `userWalletAddress` is the user wallet address that signs the payload.
- `appDomain` domain name must be encoded exactly in the DNS wire format specified by [TEP-81](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md), e.g. `com\0stonfi\0`.
- `cell` is the arbitrary payload cell.

**Smart Contract Behaviour**

If receiver of the signed data is a smart contract, then the smart contract MUST perform following steps:

1. Smart contract MUST verify that the message signature belongs to the message.
2. Smart contract MUST verify that first 32 bytes equals to the prefix `0x75569022`.
3. Smart contract MUST verify `schema_hash` field is equal to the expected hash of the schema.
4. Smart contract MUST verify that the message has not expired according to its expiration rules by comparing timestamp from the `timestamp` field. Smart contract MUST have limits on the message expiration.
5. Smart contract MUST check that `userWalletAddress` is equal to the expected wallet address.
6. Smart contract MUST check that `appDomain` is equal to the application app domain. Smart contract CAN compare hash of the `appDomain` instead.

**Which format should App choose?**

If the data is human-readable text, use Text format.

If the data should be used in the TON Blockchain, use Cell format.

If the data needs to be human-readable—but is not textual—use the **Cell** format. In this format, include a `schema` field that contains the TL-B schema of the payload. However, note that the Wallet is not required to display the content of the Cell.

Otherwise, use Binary format.

#### Sign Message

App sends **SignMessageRequest**:

```tsx
interface SignMessageRequest {
	method: 'signMessage';
	params: [<sign-message-payload>];
	id: string;
}
```

Where `<sign-message-payload>` is JSON string with the same structure as `<transaction-payload>` in [`sendTransaction`](#sign-and-send-transaction), including `valid_until`, `network`, `from`, and `messages` fields.

**Wallet behaviour:**

- Wallet MUST prepare the external message with the provided messages (same as in `sendTransaction`).
- Wallet MUST show the messages details to the user for approval.
- Wallet MUST sign the message but MUST NOT send it to the blockchain.
- Wallet returns the signed message as a base64-encoded BoC.

Wallet replies with **SignMessageResponse**:

```tsx
type SignMessageResponse = SignMessageResponseSuccess | SignMessageResponseError;

interface SignMessageResponseSuccess {
    result: { 
        internal_boc: string;  // base64-encoded BoC of the signed internal message
    };
    id: string;
}

interface SignMessageResponseError {
    error: { code: number; message: string };
    id: string;
}
```

**Error codes:**

| code | description               |
|------|---------------------------|
| 0    | Unknown error             |
| 1    | Bad request               |
| 100  | Unknown app               |
| 300  | User declined the request |
| 400  | Method not supported      |

#### Drafts

A **draft** is a payload that describes a blueprint for an action (transaction, sign message, or action). Drafts can be delivered via the [Intent](#intents) URL or via the bridge in an existing session. The following draft types are used when opening an intent link.

##### Send Transaction Draft

App sends **SendTransactionDraftRequest** (or the same structure is used when delivering a draft via [Intent](#intents) URL):

```tsx
interface SendTransactionDraftRequest {
  id: string;
  method: 'txDraft';
  params: {
    vu?: number;
    f?: string;
    n?: string;
    i: TransactionDraftItem[];
  };
}
```

Where `params` is an object with:
* `vu` (number, optional): unix timestamp. After this moment the draft is invalid.
* `f`: (string, optional): sender address in <wc>:<hex> format; semantics match `sendTransaction`.
* `n` (string, optional): target network; semantics match `sendTransaction`.
* `i` (array, required): ordered list of items. Each item has a `t` and fields matching `ton` / `jetton` / `nft` below.

```tsx
type TransactionDraftItem = SendTonItem | SendJettonItem | SendNftItem;

interface SendTonItem {
  t: 'ton';
  a: string; // message destination in user-friendly format
  am: string; // number of nanocoins to send as a decimal string
  p?: string; // raw one-cell BoC encoded in Base64
  si?: string; // raw one-cell BoC encoded in Base64
  ec?: Record<string, string>; // extra currencies to send with the message
}

interface SendJettonItem {
  t: 'jetton';
  ma: string; // jetton master contract address
  qi?: number; // arbitrary request number
  ja: string; // amount of transferring jettons in elementary units
  d: string; // address of the new owner of the jettons
  am?: string; // optional amount of TON (nanocoins) to attach for fees. If omitted, wallet MUST calculate this field
  rd?: string; // address where to send a response with confirmation of a successful transfer and the rest of the incoming message Toncoins. If omitted, user's address MUST be used
  cp?: string; // raw one-cell BoC encoded in Base64 custom data (used by either sender or receiver jetton wallet for inner logic)
  fta?: string; // amount of nanotons to be sent to the destination address
  fp?: string; // Base64-encoded custom data that should be sent to the destination address with notification
}

interface SendNftItem {
  t: 'nft';
  na: string; // address of the NFT item to transfer
  qi?: number; // arbitrary request number
  no: string; // address of the new owner of the NFT item
  am?: string; // optional amount of TON (nanocoins) to attach for fees. If omitted, wallet MUST calculate this field
  rd?: string; // address where to send a response with confirmation of a successful transfer and the rest of the incoming message Toncoins. If omitted, user's address MUST be used
  cp?: string; // raw one-cell BoC encoded in Base64 custom data (used by either sender or receiver NFT wallet for inner logic)
  fta?: string; // amount of nanotons to be sent to the destination address
  fp?: string; // Base64-encoded custom data that should be sent to the destination address with notification
}
```

**Examples:**

Request:

```json5
{
  "id": "123",
  "method": "txDraft",
  "params": {
    "vu": 1764424242,
    "n": "-239",
    "i": [
      {
        "t": "ton",
        "a": "UQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJKZ",
        "am": "20000000"
      }
    ]
  }
}
```

```json5
{
  "id": "124",
  "method": "txDraft",
  "params": {
    "vu": 1764424242,
    "n": "-3",
    "i": [
      {
        "t": "jetton",
        "ma": "EQCxE6mUtQJKFnGfaROTKOt1lZbDiiX1kCixRv7Nw2Id_sDs",
        "ja": "1000000",
        "d": "UQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJKZ",
        "fta": "10000000"
      }
    ]
  }
}
```

The wallet responds with **SendTransactionDraftResponse** (same shape as sendTransaction result):

```tsx
type SendTransactionDraftResponse =
    | SendTransactionDraftResponseSuccess
    | SendTransactionDraftResponseError;

interface SendTransactionDraftResponseSuccess {
    result: string; // Message BoC
    id: string;
}

interface SendTransactionDraftResponseError {
    error: { code: number; message: string };
    id: string;
}
```

**Error codes:**

| code | description                   |
|------|-------------------------------|
| 0    | Unknown error                 |
| 1    | Bad request                   |
| 100  | Unknown app                   |
| 300  | User declined the transaction |
| 400  | Method not supported          |

##### Sign Message Draft

App sends **SignMessageDraftRequest** (or the same structure is used when delivering a draft via [Intent](#intents) URL):

```tsx
interface SignMessageDraftRequest {
  id: string;
  method: 'signMsgDraft';
  params: {
    vu?: number;
    f?: string;
    n?: string;
    i: TransactionDraftItem[];
  };
}
```

Where `params` has the same structure as the transaction draft payload (`vu`, `f`, `n`, `i` — see [Send Transaction Draft](#send-transaction-draft)). The wallet signs the message and returns it as a base64-encoded BoC instead of sending it to the blockchain.

**Examples:**

```json5
{
  "id": "127",
  "method": "signMsgDraft",
  "params": {
    "vu": 1764424242,
    "n": "-239",
    "i": [
      {
        "t": "ton",
        "a": "UQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJKZ",
        "am": "20000000"
      }
    ]
  }
}
```

The wallet responds with **SignMessageDraftResponse**:

```typescript
type SignMessageDraftResponse =
    | SignMessageDraftResponseSuccess
    | SignMessageDraftResponseError;

interface SignMessageDraftResponseError {
    error: { code: number; message: string };
    id: string;
}

interface SignMessageDraftResponseSuccess {
    result: {
        internal_boc: string;  // base64-encoded BoC of the signed internal message
    };
    id: string;
}
```

**Error codes:**

| code | description               |
|------|---------------------------|
| 0    | Unknown error             |
| 1    | Bad request               |
| 100  | Unknown app               |
| 300  | User declined the request |
| 400  | Method not supported      |

##### Action Draft

App sends **ActionDraftRequest** (or the same structure is used when delivering a draft via [Intent](#intents) URL):

```tsx
interface ActionDraftRequest {
  id: string;
  method: 'actionDraft';
  params: {
    url: string;
  };
}
```

Where `params` is an object with:

* `url` (string, required): action URL. The wallet MUST add the `address` query parameter. The action URL MUST respond with a JSON object containing `action_type` and `action` fields — see Action URL Response Examples below.

**Examples:**

```json5
{
  "id": "126",
  "method": "actionDraft",
  "params": {
    "url": "https://example.com/dex?action=swap"
  }
}
```

**Action URL Response Examples:**

For a sendTransaction action:
```json5
{
  "action_type": "sendTransaction",
  "action": {
    "valid_until": 1658253458,
    "network": "-239",
    "messages": [
      {
        "address": "EQBBJBB3HagsujBqVfqeDUPJ0kXjgTPLWPFFffuNXNiJL0aA",
        "amount": "20000000"
      }
    ]
  }
}
```

For a signData action:
```json5
{
  "action_type": "signData",
  "action": {
    "network": "-239",
    "type": "text",
    "text": "Confirm swap"
  }
}
```

The wallet responds with **ActionDraftResponse**:

```tsx
type ActionDraftResponse =
    | ActionDraftResponseSuccess
    | ActionDraftResponseError;

type ActionDraftResponseSuccess = SignDataResponseSuccess | SendTransactionResponseSuccess | SignDataResponseSuccess;

interface ActionDraftResponseError {
    error: { code: number; message: string };
    id: string;
}
```

**Error codes:**

| code | description               |
|------|---------------------------|
| 0    | Unknown error             |
| 1    | Bad request               |
| 100  | Unknown app               |
| 200  | Action URL unreachable    |
| 300  | User declined the action  |
| 400  | Method not supported      |

#### Intents

Intents are deep-link flows that allow a dApp to prepare an action and hand it over to a wallet **without requiring a prior TonConnect session**. There are four types of intents: SendTransactionIntent, SignDataIntent, SignMessageIntent, and SendActionIntent. No fields are shared across intent types; each intent carries only the params for its method (draft or sign data).

**Intent link format**

The URL is built the same as for [connect](bridge.md#universal-link): common params `v`, `id` (optional), `r` (optional ConnectRequest), `ret`, plus intent-specific params.

- **m**: Intent delivery mode.
  - **m=intent**: Method payload is in the URL. Use **mp** (method payload) — draft or sign data encoded as `base64url(json.stringify(payload))`.
  - **m=intent_remote**: Method payload is fetched from object storage. Use **get_url** and **pk** (same as object storage approach below).

**Flow:**

**When m=intent (URL-embedded payload)**

- The app constructs the intent payload (draft or sign data).
- The app encodes it as `base64url(json.stringify(payload))` and sets **mp** in the link.
- Deep link form: `tc://?m=intent&v=2&id=<client_pub_key>&r=<urlsafe(ConnectRequest)>&mp=<base64url(payload)>&ret=back`.
- The wallet extracts the payload from **mp** and processes the intent (sends transaction, signs data, signs message, or action). If **r** is present, the wallet SHOULD complete the connect flow before processing the intent.

**When m=intent_remote (object storage)**

- The app generates its session key pair and a key pair used only to encrypt and decrypt the intent payload stored on the object_storage.
- The app constructs the intent payload and encrypts it with the public key of that pair as described in the [Session protocol](session.md#encryption).
- The app uploads the encrypted payload to the object_storage with TTL.
- The app receives a `get_url` — URL to get the stored intent from the object_storage.
- Deep link form: `tc://?m=intent_remote&v=2&id=<client_pub_key>&r=<urlsafe(ConnectRequest)>&get_url=<get_url_encoded>&pk=<priv_key>&ret=back`, where:
  - **get_url** is the URL-encoded URL to get the stored intent from the object_storage.
  - **pk** is the private key of the encryption key pair (used to encrypt/decrypt the payload on the object_storage), encoded as hex.
- The wallet retrieves the encrypted payload from the object_storage using **get_url**, decrypts it using **pk** as in the [Session protocol](session.md#decryption), then processes the intent (sends transaction, signs data, signs message, or action). If **r** is present, the wallet SHOULD complete the connect flow before processing the intent.

**Note:** The app can choose either approach. Approach 1 (Object Storage) is recommended for larger payloads. Approach 2 (URL-Embedded) is simpler but has URL length limitations.

#### Disconnect operation
When user disconnects the wallet in the dApp, dApp should inform the wallet to help the wallet save resources and delete unnecessary session.
Allows the wallet to update its interface to the disconnected state.

```tsx
interface DisconnectRequest {
	method: 'disconnect';
	params: [];
	id: string;
}
```

Wallet replies with **DisconnectResponse**:

```ts
type DisconnectResponse = DisconnectResponseSuccess | DisconnectResponseError; 

interface DisconnectResponseSuccess {
    id: string;
    result: { };
}

interface DisconnectResponseError {
   error: { code: number; message: string };
   id: string;
}
```

Wallet **shouldn't** emit a 'Disconnect' event if disconnect was initialized by the dApp. 

**Error codes:**

| code | description               |
|------|---------------------------|
| 0    | Unknown error             |
| 1    | Bad request               |
| 100  | Unknown app               |
| 400  | Method not supported      |


### Wallet events

<ins>Disconnect</ins>

The event fires when the user deletes the app in the wallet. The app must react to the event and delete the saved session. If the user disconnects the wallet on the app side, then the event does not fire, and the session information remains in the localstorage

```tsx
interface DisconnectEvent {
	type: "disconnect",
	id: number; // increasing event counter
	payload: { }
}
```

<ins>Connect</ins>
```tsx
type ConnectEventSuccess = {
    event: "connect";
    id: number; // increasing event counter
    payload: {
        items: ConnectItemReply[];
        device: DeviceInfo;
    }
}
type ConnectEventError = {
    event: "connect_error",
    id: number; // increasing event counter
    payload: {
        code: number;
        message: string;
    }
}
```
