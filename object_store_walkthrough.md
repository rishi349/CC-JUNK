# Mini Distributed Cloud Object Store: How It Works

## The big idea

You're building a tiny version of Amazon S3. A user hands the system a file, and the system has to:

1. **Cut it into pieces** (chunks of 1 MB)
2. **Spread the pieces across several machines** (workers)
3. **Keep two copies of each piece** on different machines, so losing a machine loses nothing
4. **Remember where everything is** (the controller's database)
5. **Put the file back together** when asked, and detect if any piece got damaged

Everything else (heartbeats, repair, checksums) exists to make those five things survive failures.

## The three roles

- **Client**: whoever calls the REST API (`curl`, a script). It only knows the controller's address and an object ID.
- **Controller**: the brain. It owns the metadata and makes every decision. It stores no file bytes permanently.
- **Workers (4)**: the muscle. Each one just saves, returns, and deletes chunk files on its own disk.

```
            ┌─────────┐   REST (/api/v1)   ┌──────────────┐
            │ Client  │ ─────────────────► │  Controller  │  metadata DB (SQLite)
            └─────────┘                    └──────┬───────┘
                                   /internal/*    │  (store/fetch chunks, heartbeats)
                       ┌───────────┬──────────────┼───────────┐
                  ┌────▼───┐  ┌────▼───┐     ┌────▼───┐  ┌────▼───┐
                  │worker1 │  │worker2 │     │worker3 │  │worker4 │
                  │ /data  │  │ /data  │     │ /data  │  │ /data  │  (volumes)
                  └────────┘  └────────┘     └────────┘  └────────┘
```

The key separation: **the controller knows *where* things are; workers hold *what* the things are.** If the controller's database is lost, you can't find your data. If one worker is lost, the controller still knows another worker has each chunk.

## Walkthrough: uploading a 3.5 MB file called `photo.jpg`

### Step 1: the client sends it

`POST /api/v1/objects` with the file and optional metadata like `{"owner": "venkat"}`.

### Step 2: the controller cuts it into chunks

It reads the file 1 MB at a time:

```
photo.jpg (3.5 MB)
 ├─ chunk 0: bytes 0 – 1 MB       (1 MB)
 ├─ chunk 1: bytes 1 – 2 MB       (1 MB)
 ├─ chunk 2: bytes 2 – 3 MB       (1 MB)
 └─ chunk 3: bytes 3 – 3.5 MB     (0.5 MB, last chunk is smaller)
```

Each chunk gets an ID that encodes its order, e.g. `a1b2c3_0`, `a1b2c3_1`, where `a1b2c3` is the object ID. For each chunk it also computes a **SHA-256 checksum**, a fingerprint of the bytes. If even one bit changes later, the fingerprint no longer matches. That's how corruption gets detected.

### Step 3: the controller picks two different workers per chunk

It chooses among alive workers, preferring the least loaded. The result might look like:

```
chunk 0 → worker1, worker3
chunk 1 → worker2, worker4
chunk 2 → worker1, worker2
chunk 3 → worker3, worker4
```

Two rules: the two copies are always on *different* workers, and chunks spread across all four so no single worker holds everything.

### Step 4: the controller sends each chunk to its workers

`PUT /internal/chunks/a1b2c3_0` to worker1, and again to worker3. Each worker writes the bytes to a file in its persistent volume (`/data/chunks/a1b2c3_0`). A worker writes to a temp file and then renames it, so a crash mid-write never leaves a half-chunk that looks valid.

### Step 5: only now does the controller record it

After every chunk has landed on two workers, it writes to SQLite in one transaction:

- an `objects` row (name, size, content type, whole-file checksum, number of chunks, timestamps, custom metadata)
- four `chunks` rows (ID, order, size, checksum)
- eight `replicas` rows (chunk → worker)

Then it returns `201` with the object ID. If anything failed earlier, the controller deletes the chunks already stored and returns an error, and the object never appears. A file is either fully there or not there at all.

## Walkthrough: downloading it

1. `GET /api/v1/objects/{id}`. The controller looks up the object and its chunks in order (0, 1, 2, 3).
2. For each chunk it looks up which alive worker holds a copy and fetches it.
3. It **recomputes the SHA-256** and compares it to the stored one.
   - Match: send those bytes to the client and move on to the next chunk.
   - Mismatch or worker unreachable: silently try the other copy.
4. The client receives the chunks in order, so the pieces concatenate into the original file. The client never learns which workers were involved.

Because chunks are streamed one at a time, memory use stays around 1 MB no matter how big the file is.

## How the system knows a worker died

Every 5 seconds each worker sends the controller a **heartbeat** ("I'm alive"). The controller records the time of the last one. A background task checks: if a worker's last heartbeat is older than about 15 seconds, it's marked **dead** and no new chunks go to it.

Now consider `docker stop worker2`:

- **Before:** every chunk has 2 copies. All good.
- **During:** after ~15 s, worker2 is marked dead. Any chunk that had a copy on worker2 now has only 1 alive copy. Downloads still work because the other copy exists. The system *detects* the replication factor is broken (some chunks have fewer than 2 alive replicas).
- **Repair:** a background **repair loop** finds those under-replicated chunks, reads each from its surviving copy, verifies the checksum, and writes it to a different alive worker. Replication is back to 2 using only the three survivors.
- **After `docker start worker2`:** it re-registers. Its disk still holds its old chunks (persistent volume). The controller compares them against the database. Chunks it still needs are fine, and chunks that were deleted or already re-replicated elsewhere are surplus and get cleaned up.

## How corruption is handled

Say someone appends junk to a chunk file on worker1's disk. Nothing notices immediately, since worker1 is still alive and heartbeating. But on the next download that touches the chunk:

1. The controller fetches it from worker1 and computes the SHA-256.
2. It **doesn't match** the stored checksum, so the controller discards the bytes.
3. It fetches the other copy from another worker, which matches, and sends that to the client.
4. It marks worker1's copy as bad (removes that replica row). The repair loop then sees the chunk has one good copy and recreates the second on a healthy worker.

The client always gets correct data, and the system heals itself.

## What lives where (storage layout)

```
CONTROLLER (volume: controller_data)
  └─ metadata.db   ← objects, chunks, replicas, workers tables

WORKER 1 (volume: worker1_data)       WORKER 2 (volume: worker2_data)
  └─ chunks/                            └─ chunks/
       a1b2c3_0                              a1b2c3_1
       a1b2c3_2                              a1b2c3_2
       ...                                   ...
```

Docker volumes live outside the container's lifecycle, so removing and recreating a container doesn't erase its data. That's what "persistent storage" in the spec means.

## Metadata schema (SQLite on the controller)

```sql
objects(object_id PK, name, size, content_type, checksum, num_chunks,
        created_at, modified_at, custom_metadata JSON)
chunks(chunk_id PK, object_id FK, idx, size, checksum)      -- chunk_id = f"{object_id}_{idx}"
replicas(chunk_id FK, worker_id FK, PRIMARY KEY(chunk_id, worker_id))
workers(worker_id PK, url, last_heartbeat, status)           -- status: alive/dead
```

## The mental model in one sentence

The controller is a **bookkeeper** with a database mapping *object → ordered chunks → two workers each*. Workers are **dumb disks** with an HTTP interface. Checksums let you **trust but verify** every read, and heartbeats plus a repair loop make sure the "two copies" promise is **restored** whenever something breaks.

## Suggested build order

1. Worker + controller skeleton: register, heartbeat, `workers` table
2. Upload and download with chunking, replication, and checksum verification
3. Metadata, list, delete, PATCH, and consistent JSON errors
4. Heartbeat timeout marks workers dead
5. Repair loop plus orphan cleanup on re-register
6. Corruption handling
7. Demo script covering every item in Section 10 of the spec
