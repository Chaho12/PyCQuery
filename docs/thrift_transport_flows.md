# Thrift transport flows

---

## Thrift HTTP

In HTTP mode, the client uses `THttpClient` to send requests to `http://host:port/http_path`. There is no SASL handshake; authentication is carried in **HTTP headers** (e.g. Kerberos: `Authorization: Negotiate <token>`, LDAP: Basic).

```mermaid
sequenceDiagram
    participant C as Client (PyCQuery)
    participant H as THttpClient
    participant S as Server (HiveServer2 HTTP)

    Note over C,S: 1. Create connection (service_mode=http)
    C->>H: THttpClient(url)
    alt auth is KERBEROS
        C->>C: set_auth_setting()
        C->>C: keytab or ccache to TGS, then AP-REQ
        C->>H: setCustomHeaders(Authorization Negotiate + base64)
    else auth is LDAP etc
        C->>H: setCustomHeaders Basic auth
    end
    opt timeout
        C->>H: setTimeout(ms)
    end

    Note over C,S: 2. transport.open (TCP only, no SASL)
    C->>H: open()
    H->>S: TCP connect

    Note over C,S: 3. Thrift RPC as HTTP POST, body = Thrift payload
    C->>H: write OpenSession etc
    H->>S: POST with Auth header, body Thrift payload
    S-->>H: 200 OK, body Thrift response
    H-->>C: read()
```

---

## Thrift Binary + Kerberos SASL

In Binary mode with Kerberos, a **SASL GSSAPI handshake** runs over TCP first; afterward, traffic is sent as 4-byte length + Thrift payload frames.

### Sequence diagram

```mermaid
sequenceDiagram
    participant C as Client (PyCQuery)
    participant T as TKerberosSaslTransport
    participant S as Server (Kyuubi/HiveServer2)

    Note over C,S: 1. Create connection (service_mode=binary, auth=KERBEROS)
    C->>T: wrap TSocket with TBufferedTransport
    C->>T: token_factory = _get_binary_kerberos_sasl_response
    opt timeout
        C->>T: socket.setTimeout(ms)
    end

    Note over C,S: 2. transport.open and SASL handshake
    C->>T: open()
    T->>T: token_factory(None) → AP-REQ
    T->>S: START + "GSSAPI"
    T->>S: OK + AP-REQ (initial token)

    loop challenge/response
        S-->>T: status, payload (OK or COMPLETE)
        alt status == COMPLETE
            T->>T: break (handshake done)
        else status == OK
            T->>T: token_factory(payload) → reply
            alt payload == 32 bytes, 05 04 (GSS wrap)
                T->>T: GSS_WrapNoConf(4 octets) → 32-byte token
                T->>T: reply = token, send_complete
                T->>S: COMPLETE + token
                S-->>T: status, msg (OK or COMPLETE)
                T->>T: break
            else other payload
                T->>S: OK + reply
            end
        end
    end

    Note over C,S: 3. Handshake done, Thrift frames (4-byte len + payload)
    C->>T: write(OpenSession etc)
    T->>S: 4-byte length + Thrift payload
    S-->>T: 4-byte length + response
    T-->>C: read()
```

## Token factory branches (_get_binary_kerberos_sasl_response)

```mermaid
flowchart TD
    A[payload?] -->|None or empty| B[_get_binary_kerberos_token<br/>AP-REQ]
    A -->|len=4| C[4 octets, layer 1, max 0]
    A -->|len=32, 05 04| D[GSS_WrapNoConf → 32 bytes<br/>return token, True]
    A -->|len>=16, 05 04| E[Unwrap then GSS_Wrap<br/>encrypted, return r2+r1]
    A -->|else| F[return b'']

    D --> G[open send COMPLETE + token]
    B --> H[open send OK + token]
    C --> H
    E --> H
    F --> H
    H[open send OK + reply]
```
