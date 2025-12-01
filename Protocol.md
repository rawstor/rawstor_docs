# DRAFT: Rawstor Protocol design

- TCP stateful connection
- Any request will have response frame before actual data transition (see `proto_resp_frame_t`)
- Magic number for protocol frames is `0x72737472 // "rstr" as ascii` (used for consistency and endianness checks)

#### commands:
(see `commands_t` enum)
- CMD_SET_OBJECT
- CMD_READ
- CMD_WRITE
- CMD_DISCARD
- TBD, other commands are not defined yet.

#### Auth + initialization
- First request via connection should be `CMD_SET_OBJECT` command (see `proto_basic_frame_t` struct)
- TBD: Authentication mechanism, cababilities negotiation, etc.

```mermaid
block-beta
  columns 8
  0["uint32_t magic"]:1 a["commands_t = CMD_SET_OBJECT"]:1 b["char[OBJID_LEN] objid"]:2
```

#### IO requests
- See `proto_io_frame_t` struct
- `cid` - command ID, used to correlate request and response if many requests issued simultaneously
- `hash` - hash of request data if any (for write requests), now xxh3 used
- after this request frame, data follows if any (for write requests)

```mermaid
block-beta
  columns 6
  0["uint32_t magic"]:2 a["int commands_t"]:1 aa["uint16_t cid"]:1 b["uint64_t offset"]:4 c["uint32_t len"]:2 d["uint64_t hash"]:4
  cc["DATA"]:4
```

#### IO responses
- See `proto_resp_frame_t` struct
- `cid` - command ID, used to correlate request and response if many requests issued simultaneously
- `res` - response code, if positive, request was successful (bytes written/read, etc.), negative - error code
- `hash` - hash of request data if any (for read responses), now xxh3 used
- after this response frame, data follows if any (for read responses)

```mermaid
block-beta
  columns 4
  0["uint32_t magic"]:2 a["int commands_t"]:1 aa["uint16_t cid"]:1 c["uint64_t: res"]:4 d["uint64_t hash"]:4
  cc["DATA"]:4
```
