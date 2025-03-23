# DRAFT: Rawstor High-level architecture

```mermaid
classDiagram
    ost <|-- mds
    ost <|-- common_client_lib
    mds <|-- common_client_lib
    mgs <|-- common_client_lib
    common_client_lib <|-- client_gateway_s3
    common_client_lib <|-- client_block_vm
    class ost{
      <<Server type>>
      Aka object storage server
      -Stores parts of objects
      -Object = group of chunks
      -**MAY** have redundancy, but should not
      +create(obj_id, ch_id, size)
      +read(ch_id, offset, size)
      +write(ch_id, offset, size)
      +erase(ch_id, offset, size)
      +delete(ch_id)
    }
    note for ost "Notes:
    chunk id is
    - objid:offset, we don't need anything else (?)
    - + versionid, we need it for snapshots
    The size of a chunk must be a power of two (TODO: size constraints)."

    class mds{
      <<Server type>>
      Aka metadata storage server
      -Stores mapping objid-> [ch_id1, ch_id2...]
      +getObj(objid): [offset1: ost_id:ch_id, offset2: ost_id:ch_id, ...]
      +getObjPart(objid, offset, size): [offset1: ost_id:ch_id, offset2: ost_id:ch_id, ...]
      +getFreeOst(requested_size): ost_list (not all!)
    }
    note for mds "Notes:
    - objid is some unique string (TODO think about sharding of mds and whole clusters)
    Object consists from:
    - some meta (created_at, hash?, project_id)
    - list of offsets and corresponding osts (ostid:offset)"

    class mgs{
      <<Server type>>
      Aka general configuration server (optional)
      -Stores servers roles and addresses (mds, ost)
    }
    class common_client_lib{
      <<client library>>
      Some gateway, for ex, S3
      - any data transformation should be here!
      - **SHOULD** have redundancy via writing to multiple OSTs
      - can read simultaneously from multiple OSTs
      -- compress
      -- encrypt
      -- hashsum?
      +write(?)
      +read(?)
      +flush(?)
      +snapshot(?)
    }
    class client_gateway_s3{
      <<Client example>>
      Some gateway, for ex, S3
      +S3_API()
    }
    class client_block_vm{
      <<Client example>>
      Some block storage client, for ex. VM
      +block_api()
    }
```

(Some terms are borrowed from Lustre; an attentive reader may notice some similarities):
- **\*\*T (example - OST) - *T*arget (shard)**
- **\*\*S (example - OSS) - *S*erver (node) with shards of this role**

Components (roles):
- Object storage target (**OST**)
  - directly responsible for storing data (parts of objects)
  - per disk/partition
  - does not handle redundancy
- Metadata storage target (**MDT**)
  - stores mapping "part of object->OST"
  - in the case of block storage, it is largely static, used mainly for initial allocation
- Management server (**MGS**) (optional)
  - entry point with basic information about SDS (lists of MDS/OSS and so on)
- Client library
  - a library implementing a basic client for SDS
  - higher-level "facades" should be built on top of it, such as the QEMU driver, S3 gateway, and so on
  - what distinguishes rawstor from other solutions is that the responsibility for redundancy and data preparation lies expicitly **on the client**
  - if service operations in SDS (such as recovering redundancy when a server/disk fails) are needed, such logic should be based on the client library with one difference being that it can be run in another mode closer to the data

Entity - object:
- has various properties
  - example - replication factor
- can be sliced across different OSTs (aka Chunks)
  - **IMPORTANT**: key difference of rawstor from other solutions is that you can deside how your data will be slided across OSTs (Chunk size, distribution strategy, etc.)
    - from "local" storage (one whole volume on one OST only as one `chunk`)
    - to "distributed" storage (like `CEPH` - small chunks distributed across whole cluster, although this strategy is a little bit discouraged in Rawstor, we already have Ceph, Vitastor and other Ceph-like storages)
    - best way to look at Rawstor as a "middle ground" between local and distributed storage (DRBD with massive slicing and feature-rich client, or shared-nothing Lustre with redundancy out of the box)
- must have two modes
  - mutable: needed for fast block storage
  - immutable: convenient for object storage
