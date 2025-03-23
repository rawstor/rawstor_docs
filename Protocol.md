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
- TDB, other commands are not defined yet.

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
- after this request frame, data follows if any (for write requests)

```mermaid
block-beta
  columns 6
  0["uint32_t magic"]:1 a["int commands_t"]:1 aa["u_int16_t cid"] b["u_int64_t offset"]:2 c["u_int64_t len"]:2
  cc["DATA"]:7
```

#### IO requests
- See `proto_resp_frame_t` struct
- `cid` - command ID, used to correlate request and response if many requests issued simultaneously
- `res` - response code, if positive, request was successful (bytes written/read, etc.), negative - error code
- after this response frame, data follows if any (for read requests)

```mermaid
block-beta
  columns 4
  0["uint32_t magic"]:1 a["int commands_t"]:1 aa["u_int16_t cid"]:1 c["u_int64_t: res"]:2
  cc["DATA"]:5
```
