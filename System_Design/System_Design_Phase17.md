SYSTEM DESIGN STUDY NOTES
PHASE 17: CASE STUDIES (APPLY EVERYTHING) — STANDALONE EDITION

This document walks through 37 real-world system design problems (a few, like Uber/Ola, are genuinely identical designs under different brand names, noted where relevant). Each entry is fully self-contained: every concept used to solve it is explained from scratch, right there, with a simple architecture diagram showing how the pieces fit together — you don't need to have read anything else first. If you're studying for interviews, use each entry the same way: read the core requirement, pause and sketch your own answer and diagram, then read the full walkthrough and compare.


1. TINYURL

Core requirement: given a long URL, generate a short, unique alias (e.g., short.ly/abc123) that redirects to the original URL when visited.

```
                 ┌──────────────┐
 Client ───────▶ │ Load Balancer│
                 └──────┬───────┘
                        ▼
                ┌────────────────┐
                │  App Servers    │
                └───┬────────┬───┘
                    ▼        ▼
             ┌──────────┐ ┌────────────────┐
             │  Cache    │ │ Key-Value Store │
             │ (Redis)   │ │ (short→long URL)│
             └──────────┘ └────────────────┘
                    ▲
           (cache-aside: check cache first,
            fall back to store on a miss)
```

The central challenge is generating short, unique codes reliably at scale, without two different users ever colliding on the same code. There are two common approaches. The first is a counter-based approach: maintain a globally increasing number (1, 2, 3...) and convert each number into a short string using base62 encoding (a-z, A-Z, 0-9, giving 62 possible characters per position — so even a 7-character code gives 62^7, roughly 3.5 trillion possible unique combinations). The challenge with a single global counter is that it can become a bottleneck if every server has to coordinate on "what's the next number" — this is usually solved by handing out counter RANGES to each server in advance (server A gets IDs 1-1,000,000, server B gets 1,000,001-2,000,000, and so on), so each server can generate new IDs locally without talking to anyone else. The second approach is hash-based: compute a hash (a fixed-length scrambled fingerprint) of the original URL, such as MD5, and take the first several characters of that hash as the short code. This is simpler to distribute across many servers since no coordination is needed, but two different URLs can occasionally hash to the same short prefix (a "collision"), requiring a check-and-retry step with a slightly different hash input.

The system is overwhelmingly read-heavy: a URL is shortened once but redirected to potentially millions of times. This means the read path (short code → original URL lookup) deserves the most design attention. A simple key-value store (a database that just maps a unique key to a value, like a dictionary — Redis or DynamoDB are common choices) is a natural fit, since the only real access pattern is "given this exact short code, return the long URL." Because reads vastly outnumber writes, an in-memory cache (a fast, temporary data store, typically Redis, that sits in front of the main database and serves frequently-requested data without hitting the slower database every time) sitting in front of the database absorbs almost all read traffic, and additional read replicas (extra copies of the database that only serve reads, so read traffic doesn't overload the single machine handling writes) can be added if a single cache/database instance isn't enough.

One more decision worth naming: whether the actual HTTP redirect is a 301 (permanent) or 302 (temporary). A 301 tells the visitor's browser "this redirect will never change," and the browser will cache it locally — meaning repeat visits from the same browser never even hit your server again, which is great for load, but means you lose the ability to count every single click. A 302 says "this might change," so the browser asks your server every single time, letting you track click analytics at the cost of more load on your servers.

Example: with "100 million new URLs shortened per month, and a 100:1 ratio of redirects to shortenings," you'd size your ID space generously (7-character base62 codes comfortably cover many years of growth), lean heavily on caching for the read-heavy redirect path, and choose 302 redirects specifically because click tracking is a stated requirement.


2. PASTEBIN

Core requirement: let users paste and share blocks of text (code snippets, logs, notes), each accessible via a unique, shareable URL, optionally with an expiration time and optional syntax highlighting.

```
 Client ─▶ App Server ─▶ Metadata DB (id, expiry, owner, storage key)
                │
                └──────▶ Object Storage (the actual paste content)

 Background job: scans metadata DB for expired entries,
                 deletes the matching object from storage
```

This looks similar to TinyURL on the surface (generate a unique short code, look it up later) but the actual CONTENT being stored is fundamentally different: TinyURL stores a tiny pointer (another URL, maybe 100 bytes); Pastebin stores potentially large blocks of text (megabytes). This difference changes where the data should actually live. Rather than storing the full paste content inside a traditional database row, large content is better kept in object storage — a storage system specifically built for holding large, unstructured "blobs" of data (like a file), accessed by a simple unique key rather than through complex database queries (Amazon S3 is the most common real-world example). The database itself then only stores small metadata records (paste ID, creation time, expiry time, an optional owner ID, and the object storage key pointing to the actual content) — this keeps the database small, fast, and focused purely on lookups, while the actual bulky content lives in a storage system built specifically for handling large files cheaply and reliably.

Expiration is a genuinely first-class feature here, unlike most URL shorteners. This is implemented using a TTL (Time To Live) — a timestamp attached to each paste's metadata specifying when it should stop being accessible — combined with a background cleanup job (a separate process that runs periodically, checks for anything past its expiry, and deletes it) that actually removes the expired content from object storage, since object storage doesn't automatically expire things the way an in-memory cache does.

Popular pastes benefit from the same caching strategy as TinyURL — a fast cache in front of the metadata lookup avoids hitting the database repeatedly for frequently-viewed content. Access control is also a common requirement: pastes might be public (anyone with the link can view), unlisted (accessible only if you have the exact link, not searchable/browsable), or private (require the owner to be logged in) — this connects to authentication (verifying who a user is) and authorization (checking what that verified user is allowed to do) checks on the read endpoint.

Example: the key design decision an interviewer wants to see you recognize is the storage-tier split — small, fast-changing metadata in a database, large and mostly-static content in object storage — plus genuine, working TTL-based expiration, which TinyURL typically doesn't need to worry about at all.


3. DROPBOX / GOOGLE DRIVE

Core requirement: let users upload, store, and sync files and folders across multiple devices, with changes on one device automatically propagating to all of a user's other devices, and support sharing files/folders with other users.

```
 Device A ─┐                     ┌─▶ Metadata DB (files, folders,
           ├─▶ Sync Service ────┤    permissions, chunk hashes)
 Device B ─┘        │            └─▶ Object Storage (chunk data)
                     │
                     ▼
          Pub/Sub "file changed" event
                     │
        ┌────────────┴────────────┐
        ▼                          ▼
  Device A (push notify)    Device B (push notify)
```

The actual file content (potentially very large files) is stored in object storage, exactly as described under Pastebin above — this is a very common, recurring pattern across many systems: store large binary content in object storage, keep small structured metadata (file name, folder location, owner, sharing permissions, version history) in a traditional relational database, since metadata benefits from strong consistency and the ability to run structured queries ("show me everything in this folder," "who has access to this file").

The genuinely interesting technical problem here is efficient syncing. Rather than re-uploading an entire file every time even a tiny part of it changes, files are split into fixed-size chunks (say, 4MB pieces), and each chunk is identified by a hash of its own content (a fixed-length fingerprint computed from the chunk's bytes — if two chunks have identical content, they produce the identical hash). When a file changes, the client recomputes the hashes of all its chunks, compares them against the previously-uploaded chunk hashes, and only uploads the chunks that have actually changed. This same content-hashing trick also enables deduplication: if two completely different users happen to have a byte-identical file (or even just a byte-identical chunk within otherwise different files), the system only needs to physically store that chunk once, since its hash — and therefore its storage location — is the same regardless of who uploaded it.

Propagating changes to other devices uses a publish-subscribe pattern: rather than each device repeatedly asking "has anything changed?" (wasteful and slow to notice changes), the server actively pushes a notification to a user's other currently-connected devices the moment a change happens, typically over a persistent connection (like a WebSocket, a continuously-open two-way connection between a client and server that allows the server to send data at any time, not just in response to a request). A device that was offline at the time simply catches up by requesting anything it missed the next time it reconnects.

Conflict resolution is the hardest remaining piece: if the same file is edited on two different devices while both are offline, and both later come back online, you get a genuine conflict — two different, valid versions of the same file with no obvious way to automatically know which one is "correct." The simplest resolution is to keep both versions (renaming one as a "conflicted copy" and letting the user manually sort it out). Google Drive's documents go a meaningful step further, supporting genuine real-time collaborative editing (multiple people typing into the same document at the same moment) — this requires either Operational Transformation (a technique that mathematically transforms each person's individual edits so they can be applied in a different order on different devices and still converge to the same final result) or CRDTs (Conflict-free Replicated Data Types — data structures specifically designed so that concurrent, independent changes can always be automatically, mathematically merged without conflict, no matter what order they're applied in).

Example: "how do you avoid re-uploading an entire 2GB video file just because the user renamed it" is a good test of whether you've internalized the chunking approach — the answer is that a rename typically only touches metadata (the file's name field in the database), not the actual chunk content at all, so nothing needs to be re-uploaded.


4. WHATSAPP

Core requirement: real-time one-on-one and group text messaging, with message delivery confirmation, read receipts, online/offline presence indicators, and message history — critically, with end-to-end encryption so that not even WhatsApp's own servers can read message content.

```
 User A ─(WebSocket)─▶ Chat Server 1 ─┐
                                        ├─▶ Presence Service (Redis:
 User B ─(WebSocket)─▶ Chat Server 2 ─┘     "user → which server")
                                        │
                                        ▼
                              Message Store (Cassandra)
                          (messages encrypted end-to-end;
                           server only stores ciphertext)
```

Real-time delivery uses WebSockets: each user's app maintains a persistent, always-open connection to one of many chat servers. When User A sends a message to User B, the server handling A's connection needs to figure out which specific server (out of potentially thousands) User B is currently connected to, then forward the message there to be pushed down to B's open connection. This requires a presence/routing layer — a fast lookup service (commonly backed by Redis, an in-memory key-value store known for very fast reads/writes) that maps "this user ID" to "this specific server they're currently connected to," updated every time a user connects or disconnects.

Message delivery needs to be reliable even though networks are unreliable — messages can be lost, delayed, or a server can crash mid-delivery. The standard approach is at-least-once delivery: the system guarantees a message will eventually be delivered, but under certain failure conditions it might be delivered more than once. To make this safe, each message carries a unique ID, and the receiving client deduplicates (silently ignores) any message ID it's already seen and displayed, so a rare duplicate delivery never shows up twice to the user.

Message ordering matters within a single conversation — a reply should never visually appear before the message it's replying to. This is typically achieved by routing all messages belonging to the same conversation through the same specific partition (a subdivision of a larger data stream or database, where everything with the same "key" — here, conversation ID — always lands in the same partition and is processed in the order it arrives, guaranteeing order within that one conversation, even though there's no guaranteed order ACROSS different, unrelated conversations happening elsewhere in the system).

Message storage at WhatsApp's scale needs a database built for very high write volume and simple, predictable access patterns (fetch messages for one conversation, ordered by time) — a wide-column database like Cassandra is a strong fit: unlike a traditional relational database, it's specifically optimized for enormous write throughput and doesn't need to support complex, ad-hoc queries or joins across many, unrelated tables.

End-to-end encryption is WhatsApp's defining, distinguishing feature: each message is encrypted directly on the sender's device using a key that only the recipient's device holds the matching decryption key for — the server that actually stores and forwards the message only ever sees encrypted, unreadable bytes. This has a real, direct architectural consequence: full text search across message history (searching for a specific word across all your past conversations) can't be done on the server at all, since the server can't read the content — it has to happen entirely on the client device instead, which already has access to the decrypted messages locally.

Example: "how does a message from a phone on WiFi reach a phone on cellular data, potentially on the other side of the world, within milliseconds" is really asking you to describe the WebSocket-plus-presence-routing design above, end to end.


5. DISCORD

Core requirement: real-time text messaging organized into large, persistent, publicly-joinable communities ("servers," each containing many separate channels), plus real-time voice and video chat — supporting communities of potentially hundreds of thousands of simultaneous members.

```
 Text path:  Client ─WS─▶ Chat Server ─▶ Pub/Sub topic (per channel)
                                             │
                                    fan-out to every subscribed
                                    member currently connected

 Voice path: Client ─UDP─▶ Voice Relay Server ─UDP─▶ Other participants
             (lossy, low-latency — a late packet is skipped, not waited on)
```

The text-messaging portion largely reuses the same core design as WhatsApp above — WebSocket connections, a presence/routing layer, and a wide-column or document database for message storage — but the fan-out problem (delivering one message to many recipients at once) is much bigger here: a message posted in a channel with 50,000 members needs to be pushed out to a huge number of currently-connected clients simultaneously, which is a natural fit for a publish-subscribe pattern (a messaging pattern where a sender publishes once to a named "topic," and every subscriber currently listening to that topic automatically receives a copy — rather than the sender needing to know or individually message each recipient) with each Discord channel conceptually acting as its own topic.

Voice and video chat is a genuinely separate technical problem from text messaging. It uses UDP (a network transport protocol that prioritizes speed over guaranteed delivery — unlike TCP, which guarantees every packet arrives, in order, UDP simply sends packets and doesn't worry if a few are lost or arrive out of order) because for real-time audio, a dropped or late packet should simply be skipped rather than waited for — a half-second-old audio packet arriving late and forcing everything to pause and catch up would make a call feel far worse than a tiny, barely-noticeable gap in the audio.

Example: the specific thing an interviewer wants to see you name here is why voice specifically uses a different (UDP-based) transport than text messages do — it's the trade-off between guaranteed-but-possibly-delayed delivery (fine for text) versus fast-but-occasionally-lossy delivery (necessary for anything real-time and continuous, like audio).


6. SLACK

Core requirement: team messaging organized into company "workspaces," each containing many channels, with heavy emphasis on searchability across a company's entire message history, plus integrations with external tools.

```
 Client ─▶ Chat Server (WS) ─▶ Message DB
                │
                └────▶ Search Indexer ─▶ Inverted Index
                                          (Elasticsearch)
                        (server CAN read plaintext —
                         no end-to-end encryption by default)
```

Like Discord, this reuses the core chat architecture (WebSockets for real-time delivery, publish-subscribe for channel fan-out, a database built for high write volume). The defining difference from WhatsApp is that Slack messages are NOT end-to-end encrypted by default — the server can read message content — and this is precisely what makes Slack's other headline feature possible: robust, server-side full text search across a company's entire message history. This is implemented using a dedicated search engine (a specialized system built specifically for fast text search across large volumes of content, using a data structure called an inverted index — instead of scanning every message for a search term, the system pre-builds a lookup table mapping each individual word to every message that contains it, so a search simply looks up the word directly rather than scanning everything). Elasticsearch is the most common real-world choice for this kind of search infrastructure.

Workspace-level organization also introduces a genuine multi-tenancy consideration (multiple separate, independent customers — here, companies — sharing the same underlying infrastructure, while keeping each company's data logically and securely separate from every other company's data) — every single query and permission check needs to be correctly scoped to the requesting user's specific workspace, so one company's employees can never accidentally see another company's private messages.

Example: "why can Slack offer full message search but WhatsApp largely can't" is the direct, pointed question this case study is built around — the answer is the encryption trade-off described above, not some difference in search technology itself.


7. ZOOM

Core requirement: real-time, low-latency audio and video conferencing between multiple participants simultaneously, with screen sharing and call recording.

```
 Participant 1 ─┐                        ┌─▶ Participant 2
 Participant 2 ─┼─▶ SFU (media relay) ───┼─▶ Participant 3
 Participant 3 ─┘   (1 stream in,        └─▶ Participant 1
                      N-1 streams out
                      per participant)
                          │
                          ▼
                  Recording pipeline ─▶ Object Storage
```

The core technical challenge is transporting live audio/video with minimal delay. This uses WebRTC (Web Real-Time Communication — a set of open standards and browser-native technology specifically built for peer-to-peer-style real-time audio/video/data transport), running over UDP for the same reason described under Discord above: a slightly-late or dropped video frame should be skipped, not waited for, or the whole call would feel laggy and stuttery.

There are two fundamentally different architectures for handling multiple participants in one call, and the choice between them matters enormously at scale. In a mesh architecture, every single participant sends their own video/audio directly to every other participant — this works fine for a 2-3 person call, but the total number of connections and the bandwidth/CPU cost on each individual device grows quadratically with the number of participants (a 10-person call would require each device to maintain 9 separate outgoing connections, quickly overwhelming a normal home internet connection and a normal laptop's CPU). The alternative, and what virtually all real production systems including Zoom actually use, is an SFU (Selective Forwarding Unit) — each participant sends just ONE outgoing stream to a central server, and that server is responsible for relaying/forwarding each participant's stream to every other participant who needs it. This means the cost on each individual participant's device scales linearly (just one outgoing connection, regardless of how many people are in the call), with the heavier relaying work concentrated on the server infrastructure instead, which can be scaled far more easily than an individual user's home internet connection.

Screen sharing is handled as just another video stream through the same SFU pipeline. Recording works as a separate, asynchronous process — the SFU (or a dedicated recording service) captures the combined media streams and writes them to object storage (the large-file storage system already described under Pastebin/Dropbox above), running independently of the live call itself so recording never adds delay to the actual conversation.

One more genuinely necessary piece: because most users' devices sit behind home or office routers (which hide a device's real network address from the outside internet, a mechanism called NAT — Network Address Translation), establishing a direct connection between two participants' devices is often not straightforward. WebRTC solves this using STUN and TURN servers — a STUN server simply helps a device discover its own public-facing network address so a direct connection can be attempted, while a TURN server acts as a full relay of last resort, forwarding traffic between two devices that genuinely cannot connect to each other directly for any reason.

Example: "why not just have every participant send video directly to every other participant" is the setup question this case study is built around — the expected answer names the SFU architecture specifically and explains the linear-versus-quadratic bandwidth scaling difference.


8. YOUTUBE

Core requirement: allow anyone to upload video content, and stream that content smoothly to a massive, global audience across widely varying devices and network speeds.

```
 Upload ─▶ Object Storage (raw file) ─▶ Message Queue ─▶ Transcoding Workers
                                                              │
                                            (240p, 480p, 720p, 1080p chunks)
                                                              ▼
                                                        Object Storage
                                                              │
                                                              ▼
 Viewer ◀── CDN edge cache ◀── adaptive-bitrate chunk delivery
```

An uploaded video is never served to viewers exactly as uploaded — it goes through a transcoding pipeline: an automated process that converts the original file into multiple different resolutions (240p, 480p, 720p, 1080p, etc.) and formats, so playback can later be adapted to whatever a specific viewer's device and current network speed can actually handle. Because transcoding a large video file can take real time (minutes, sometimes longer for very long videos), this needs to happen asynchronously — the upload itself completes quickly and immediately confirms success to the uploader, while the actual transcoding work is handed off to a message queue (a system that holds a list of pending jobs/tasks for other worker processes to pick up and process independently, decoupling the fast "upload received" step from the slower "processing" step) and processed by a separate pool of transcoding workers in the background.

Playback itself uses adaptive bitrate streaming: the video is split into short chunks (a few seconds each), and the video player continuously, automatically requests whichever specific bitrate/resolution chunk best matches the viewer's currently-measured network speed, silently switching up or down as conditions change during playback — this is why a video might start slightly blurry and then sharpen up, or briefly drop in quality during a network hiccup, rather than simply freezing and buffering.

The actual video files themselves — both the original upload and every transcoded version — are large, unstructured binary blobs, so they belong in object storage, exactly the same storage tier already described in the Pastebin and Dropbox case studies above. Because video is extremely bandwidth-heavy (a huge amount of data per viewer, per minute watched), a CDN (Content Delivery Network — a globally distributed network of caching servers positioned physically close to end users in many different cities/countries, so a viewer's request is served from a nearby location rather than traveling all the way back to one single, central data center) is essential for reducing both latency and the load on the origin storage/servers, since a small number of genuinely popular videos account for a hugely disproportionate share of all views.

Video metadata (title, description, view count, comments) is much smaller, structured data, well suited to a more traditional database, with view counts specifically often cached in a fast in-memory store since they update extremely frequently and don't need perfectly precise, instantaneous accuracy on every single view.

Example: the two sub-problems an interviewer specifically wants recognized here are (1) the asynchronous transcoding pipeline (upload triggers a background job, it doesn't make the uploader wait) and (2) CDN-backed adaptive delivery — naming and explaining both is usually sufficient to demonstrate the core of this design.


9. NETFLIX

Core requirement: stream a large, licensed catalog of movies and TV shows to a global subscriber base, with heavy emphasis on personalized content discovery/recommendation.

```
 Same pipeline shape as YouTube (transcode → object storage → CDN),
 plus a Recommendation Service consulted on every homepage load:

 Client ─▶ Homepage Service ──▶ Recommendation Service
               │                  (scores catalog per user,
               │                   using past viewing/ratings)
               ▼
        CDN-cached video chunks (pre-positioned at ISP edges
        ahead of anticipated regional demand)
```

Architecturally, this shares the same foundation as YouTube — transcoding pipeline, object storage, adaptive bitrate streaming, CDN delivery — but two things distinguish it meaningfully. First, Netflix's content catalog is centrally curated and licensed (a relatively small, fixed set of titles, added deliberately by the company) rather than user-generated at YouTube's uncontrolled, continuous scale — this means the upload/ingestion pipeline is far less of a bottleneck, and much more engineering effort goes into the viewing/discovery experience instead. Second, recommendation is arguably the single most important part of the entire product: Netflix's recommendation engine analyzes what a user (and users with similar taste) have watched, rated, and how long they watched before stopping, to decide what to prominently surface on the homepage — this is a machine learning system in its own right, distinct from the video-delivery infrastructure, that continuously scores and ranks content specifically for each individual user.

Netflix is also known for proactively pre-positioning popular content at CDN edge locations (physically installing servers with cached copies of trending/anticipated-to-be-popular content directly inside ISP data centers, ahead of actual demand) rather than purely reactively caching content only after it's first requested — since they can often predict, with reasonable confidence, which content will be popular in which regions ahead of time.

Example: "why would Netflix's engineering priorities differ from YouTube's, even though both are video streaming platforms" tests whether you distinguish the user-generated-content-at-massive-scale problem (YouTube's defining challenge) from the personalized-discovery-of-a-curated-catalog problem (Netflix's defining challenge).


10. SPOTIFY

Core requirement: stream audio content (songs and podcasts) to users, support offline downloads for subscribers, and provide personalized playlists and recommendations.

```
 Client ─▶ App Server ─▶ Object Storage (whole audio files,
              │                cached in full — no chunked
              │                adaptive bitrate needed, files
              │                are small enough already)
              ▼
     Offline download: encrypted local copy + DRM license check
              │
              ▼
     Play event ─▶ Event log ─▶ Royalty/analytics pipeline
```

Despite being another "streaming" system, audio is dramatically smaller than video — a full high-quality song might be only a few megabytes, compared to a video's gigabytes — and this genuinely changes the infrastructure approach. Rather than YouTube/Netflix's chunk-by-chunk adaptive bitrate delivery (necessary because video files are far too large to reasonably cache or download whole), Spotify can often simply cache and deliver entire audio files as complete units, since even a full song is small enough to download and cache in full without meaningfully straining bandwidth or storage.

Offline downloads (letting a subscriber download songs for playback without an internet connection) require storing an encrypted copy of the audio file locally on the user's device, combined with DRM (Digital Rights Management — technical controls specifically designed to enforce a content license, e.g., ensuring downloaded songs stop being playable once a user's subscription lapses) logic that periodically re-validates the user's subscription status even while technically offline.

Recommendation (Spotify's well-known "Discover Weekly" playlist feature is a real-world example of a collaborative filtering recommendation system — an approach that recommends items based on what OTHER users with similar taste have also enjoyed, rather than purely analyzing the attributes of the songs themselves) is a major part of the product, alongside licensing/royalty tracking: every single play needs to be accurately, durably logged, since these logs directly determine how much artists and rights-holders get paid — effectively a very high-volume event-logging pipeline running underneath the visible, user-facing product.

Example: the meaningful delta from YouTube/Netflix an interviewer wants recognized here is specifically the file-size difference (changing the caching/delivery approach) and the DRM-for-offline-playback requirement, which video streaming platforms handle somewhat differently.
11. INSTAGRAM

Core requirement: let users post photos and videos, follow other users, and see a continuously updating, personalized feed of content from accounts they follow.

```
 Post created ──▶ Object Storage (photo/video) + CDN
        │
        ▼
 Normal account?  ──yes──▶ Fan-out on write: push post ID
        │                   into every follower's cached feed
        no (celebrity)
        │
        ▼
 Fan-out on read: stored once, merged into
 followers' feeds live at load time

 Feed load: fetch pre-computed feed (Redis) + merge in any
 celebrity accounts' recent posts fetched live, then rank
```

The central architectural decision for any feed-based system is how the feed is actually assembled — this is called the fan-out problem. In fan-out on write, the moment a user posts something, the system immediately pushes that post into the pre-computed feed of every single one of their followers (typically stored in a fast in-memory cache) — this makes reading a feed extremely fast later (the feed is already assembled, just fetch it), but makes posting expensive for accounts with huge follower counts, since one post might need to be written into millions of individual followers' feeds all at once. In fan-out on read, nothing happens at post time beyond saving the post itself — instead, a user's feed is assembled on the fly, at the moment they actually open the app, by fetching and merging recent posts from everyone they follow — this keeps posting cheap regardless of follower count, but makes every single feed load more expensive, since it requires gathering and merging content from potentially hundreds of different followed accounts in real time.

Most large real systems use a hybrid: fan-out on write for the vast majority of normal users (whose follower counts are modest, so pushing their posts out immediately is cheap), but fan-out on read specifically for celebrity/high-follower accounts (avoiding the "write storm" of instantly updating millions of individual feeds for one single post) — a user's final feed is then assembled by merging their pre-computed feed (from normal accounts they follow) with a live, on-the-fly fetch of any celebrity accounts they follow.

Since Instagram is fundamentally a photo/video platform, it also directly needs the same object storage plus CDN delivery approach already described under YouTube — images and videos are large binary content, best kept out of the main structured database and served from storage built specifically for that purpose, cached at edge locations close to viewers.

Feeds are also typically ranked rather than shown in pure chronological order — a ranking algorithm scores each candidate post by predicted relevance (factors like how recently it was posted, how much engagement it's already received, and how close your relationship is to the poster) and orders the feed by that score rather than simply by timestamp.

Example: "how would you handle a celebrity account with 100 million followers posting something" is almost always asked specifically to test whether you know to reach for the hybrid fan-out approach, rather than naively fanning out every single post to every single follower's feed unconditionally, regardless of follower count.


12. FACEBOOK NEWS FEED

Core requirement: essentially identical to Instagram above — a personalized, ranked, continuously updating feed of content from friends/pages a user follows, just centered on text/link posts and Facebook's specific social graph rather than primarily photos.

```
 Same hybrid fan-out-on-write / fan-out-on-read design as Instagram,
 just without the heavy media-storage/CDN emphasis — content here
 is more varied (text, links, photos, video all mixed together)
```

This is architecturally the same system as Instagram's feed (topic 11) — the same fan-out-on-write-vs-read hybrid, the same ranking-by-relevance approach — without Instagram's especially heavy emphasis on media storage and delivery. If asked this specific question in an interview, the expected answer is essentially the Instagram feed design, adjusted for the fact that content here is more varied (text, links, photos, videos, all mixed together) rather than overwhelmingly photo/video-centric.


13. X (TWITTER)

Core requirement: like Instagram/Facebook, a personalized feed of posts from accounts a user follows — but with much heavier emphasis on real-time delivery (a post should reach followers within seconds) and on surfacing globally "trending" topics.

```
 Tweet ──▶ Pub/Sub (immediate fan-out, seconds not minutes)
                       │
                       ▼
              Followers' cached feeds

 Trending: every tweet ──▶ Stream Processor (sliding-window
                             hashtag/word counter) ──▶ live
                             "Trending Now" leaderboard
```

The core feed architecture is the same hybrid fan-out approach described under Instagram above. What genuinely distinguishes this system is the real-time delivery expectation — unlike, say, a YouTube recommendation homepage (which can tolerate being computed once overnight and staying essentially static all day), a tweet needs to actually reach followers' feeds within seconds of being posted, which pushes harder on the "fan-out on write" side of the hybrid actually completing quickly, using a publish-subscribe messaging system to push new posts out immediately as they're created, rather than relying on any kind of batch/scheduled update process.

"What's trending right now" is a genuinely separate technical problem: a real-time streaming aggregation system continuously counts how often different words/hashtags/topics are being mentioned across the entire platform within a recent, sliding time window (e.g., the last hour), using a stream processing system (software specifically designed to continuously process a never-ending flow of incoming events and maintain running, up-to-the-second aggregate counts/statistics, rather than periodically running one big batch calculation over a fixed, already-collected dataset) — this needs to be current, essentially, to within seconds or minutes, not something that can be recomputed just once a day.

Example: the specific delta interviewers want recognized here versus the general feed design is this real-time delivery/trending requirement, which pushes the design meaningfully harder toward true streaming infrastructure than a system like YouTube's recommendation homepage, which can comfortably tolerate being somewhat stale.


14. REDDIT

Core requirement: support many independent communities (subreddits), where users post content and leave comments in nested, threaded discussions, with both posts and comments ranked by a score combining vote count and recency.

```
 Post votes ──▶ Redis atomic counter ──(periodic flush)──▶ Durable DB
                                                                │
 Ranking = f(net_score, log(age))  ◀────────────────────────────┘
 (early votes weigh more than later ones — new posts get a fair
  chance to rise instead of old posts permanently dominating)

 Comment tree: comment → parent_id, precomputed/cached as one
 nested structure for popular, high-traffic posts
```

Unlike the linear, follow-based feeds of Instagram/Twitter, Reddit's defining technical challenge is its ranking algorithm. Reddit's actual "hot" ranking formula famously combines a post's net vote score with its age using a logarithmic scale (a mathematical scale where each additional unit represents a much larger, exponentially growing real difference — think of how each step up the Richter earthquake scale represents 10x more energy, not just one more unit) — the specific effect of this is that early votes on a post matter meaningfully more, per vote, than an equivalent number of later votes, which gives brand new posts a genuine, fair chance to rise to the top, rather than posts that have simply existed longer (and so have had more total time to accumulate votes) permanently, unfairly dominating the rankings.

Comments form deeply nested trees (a reply to a reply to a reply, potentially many levels deep). At small scale this is naturally modeled with each comment simply storing a reference to its parent comment; at genuine scale, rendering a popular post's entire, deeply-nested comment tree by running many small individual database queries on every page load would be far too slow, so the full nested structure for popular posts is typically precomputed and cached as one single, ready-to-render structure, refreshed periodically or incrementally as new comments arrive, rather than reconstructed live from scratch on every single view.

Vote counting itself needs to handle very high, bursty volume on popular posts — rather than having every single individual vote directly update the primary database (which would be slow and create contention under heavy simultaneous voting), votes are typically first recorded as fast, atomic increments in an in-memory store, with the results periodically, efficiently batched and flushed to the durable underlying database.

Example: being able to specifically name and explain the time-decay ranking formula — why raw vote count alone is an insufficient ranking signal, and what problem the logarithmic time-decay specifically solves — is usually the single most memorable, distinguishing signal in this particular case study.


15. UBER / OLA

Core requirement: match riders requesting a trip with nearby available drivers in real time, track the trip as it happens, and calculate dynamic ("surge") pricing based on the current local balance of supply and demand.

```
 Driver app ─(location every few sec)─▶ Kafka ─▶ Geo-Index (geohash/quadtree)
                                                        │
 Rider requests trip ───────────────────────────────────┤
                                                        ▼
                                          Matching Service (nearby
                                          candidates → best driver)
                                                        │
                                                        ▼
                              Saga: match → payment → confirm
                              (each step has a compensating
                               "undo" action if a later step fails)
```

The foundational technical challenge is geospatial indexing: efficiently answering "which drivers are currently near this specific rider" without having to compare the rider's location against every single driver in the entire system (which would be far too slow at scale). Two common techniques solve this. Geohashing encodes a 2D latitude/longitude coordinate into a single short text string, specifically designed so that nearby physical locations produce strings sharing a common prefix — this means "find nearby drivers" can become a fast, simple prefix-matching lookup rather than a slow geometric distance calculation across every driver. Quadtrees instead recursively divide the map into a tree of ever-smaller rectangular regions (subdividing further wherever driver density is high), letting you quickly narrow down to just the small specific region containing the rider, and search only within it.

Drivers' apps continuously send location updates (every few seconds) — at scale, this is an enormous, continuous stream of incoming writes, well suited to a message queue or streaming platform (like Kafka, a system specifically built for durably, reliably handling extremely high-volume streams of incoming events) that feeds both the live driver-matching system and any downstream analytics.

Matching takes the rider's request, finds a set of nearby candidate drivers via the geospatial index, and scores them by factors like distance, driver rating, and current availability, to actually assign one specific driver — this needs low latency, since a rider is actively, impatiently waiting.

Surge pricing is fundamentally a real-time analytics problem: continuously calculating the ratio of active ride requests to currently available drivers within small geographic regions, and adjusting prices upward in areas where demand is clearly, currently outstripping supply.

A trip moves through a clear sequence of states (requested → matched → driver en route → in progress → completed), and this whole multi-step flow — matching a driver, calculating the fare, processing payment — spans multiple independent parts of the system. This kind of multi-step process, where several separate steps each need to succeed together (or be safely, individually undone if a later step fails) is commonly handled with the Saga pattern: rather than trying to make the entire multi-step flow one single, giant atomic operation (which is impractical once it spans multiple independent services), each step is executed and committed individually, with a corresponding "compensating" action defined for each step to undo it specifically if something later in the sequence fails (e.g., if payment ultimately fails after a driver was already matched and dispatched, a compensating action cancels/releases that driver assignment).

Example: interviewers most often focus on the geospatial matching sub-problem specifically — being able to name geohashing or quadtrees, and explain concretely why comparing a rider's location against every single driver one by one simply doesn't scale, is usually the key signal being tested.

Note: Ola is architecturally the identical system to Uber, operating in a different market — there's no meaningful design difference between the two beyond branding and regional specifics.


16. SWIGGY / ZOMATO

Core requirement: connect three distinct groups of participants — customers, restaurants, and delivery drivers — where customers browse and order from nearby restaurants, restaurants prepare the food, and drivers pick up and deliver it, with live status tracking throughout.

```
 Customer ─▶ Restaurant Service (menu/availability) + Search
                    │
                    ▼
             Order Service ──Saga──▶ Payment Service
                    │
                    ▼
             Delivery Service ──▶ nearby drivers (geo-index,
                                    same technique as Uber)
                    │
                    ▼
       Live status pushed to customer (WebSocket / SSE)
```

Unlike Uber's two-sided model (just riders and drivers), this is a genuine three-sided marketplace, which naturally suggests splitting the system into independent services, each owning one part of the problem: a Restaurant Service (menus, current availability/open-closed status), an Order Service (the order's overall lifecycle), and a Delivery Service (driver assignment and live tracking) — these separate services communicate with each other by publishing and reacting to events (a design where services announce "something happened" rather than directly, tightly calling into each other, which keeps them independently deployable and resilient to each other's temporary failures).

Restaurant discovery reuses the exact same geospatial indexing techniques described under Uber above ("restaurants near this customer"), typically combined with a dedicated text search system for searching by cuisine type, specific dish name, or restaurant name.

The full order lifecycle — place order → restaurant confirms and starts preparing → payment processed → driver assigned → picked up → delivered — spans multiple independent services and is a strong, natural fit for the same Saga pattern described under Uber above, with compensating actions defined for failure cases (e.g., automatically refunding the customer if a restaurant is unable to actually fulfill the order after all).

Driver assignment here is somewhat more complex than Uber's rider-driver matching, since a delivery driver's route needs to account for BOTH the restaurant's location (pickup) and the customer's location (drop-off) — and at scale, efficient systems often batch multiple nearby orders onto a single driver's route, rather than assigning one driver per single order.

Customers expect live status updates ("preparing," "out for delivery," "arriving in 5 minutes") — this is implemented using a persistent, real-time connection (WebSockets, or Server-Sent Events, a simpler one-directional variant specifically for the server continuously pushing updates to the client) that pushes each state change to the customer's app the moment it happens, rather than requiring the app to repeatedly ask "any updates yet?"

Example: the distinguishing feature versus a plain ride-sharing design is this three-sided coordination and the correspondingly more complex order-lifecycle Saga, spanning restaurant, payment, and delivery all together as one coordinated flow.

Note: Zomato is architecturally the identical system to Swiggy, operating in the same and overlapping markets — no meaningful design difference beyond branding.


17. AMAZON

Core requirement: a large-scale e-commerce platform — a browsable, searchable product catalog, a shopping cart, checkout, inventory management, and order fulfillment tracking, all at massive, global scale.

```
 Client ─▶ Search Service (inverted index, faceted filters)
       ─▶ Cart Service (key-value store, persists across devices)
       ─▶ Checkout ──Saga──▶ Inventory (atomic check-and-decrement)
                          ──▶ Payment
                          ──▶ Shipping/Logistics dispatch
```

Product catalog and search directly need a dedicated text search system (the same inverted-index-based search technology described under Slack above), supporting faceted search — letting a customer filter results by category, price range, brand, and rating all simultaneously, alongside a free-text search query.

The genuinely hard, defining sub-problem here is inventory management: the system must never sell more of an item than actually exists in stock, even with many different customers simultaneously trying to buy the last few remaining units of a popular item at the exact same moment. Simply reading the current stock count, checking if it's greater than zero, and then separately decrementing it is unsafe — two concurrent checkout attempts could both read "1 remaining" before either one has actually decremented it, resulting in both being allowed to purchase the same single last unit. The correct fix is to make the check-and-decrement one single, indivisible ("atomic") database operation — decrement the count and simultaneously verify the result never goes below zero as one combined step, so a second, concurrent attempt correctly, safely fails rather than both succeeding.

The shopping cart needs to persist across sessions and devices (add something on your phone, see it later on your laptop) — a fast key-value store is a natural fit here, similar to how session/login state is typically stored, just representing cart contents instead.

Checkout is, once again, a multi-step process spanning several independent concerns (reserve inventory, process payment, dispatch to shipping/logistics) and is handled with the same Saga pattern already described under Uber and Swiggy above — with a compensating action to release the reserved inventory if payment ultimately fails.

Example: "how do you prevent overselling the very last unit of a popular item during a flash sale" is the near-universal specific follow-up question here — the expected answer centers on that atomic check-and-decrement operation, not a naive, separate "check stock, then decrement" pair of steps, which contains a race condition (a bug where the correctness of the result depends on the precise, unpredictable timing of multiple things happening at once).


18. FLIPKART

Architecturally identical to Amazon (topic 17) — the same catalog/search, inventory, cart, and checkout-as-a-Saga design (see the diagram under topic 17), operating in a different regional market. No meaningful architectural delta beyond branding and specific local logistics/payment integrations.


19. GOOGLE MAPS

Core requirement: render interactive maps, support searching for locations/businesses, and calculate the fastest or shortest route between two points, taking real-time traffic conditions into account.

```
 Map tiles: pre-rendered per zoom level ──▶ CDN cache ──▶ Client

 Routing: roads/intersections modeled as a Graph
          edge weight = current travel time (updates live from
          anonymized phone speed/location data streamed in)
          Dijkstra / A* finds shortest path over that live graph

 Search "coffee near me": geo-index (geohash/quadtree) + text search
```

Map data itself is pre-rendered into small "tiles" — individually cacheable image or vector files, each covering one small rectangular geographic area at one specific zoom level. Because map tiles for popular areas are requested extremely frequently and change relatively rarely, they're heavily cached at CDN edge locations (the same geographically-distributed caching infrastructure already described under YouTube above), so most requests for common map areas never even need to reach a central server at all.

Route calculation is the genuinely hard, defining sub-problem. Roads and intersections are modeled as a graph — a data structure made up of nodes (intersections) connected by edges (individual road segments), where each edge has a "weight" representing how long it currently takes to traverse that specific segment. Finding the fastest route between two points is then a shortest-path problem, solved using algorithms like Dijkstra's algorithm (which explores outward from the starting point, always expanding the currently-cheapest known path first, guaranteeing it finds the genuinely shortest overall path) or, more commonly at real map scale, A* (pronounced "A-star" — a more efficient variant of the same basic idea that additionally uses a smart estimate, or "heuristic," of how far a given point is from the destination, letting it intelligently avoid wastefully exploring directions that are obviously heading the wrong way).

Critically, real-time traffic needs to continuously update those edge weights (how long a given road segment currently, actually takes to drive), rather than treating road distances as fixed values — this data comes from aggregating anonymized location/speed data from other users' phones currently on the road, itself a high-volume, continuous streaming data ingestion problem, feeding a constantly, live-updating version of the underlying road graph.

Location search (finding "coffee shops near me") reuses the same geospatial indexing techniques (geohashing/quadtrees) already described under Uber above, combined with a text search system for matching business names and categories.

Example: the specific technical depth expected here is naming a real shortest-path algorithm (Dijkstra or A*) by name and specifically explaining how real-time traffic data continuously feeds into and updates the graph's edge weights, rather than treating the map as a fixed, static, unchanging graph.


20. PAYMENT GATEWAY (STRIPE/PAYPAL-STYLE)

Core requirement: securely process payments between customers and merchants — handling credit card and bank transactions — while guaranteeing money is never lost, duplicated, or double-charged, and integrating correctly with external banking/card networks.

```
 Client ─▶ Payment API (checks idempotency key first)
                │
                ▼
        DB transaction: record payment + outbox event ── atomic,
                                                            same commit
                │
                ▼
       Background publisher ──▶ notifies fraud/accounting/notifications
                │
                ▼
       Saga: authorize → capture → settle (each step reversible)
```

Unlike most of the systems covered so far, this is a system where strong, immediate consistency is genuinely non-negotiable, not a trade-off to be weighed. Every operation touching an account balance or a transaction record needs the classic ACID guarantees of a traditional database transaction: Atomicity (an operation either completes entirely or has no effect at all — never partially applied), Consistency (the operation always leaves the data in a valid, correct state, respecting all defined rules), Isolation (concurrent operations don't interfere with or corrupt each other), and Durability (once a transaction is confirmed successful, it's guaranteed to survive even a subsequent crash).

Idempotency is critical here: because networks are unreliable, a client might send a payment request, experience a timeout without knowing whether it actually succeeded, and reasonably retry — the system must guarantee that retrying the exact same payment request never results in a double charge. This is achieved by having the client generate a unique "idempotency key" for each distinct payment attempt; the server checks whether it has already, successfully processed that exact key before doing anything, and if so, simply returns the original result again rather than processing the payment a second time.

A single payment typically touches multiple separate systems (the merchant's own internal ledger, the card network, the receiving bank) — this multi-step flow, spanning genuinely separate systems, is again a natural fit for the Saga pattern already described under several earlier case studies, with careful compensating actions (e.g., reversing an internal ledger entry if a downstream bank call ultimately fails partway through).

A related, important technique here is the Outbox pattern: recording a transaction in your own database AND reliably notifying other downstream systems (fraud detection, customer notifications, accounting) about it need to genuinely happen together, atomically — but a database write and a separate message being published to other systems are two genuinely independent operations that could each succeed or fail independently of each other, risking the two ending up out of sync. The Outbox pattern solves this by writing both the actual data change AND a record of the "event to be published later" into the exact same single database transaction (guaranteeing they succeed or fail together, as one unit), with a separate background process then reliably reading these pending "outbox" entries and actually publishing them onward, retrying as needed until it succeeds — since it's simply reading already-safely-saved rows from the database, not something fragile that only exists briefly in memory.

Security is paramount: all data is encrypted both at rest (while stored) and in transit (while being transmitted over the network), any API keys or credentials the system itself needs are kept in a dedicated secrets management system (rather than hardcoded anywhere in source code, which could be leaked), and compliance standards like PCI-DSS typically require that raw card numbers are never actually stored directly at all — instead, a specialized third-party processor handles the truly sensitive card details and returns a safe, non-sensitive "token" that the rest of the system can use instead.

Example: "which consistency model would you choose here" has one of the clearest, least-debatable answers of any system in this entire set — strong consistency, full stop — and the interviewer specifically wants to hear you justify why (real money, real regulatory consequences) rather than defaulting to a "favor availability" instinct that might genuinely fit many other systems in this list.
21. BANKING SYSTEM

Core requirement: manage customer accounts and balances, process deposits, withdrawals, and transfers, with absolute correctness (money must never be created or destroyed by a bug) and a complete, permanent, auditable record of every single transaction.

```
 Transfer request
        │
        ▼
 Single ACID transaction:
   debit Account A  ─┐
   credit Account B ─┴─ both succeed together, or neither happens

 Event log (append-only): AccountOpened, Deposited($100),
 Withdrew($30) ... current balance = sum of all events replayed
 (self-verifying: total debits must always equal total credits)
```

The standard, correct approach to representing money movements is double-entry bookkeeping, a genuinely foundational accounting concept: every single transaction is recorded as at least two balanced entries — a debit to one account and a matching credit to another, of the exact same amount. This structure is self-verifying: at any point, the total sum of all debits across the entire system must exactly equal the total sum of all credits — if it ever doesn't, you know immediately, mathematically, that something has gone wrong somewhere, which is a far stronger, built-in correctness guarantee than simply updating a single "current balance" number directly and hoping nothing ever corrupts it.

A particularly strong architectural fit here is event sourcing: instead of storing just an account's current balance, the system stores the complete, ordered, permanent sequence of every event that ever happened to that account (AccountOpened, Deposited $100, Withdrew $30, Deposited $50...) — the current balance is then simply calculated, whenever needed, by replaying and summing up that entire event history. This directly, naturally provides the complete, permanent audit trail that banking regulation genuinely requires, essentially as a free side effect of how the data is structured, rather than needing to be bolted on separately as an afterthought.

Exactly as with the Payment Gateway case study above, strong consistency and full ACID transactions are non-negotiable — a transfer between two accounts must be genuinely atomic, meaning both the debit from one account and the credit to the other succeed together as one indivisible unit, or neither happens at all. If both accounts happen to live within the same single database, a normal database transaction handles this directly. If they live in genuinely separate systems (e.g., different banks, or different internal systems), a full distributed transaction approach is needed — either the more heavyweight Two-Phase Commit protocol (where a central coordinator first asks every involved system "are you ready to commit," and only tells everyone to actually commit once every single one has confirmed "yes" — providing strong correctness at the cost of everything staying blocked and waiting if any one participant is slow or fails) or the Saga pattern already described in earlier case studies (a series of individually-committed local steps, each with a defined compensating action to undo it if something later fails).

Example: "how do you guarantee a transfer between two accounts never loses or duplicates money, even if the system crashes exactly halfway through" is the defining question for this case study — the expected answer centers on atomic, all-or-nothing transaction guarantees at the database level, combined with double-entry bookkeeping as a structural, mathematically self-verifying correctness check layered on top.


22. STOCK TRADING PLATFORM

Core requirement: allow users to place buy and sell orders for financial securities, automatically matching compatible buy and sell orders together, at extremely high throughput and extremely low, consistent latency.

```
 Buy orders  ──▶  Order Book for Security X  ◀── Sell orders
                (single-threaded, in-memory,
                 one matching engine per security —
                 no cross-machine coordination in this hot path)
                        │
                        ▼
              Match found ──▶ durable event log (for audit +
                                crash recovery of order book state)

 Security Y, Z, ... each get their OWN independent matching
 engine running in parallel — scaling is across securities,
 not within one security's own order book
```

This is one of the most latency-sensitive systems in this entire set — the core order-matching logic often needs to process orders within microseconds (millionths of a second), not merely milliseconds, because even tiny latency differences between competing traders can be genuinely, financially exploited by other market participants (a well-known real-world phenomenon called latency arbitrage — profiting specifically from being even slightly faster than everyone else).

The order matching engine is the defining sub-problem. Incoming buy and sell orders are organized into an "order book" (buy orders sorted from highest to lowest price, sell orders sorted from lowest to highest price), and the engine continuously looks for compatible matches (a buy order's price meeting or exceeding a sell order's price, meaning both sides are willing to trade). Because of the extreme latency requirement, this matching logic is very often deliberately implemented as a single-threaded, entirely in-memory process, handling one specific security at a time — this might sound like it goes against typical "scale by distributing across many machines" advice, but that's precisely the point: coordinating between multiple separate machines (via locks or network calls) would itself introduce exactly the kind of latency this system is specifically trying to avoid in its single hottest, most critical path. Every order event is still durably logged (using the same event-sourcing idea described under Banking System above), both for a complete audit trail and so the exact order book state can be precisely reconstructed if the matching process ever needs to restart after a crash.

Rather than trying to make one single matching engine handle every security at once (which would create a single, massive bottleneck), the system instead runs many independent, single-threaded matching engines in parallel, each one responsible for just one specific security — this achieves real horizontal scaling not by splitting up the work for any ONE security, but by cleanly dividing the overall workload across many entirely separate, independent securities.

Example: this case study specifically tests whether you understand that "scale it more" doesn't always mean "spread it across more machines" — for this one specific hottest, most latency-critical path, a deliberately simple, single-threaded, in-memory design genuinely outperforms a more conventionally "distributed" approach, precisely because it avoids all network and coordination overhead in that one critical path entirely.


23. HOTEL BOOKING

Core requirement: let users search for and book available hotel rooms for specific date ranges, while strictly, absolutely preventing the same specific room from ever being double-booked for overlapping dates.

```
 Two concurrent booking attempts for Room 101, same dates:

 Attempt A ─▶ lock row/date-range ─▶ book ─▶ release lock
 Attempt B ─▶ blocked, waits ──────────────────────▶ retries,
                                                       sees room taken
 (pessimistic locking) — or —
 Attempt A/B both read version=5, whichever commits first wins,
 the other's write is rejected on version mismatch (optimistic)

 Booking flow (Saga): reserve room → charge payment → confirm
 (compensating action: release the hold if payment fails)
```

The defining, genuinely hard sub-problem is double-booking prevention, and it's meaningfully harder than, say, Amazon's simple "don't oversell inventory" problem, because availability here is date-RANGE-based rather than a simple binary in-stock/out-of-stock flag — a specific room is unavailable only for a specific range of dates, so checking availability requires a range-overlap check (does this requested date range overlap with any already-confirmed booking for this same room), not simply checking whether a counter is above zero.

Handling two customers who both try to book the exact same room for overlapping dates at nearly the same moment requires explicit concurrency control at the database level. One approach is pessimistic locking: explicitly lock the specific relevant room/date-range the moment one booking attempt begins working with it, forcing any second, concurrent, conflicting attempt to simply wait until the first one either completes or fails. The alternative is optimistic concurrency control: let both attempts proceed without any upfront lock, but attach a version number (or timestamp) to the availability record, and at the final moment of actually committing the booking, check whether that version number has changed since the attempt first read it — if it has, someone else got there first, and this specific attempt is safely rejected and can be retried against the now-current data.

Searching for available hotels matching a location, date range, and price range is a faceted search problem, using the same kind of dedicated search infrastructure already described under Slack and Amazon above, while the underlying availability data itself is typically partitioned (split across multiple database machines, each handling a subset of the overall data) by hotel or by geographic region, to keep any single machine's data volume manageable as the system grows.

The overall booking flow — reserve the room temporarily, process payment, confirm the booking — is, once again, a multi-step process spanning several independent concerns, handled with the same Saga pattern seen across many of the earlier case studies, with a compensating action that releases the temporary hold on the room if payment ultimately fails.

Example: "two users click 'book' on the very last available room for the exact same dates at the exact same moment — walk me through what happens" is the near-universal specific question here — the expected answer centers on explicit concurrency control (pessimistic locking or optimistic version checking) applied precisely at the moment of reservation, not a naive, unsafe "check availability, then separately book" pair of steps.


24. FLIGHT RESERVATION

Architecturally almost identical to Hotel Booking (topic 23, see diagram above) — the same date/inventory-range double-booking prevention challenge (here, preventing the same specific seat from being sold twice for the same flight), the same concurrency-control solutions (pessimistic locking or optimistic version checking), and the same Saga-based booking flow (reserve seat → process payment → confirm). The main practical difference is that flight inventory is typically modeled per-flight-per-seat-class rather than per-date-range, but the underlying concurrency-safety problem, and its solution, is fundamentally the same one described above.


25. SEARCH ENGINE

Core requirement: crawl, index, and rank content from across a large corpus (the public web, or a specific company's own documents), returning genuinely relevant results for a user's search query within milliseconds.

```
 Crawler ──▶ raw documents ──▶ Tokenize → Stem → build
                                Inverted Index (word → doc list)
                                       │  sharded across many
                                       │  machines
 Query ──▶ tokenize ──▶ look up each term's postings list
                ──▶ merge/intersect ──▶ score (TF-IDF / BM25)
                ──▶ ranked results
```

The foundational data structure making fast text search possible is the inverted index, already introduced briefly under Slack above — rather than storing "this document contains these words" (a "forward" mapping), an inverted index stores the reverse: "this specific word appears in these specific documents." This means answering "which documents contain the word 'wireless'" is a fast, direct lookup, rather than requiring a scan through every single document's full text every single time a query comes in.

Before text can be indexed, it goes through processing steps: tokenization (splitting raw text into individual words, carefully handling punctuation, hyphenation, and language-specific rules) and stemming (reducing different grammatical forms of a word — "running," "runs," "ran" — down to a shared root, so a search for one form correctly also matches documents containing the others).

Ranking determines the ORDER results are shown in, not just which documents match at all. A classic, foundational scoring approach called TF-IDF (Term Frequency-Inverse Document Frequency) combines two signals: how often a search term appears within a specific document (the more it appears, the more that document is presumably "about" that term) and how rare that term is across the ENTIRE document collection (common words like "the" carry very little distinguishing signal, since they appear almost everywhere, while a rare, specific term is a much stronger, more meaningful signal when it does actually match). Modern search engines typically use a more refined successor called BM25, which additionally accounts for diminishing returns (a document mentioning a term 50 times isn't 50 times more relevant than one mentioning it once) and document length (longer documents naturally contain more words, so raw counts alone would unfairly favor them).

At real scale, the index itself is sharded — split across many separate machines, each holding and searching a subset of the total document collection — with a query fanned out to all relevant shards in parallel, and results merged and re-ranked together afterward.

The data-acquisition piece feeding all of this, specifically for web-wide search, is a web crawler — covered in full depth in the next entry below, since it's substantial enough to deserve its own standalone treatment.

Example: if asked to design a general search engine, crawling/data-acquisition is usually the genuinely novel part interviewers most want to see — indexing and ranking, while important, largely amount to correctly applying the concepts described above.


26. WEB CRAWLER

Core requirement: automatically, continuously discover and fetch pages across the web (or a defined subset of it), feeding them into a search index, while being efficient, avoiding duplicate work, and being respectful of the websites being crawled.

```
 Seed URLs ──▶ Crawl Frontier (priority queue, per-domain
                politeness rate-limiting)
                       │
                       ▼
             Fetch page ──▶ Bloom filter check
              (seen this exact URL before? skip if so)
                       │
                       ▼
        Extract new links ──▶ back into Frontier
        Extract content ────▶ MinHash near-duplicate check
                                       │
                                       ▼
                              Search index (if genuinely new)
```

A crawler starts from a set of known "seed" URLs, fetches each page, extracts both its actual content AND every link it contains, adds newly-discovered links to a queue of URLs still waiting to be visited (called the "crawl frontier"), and repeats this process indefinitely. The frontier isn't processed in simple first-in-first-out order — pages are typically prioritized by some estimate of importance (a page linked to by many other pages is likely more valuable to crawl and index sooner) and by how frequently a given page has historically tended to actually change (a news homepage should be recrawled far more often than a page that hasn't changed in years).

Politeness is a genuine, deliberate design requirement: a crawler must respect each site's robots.txt file (a standard, widely-honored convention where a website explicitly specifies which parts of itself crawlers are and aren't allowed to access) and must rate-limit its own request rate on a strictly PER-DOMAIN basis, so it never overwhelms any single website with too many simultaneous requests, regardless of how much total crawling capacity the crawler itself has available.

Duplicate detection needs to work at two genuinely different levels. Avoiding re-crawling a URL you've already visited is handled efficiently using a Bloom filter — a specialized, extremely space-efficient data structure that can quickly answer "have I probably already seen this specific value" with a small, tunable, acceptable chance of a false positive (occasionally, rarely saying "yes, seen it" for something genuinely new), trading a small amount of accuracy for dramatically reduced memory usage compared to storing every single seen URL explicitly and completely. Separately, detecting near-duplicate CONTENT (two entirely different URLs that happen to serve genuinely near-identical content) typically uses a content-fingerprinting technique like MinHash, specifically so the same effective content doesn't get wastefully indexed multiple times under different URLs.

At real scale, the crawling workload itself is distributed across many separate crawler worker machines, typically partitioned by domain — so all the politeness/rate-limiting state for one given website naturally, consistently stays together on one single worker, rather than being scattered unpredictably across many different workers that would each need to somehow coordinate rate limits with each other.

Example: "how do you avoid crawling the same effective page twice, even if it's reachable via two entirely different URLs" is the specific, pointed follow-up question this case study is generally built around — the expected answer distinguishes exact-URL deduplication (Bloom filter) from near-duplicate-content detection (content fingerprinting), since these are genuinely two separate problems with two separate solutions.


27. NOTIFICATION SERVICE

Core requirement: send notifications to users across multiple channels (push notifications, email, SMS), triggered by events happening elsewhere across a broader system, at potentially very high overall volume.

```
 Event (e.g. "order shipped") ──▶ Pub/Sub topic
                                        │
                 ┌──────────────────────┼──────────────────────┐
                 ▼                      ▼                      ▼
          Push queue              Email queue              SMS queue
        (own worker pool)      (own worker pool)        (own worker pool)
          (a slow email provider never blocks push delivery, and vice versa)
                 │
                 ▼ (on repeated failure)
           Dead Letter Queue ──▶ manual inspection
```

Notifications are triggered by events happening in other, independent parts of the system (an order shipped, a friend request received, a payment failed) — this is a natural fit for an event-driven design, where other services simply publish an event ("this happened") without needing to know or care who, if anyone, is actually listening, and the Notification Service independently subscribes to relevant events and reacts to them, fully decoupled from whatever originally triggered the notification.

A single triggering event might need to result in notifications across multiple different channels at once (a push notification AND an email, say) — each channel typically gets its own separate, dedicated queue and pool of worker processes, so that if one specific channel's external provider (say, the email service) is temporarily slow or failing, it doesn't delay or block delivery through the other, entirely unrelated channels (this specific isolation technique — giving each dependency its own separate, dedicated pool of resources so one struggling dependency can't starve resources needed by others — is sometimes called the "bulkhead" pattern, named after a ship's watertight internal compartments, which prevent flooding in one compartment from sinking the entire vessel).

Users typically have configurable preferences (which notification types they want, and through which specific channels), requiring a preferences lookup before actually sending anything, plus rate limiting to avoid overwhelming a user with excessive individual notifications — often implemented as batching/digesting (grouping several related, individual events together into one single combined summary notification, e.g., "you have 5 new likes" rather than sending 5 completely separate individual notifications back to back).

External notification providers (push services, email/SMS gateways) can and do fail transiently — this is handled with automatic retries using exponential backoff (waiting progressively longer between each successive retry attempt, giving a struggling provider genuine breathing room to recover, rather than hammering it repeatedly and immediately), combined with a dead letter queue — a separate holding queue where a notification is moved after repeatedly, persistently failing past some defined retry limit, so it's not silently, permanently lost, and can later be manually inspected and understood by an engineer.

Because the same underlying triggering event might occasionally be delivered more than once (a common, accepted trade-off in many reliable messaging systems, called "at-least-once delivery"), the service needs to deduplicate by a unique event ID, so a user is never accidentally sent the exact same notification twice.

Example: "how do you avoid spamming a user with 20 separate notifications for 20 related events happening within a short window" is a common, expected follow-up — the answer is batching/digesting notifications over a short time window before actually sending, rather than firing one individual notification per individual triggering event unconditionally.


28. API RATE LIMITER

Core requirement: build a standalone service that other applications/APIs can call to check "is this specific incoming request currently allowed, or should it be throttled" — used to protect systems from being overwhelmed by any single client sending too many requests too quickly.

```
 App Instance 1 ─┐
 App Instance 2 ─┼─▶ Shared Redis counter (per client key)
 App Instance 3 ─┘    atomic increment + check against limit
                       (token bucket: tokens refill steadily,
                        each request consumes one; empty
                        bucket = request rejected)

 At extreme scale: shard counters across multiple Redis nodes
 by client key (consistent hashing)
```

There are several standard algorithms for actually deciding whether a given request should be allowed. A fixed window approach counts requests within fixed, discrete time blocks (e.g., "at most 100 requests between 12:00:00 and 12:00:59, then the counter resets") — simple, but has a known weakness: a client could send 100 requests right at the very end of one window and another 100 immediately at the start of the very next window, effectively achieving 200 requests within just a couple of real seconds, despite technically staying within the stated limit on paper each individual window. A sliding window instead continuously considers a genuinely moving time window relative to right now (rather than a fixed clock boundary), correctly smoothing out that exact edge case, at the cost of being somewhat more complex to implement correctly. A token bucket holds a maximum number of "tokens," which steadily refill over time at a constant rate; each request must successfully consume one token to be allowed through, and if the bucket is currently empty, the request is rejected — this naturally, elegantly allows for reasonable short bursts of traffic (as long as tokens have genuinely accumulated during a preceding quieter period) while still strictly enforcing a steady, sustainable long-term average rate. A leaky bucket works similarly but processes queued requests out at a constant, fixed rate regardless of how bursty the incoming traffic actually is, smoothing everything into a steady, predictable outgoing rate.

Designing this as a genuinely STANDALONE, shared service (rather than logic built individually into each separate application) shifts the interesting problem from "which algorithm" to "how do you correctly share rate-limit state across many separate application server instances." If each individual server instance tracked its own local counter completely independently, a client could easily exceed the intended overall limit simply by having their requests happen to land on different specific server instances. The standard fix is a fast, centralized shared data store (commonly Redis, an in-memory key-value database known for very fast atomic increment operations) that every application instance reads from and writes to, ensuring they all correctly, consistently share and update the exact same underlying counters.

At extremely high request volumes, even a single centralized store can itself become a bottleneck — this is addressed either by sharding the rate-limit state itself (distributing different keys' counters — e.g., by user ID or API key — across multiple separate store instances) or by accepting a degree of approximate enforcement (each application instance maintains its own local, approximate counter and only periodically syncs with the shared central store, trading perfect precision for meaningfully reduced coordination overhead).

Example: the specific delta an interviewer wants recognized here, beyond simply reciting the algorithms above, is explicitly discussing how rate-limit STATE is correctly, safely shared across multiple separate server instances — that's the genuinely systems-level part of this specific question.


29. DISTRIBUTED CACHE

Core requirement: design a caching system itself (conceptually, "build your own Redis or Memcached"), rather than simply using one as a ready-made component inside some other larger system.

```
        Consistent-hash ring
     Node A ── Node B ── Node C ── Node D ── (back to A)
        │         │         │         │
     replicas  replicas  replicas  replicas
     (fault tolerance: if Node B dies, its replica
      elsewhere still has the data)

 Key "user:123" → hashed onto ring → lands between Node B and C
                → owned by Node C, replicated to Node D

 Eviction inside each node once full: LRU or LFU
 Invalidation: value changes ──▶ Pub/Sub broadcast to all nodes
```

Distributing cached data across multiple cache nodes uses consistent hashing — a specific hashing technique where both the cache nodes AND the data keys are placed onto a conceptual circular ring, with each key assigned to whichever node is "next" going around that ring. The key benefit of this specific approach, compared to a naive "hash the key modulo the number of servers" scheme, is that adding or removing a single node only requires moving the small portion of keys that fall specifically between that node and its neighbor on the ring — not almost every single key across the entire system, which is what a naive approach would require every time the number of servers changes.

Eviction policy determines what gets removed once an individual cache node's own memory becomes full. LRU (Least Recently Used) removes whichever entry hasn't been accessed for the longest time, on the reasonable assumption that data you haven't touched recently is less likely to be needed again soon. LFU (Least Frequently Used) instead removes whichever entry has been accessed the fewest total number of times overall, which is more resistant to being fooled by a single, temporary burst of access to normally-unpopular data, but can conversely let old, no-longer-relevant "frequently used" items linger longer than they genuinely should.

Replication of cache data across multiple nodes provides fault tolerance — if a single cache node fails, its data isn't entirely lost, forcing every single one of those specific keys back to an expensive database round-trip all at once; instead, a replica node already holding a copy of that same data can immediately, seamlessly take over.

There are two general architectural approaches to how clients actually find the right cache node for a given key: in a client-side approach, the calling application itself directly knows how to hash a key and talk to the correct specific cache node; in a proxy-based approach, a separate router/proxy layer sits in between and handles this routing on the application's behalf, simplifying the client at the cost of an extra network hop.

Keeping cache data correctly up to date when the underlying source data changes (cache invalidation, at a distributed, multi-node level) is commonly solved with a publish-subscribe broadcast — when a specific value changes, a notification is published, and every relevant cache node (and any client-side caches) that's subscribed receives it and correctly invalidates or updates its own local copy of that specific data.

Example: this case study is essentially "prove you deeply understand caching and data distribution, by designing the actual underlying thing" — expect to be asked to specifically justify your chosen eviction policy, your data distribution strategy, and your replication approach, all together as one coherent, self-consistent system.
30. DISTRIBUTED FILE SYSTEM

Core requirement: design a file system (conceptually, "build your own HDFS or Google File System") that reliably stores very large files across many separate machines, optimized specifically for high-throughput, large, sequential reads and writes.

```
              Coordinator cluster (Raft/Paxos consensus)
              "file.mp4 = chunks [C1, C2, C3]"
              "C1 → workers X,Y,Z   C2 → workers Y,Z,W ..."
                         │
      Client asks: "where are file.mp4's chunks?"
                         │
                         ▼
      Coordinator returns worker list ──▶ Client reads chunk
      data DIRECTLY from those workers (bulk data never flows
      through the coordinator itself)

      Worker heartbeats ──▶ Coordinator (dead worker detected
      → automatically re-replicate its chunks elsewhere)
```

Large files are split into fixed-size chunks — typically much larger than a traditional local file system's blocks (often 64MB or 128MB each), since this system is specifically optimized for large-scale sequential access patterns, not the small, random-access reads/writes a database would typically need. Each individual chunk is replicated across multiple different machines (commonly 3 copies, deliberately spread across physically separate racks/machines) both for fault tolerance (losing any single machine never loses any actual data) and to enable multiple clients to read different replicas of a popular file in parallel.

The classic, standard architecture here uses a dedicated coordinator (sometimes called a "NameNode" or "Master") that keeps track of the overall file system's metadata — which specific files exist, exactly which chunks make up each individual file, and precisely which specific machines currently hold each of those chunks — while the actual chunk DATA itself lives on many separate worker machines. A client wanting to read a file first asks the coordinator "which specific machines hold this file's chunks," and then reads the actual bulk chunk data directly from those specific worker machines — critically, the actual large volume of real data itself never has to flow through the coordinator at all, which would otherwise quickly become an overwhelming, single-point bottleneck. Because the coordinator itself is a genuinely critical single point of failure, it's often run as a small cluster of coordinator machines that use a consensus protocol (an algorithm that lets a group of machines reliably agree on a single, shared, correct value even if some individual machines fail or messages between them are delayed) to stay consistently, reliably in sync with each other.

The coordinator continuously monitors worker machine health via periodic "heartbeat" signals (a lightweight, regular "I'm still alive" message), and if a worker stops sending heartbeats and is presumed to have failed, the coordinator automatically triggers re-replication of any chunks that machine was responsible for, restoring each affected chunk back to its target number of healthy replicas elsewhere.

Example: "why not just distribute files directly using a simple random hash, the way a distributed cache does" is a good, pointed way to test understanding here — the expected answer centers on the specific need for a centralized metadata/coordination layer, precisely because a file system needs precise, reliable tracking of exactly which specific chunks make up a given file and where each currently lives, in a way that a purely hash-based key-to-location scheme alone genuinely doesn't provide.


31. EVENT BUS

Core requirement: design a general-purpose event bus (conceptually, "build your own Kafka") — a system letting many independent producers publish events, and many independent consumers subscribe to and reliably receive them.

```
 Topic "orders" split into partitions, each replicated:

 Partition 0 (key: order_id % N) ──▶ Broker A (leader), B, C (replicas)
 Partition 1                     ──▶ Broker B (leader), C, A (replicas)
 Partition 2                     ──▶ Broker C (leader), A, B (replicas)

 Producer ──▶ writes to the partition matching its key
                 (same key ⇒ same partition ⇒ order preserved)

 Consumer Group "billing-service":
   consumer 1 reads partition 0, consumer 2 reads partition 1, ...
   (each tracks its own "offset" — how far it's read — so a
    restart resumes exactly where it left off)
```

Partitioning is the core mechanism enabling both horizontal scalability and ordering guarantees — a stream of events (a "topic") is split into multiple separate partitions, each hosted on a different broker machine, with events sharing the same specific key (e.g., the same conversation ID, or the same user ID) always routed to the same specific partition — this guarantees ordering is preserved WITHIN that one partition, even though there's no guarantee of ordering ACROSS different, unrelated partitions.

Each partition's data is durably stored and replicated across multiple broker machines, so that no committed event is ever lost even if any single specific machine fails — an event bus is explicitly expected to reliably retain events (often for a defined period, allowing consumers to "replay" past events if needed), not merely pass them through transiently and then discard them the moment they're first read.

Consumer group coordination lets multiple separate consumer processes work together to collectively process a topic's events in parallel, while ensuring no two consumers within the same group ever redundantly process the exact same specific partition at the same time — this requires a form of distributed coordination (typically using the same consensus mechanisms mentioned under Distributed File System above) to correctly track and automatically rebalance exactly which consumer is currently responsible for which partition, especially whenever a consumer joins or leaves the group.

Delivery guarantees are a genuine, explicit design choice that needs to be clearly stated. The standard, pragmatic default is at-least-once delivery (guaranteeing an event will never be silently lost, but occasionally, rarely delivering it more than once under certain failure conditions) combined with idempotent consumers (consumer logic specifically designed so that processing the same event twice has no harmful additional effect, typically by checking a unique event ID before acting on it) — achieving genuinely perfect "exactly-once" delivery is possible but meaningfully harder and more expensive, so most real systems accept at-least-once plus idempotency as the practical, pragmatic standard instead.

Each consumer group tracks its own "offset" — precisely how far it has successfully read within each individual partition — so that after a restart, it correctly knows exactly where to resume from, rather than either re-reading everything from the very beginning or accidentally skipping events it hasn't actually processed yet.

Example: expect this question to essentially be "explain a real messaging system's actual internal architecture, as if you were the one building it from scratch" — partitions, replication, consumer groups, and offset tracking are the specific, expected core building blocks of a genuinely strong answer.


32. REAL-TIME ANALYTICS PLATFORM

Core requirement: collect large volumes of raw event data (user clicks, transactions, sensor readings) and process it into aggregated, queryable insights — supporting both real-time ("what's happening right now") and historical/batch ("what happened over the last quarter") analysis simultaneously.

```
 Raw events ──▶ Event Bus (durable, replayable log)
                     │
        ┌────────────┴────────────┐
        ▼                          ▼
  Stream Processor          Batch Processor
  (real-time rolling        (nightly, full historical
   counts/aggregates)        recompute, more thorough)
        │                          │
        ▼                          ▼
  Fast-access store          Data Warehouse
  (live dashboards)          (BI / SQL queries)

 Raw data also lands untouched in a Data Lake (original,
 unprocessed format — flexible for future, not-yet-defined uses)
```

There are two standard architectural approaches worth knowing by name. A Lambda architecture runs two entirely separate, parallel processing paths from the same raw incoming data: a batch layer that periodically, thoroughly processes large historical volumes (favoring completeness and accuracy over speed, since it can take its time), and a separate speed/streaming layer that processes only very recent data with much lower latency (favoring quick availability over perfect completeness) — results from both layers are then merged together whenever a query is actually made. A Kappa architecture simplifies this down to just one single, unified streaming pipeline, treating even historical/batch-style processing as simply "replaying" that exact same event stream from further back in time — this avoids the real, genuine complexity and cost of maintaining two entirely separate codebases doing conceptually similar work, trading away some of Lambda's flexibility for meaningfully reduced overall system complexity.

Raw events flow in through a high-throughput, durable messaging/streaming platform (the same kind of event bus technology described in the previous case study), acting as the reliable, replayable backbone that the rest of the pipeline builds on top of. Stream processing frameworks then consume this raw event stream and perform continuous, real-time aggregation (e.g., "maintain a constantly-updating rolling count of events per minute, broken down per category"), feeding results into a fast-access store specifically for real-time dashboards. Separately (or, in a Kappa-style design, via that same underlying pipeline simply replayed over a longer historical window), large historical datasets are periodically processed using batch computing frameworks, computing more complex, computationally expensive aggregations across the FULL historical dataset that wouldn't be practical to continuously compute in true real time.

Storage is typically split into two distinct tiers: raw event data lands in a data lake — a large, flexible storage repository holding data in its original, raw, native format, without requiring a rigid, predefined schema upfront (this flexibility is valuable since you may not yet fully know every future analytical use case at the time data is first collected). Processed, cleaned, and structured aggregates are then separately loaded into a data warehouse — a repository specifically optimized for fast, reliable, structured analytical querying, requiring data to already be cleaned and organized into a well-defined schema before it's loaded in, which is exactly what makes subsequent business-intelligence queries against it fast and reliable.

Example: being able to name and briefly contrast Lambda versus Kappa architecture specifically, and to clearly justify the data-lake-then-warehouse storage split, is usually what separates a genuinely strong answer here from a merely adequate one.


33. FRAUD DETECTION SYSTEM

Core requirement: analyze transactions (or other user actions) in real time and flag or block those that appear likely to be fraudulent, using machine learning models, while carefully balancing missed fraud (false negatives) against incorrectly flagging genuinely legitimate transactions (false positives).

```
 Transaction ──▶ Fast rule checks (obvious fraud? block immediately)
                        │  (ambiguous cases fall through)
                        ▼
              Feature lookup (Redis: "txns by this card in
              past hour" — computed incrementally, low latency)
                        │
                        ▼
              ML model (online inference, tens-of-ms budget)
                        │
                        ▼
              Approve / Decline / Flag for review

 Drift monitor: compares live transaction patterns vs. training
 data distribution ──▶ triggers retraining when fraud tactics shift
```

Because a fraud decision must genuinely happen within the live transaction's own critical path, this requires online inference — running the ML model's prediction synchronously, immediately, in direct response to a single specific incoming transaction, typically within a strict budget of just tens of milliseconds, rather than computing predictions ahead of time on some periodic schedule (which simply isn't possible here, since you can't know in advance which specific transaction is about to happen).

Feature engineering — the process of transforming raw transaction data into the actual meaningful input signals a model uses — is especially critical here: the real predictive power overwhelmingly comes from well-designed features (like "number of transactions by this same card in the past hour," or "how much does this specific transaction amount deviate from this particular user's normal, typical historical spending pattern") rather than from the specific choice of ML algorithm itself. These specific features need to be looked up or incrementally maintained with very low latency at the exact moment of the transaction — commonly using a fast in-memory store maintaining continuously-updating sliding-window counters.

Fraudsters actively, continuously adapt their own tactics specifically to evade whatever detection methods are currently in place — this means the underlying relationship between "what a transaction looks like" and "is this actually fraud" genuinely, meaningfully changes over time, a phenomenon called concept drift (as distinguished from simple "data drift," where only the raw input data's statistical properties shift, while the underlying relationship to the correct outcome stays the same). This makes a genuinely tight, often event-triggered retraining pipeline essential here rather than optional — a fraud model can become stale and meaningfully less effective within weeks, not months, unlike many other, more stable ML use cases.

A tiered, cascading approach is common in real practice: fast, simple rule-based checks (e.g., "is this transaction amount above some clearly excessive fixed threshold") run first and can immediately, definitively block the most obviously fraudulent cases without needing the full model at all; genuinely borderline or ambiguous cases are then passed on to the more expensive, more accurate full ML model for a more careful, thorough evaluation — balancing overall system latency against genuine detection accuracy.

Example: "how do you keep the model accurate as fraudsters actively, continuously change their own tactics over time" is the defining, central question of this entire case study — the expected answer centers specifically on concept drift and a correspondingly tight, responsive retraining pipeline, distinct from the simpler, slower-moving kind of "data drift" many other, less adversarial ML systems mainly need to worry about.


34. RECOMMENDATION SYSTEM

Core requirement: suggest relevant items (products, videos, content) to users based on their own past behavior and the behavior of other, similar users, at scale and with reasonably low latency.

```
 Offline batch job (nightly):
   compute similarity structures / candidate lists per user
   ──▶ cache results in fast-access store

 Request time:
   Candidate Generation (cheap, broad — narrows millions
   of items down to a few hundred plausible candidates)
                    │
                    ▼
   Ranking (expensive, precise — scores just those few
   hundred candidates, returns top N)
                    │
                    ▼
   User interaction (click/purchase/ignore) ──▶ feeds back
   as new training signal for the next model update
```

There are two foundational approaches. Collaborative filtering recommends items based on the behavior of OTHER, similar users ("users who liked X also liked Y") — this works well once you have substantial behavioral data to draw on, but struggles badly with brand new users or brand new items that have no history yet at all (a well-known challenge called the "cold start" problem). Content-based filtering instead recommends items that are inherently similar, in their own attributes/features, to items a specific user has previously liked — this works fine even for a new user, as long as you know their preferences on at least a few items, but tends to over-narrow recommendations toward very similar items, rather than ever genuinely surprising a user with something new and different. Most real, production recommendation systems combine both approaches together as a hybrid.

Because scoring literally every single possible item for every single user, in true real time, would be far too computationally expensive at real scale, production systems typically use a two-stage pipeline: a fast candidate generation stage first narrows an enormous overall catalog (potentially millions of items) down to a much smaller set of perhaps a few hundred plausible candidates (often using precomputed similarity structures), followed by a more expensive, more precise ranking stage that carefully scores and orders just those few hundred remaining candidates.

Many recommendation signals (like "users who bought X also bought Y") are computed periodically, ahead of time, in large offline batch jobs rather than being recalculated live for every single individual request, with the precomputed results then cached in a fast-access store for quick retrieval whenever actually needed at serving time. User interactions with the shown recommendations (clicks, purchases, or simply ignoring a suggestion) then feed back into the system as new training data/signal, continuously improving future recommendations over time.

Example: the "cold start problem" (how do you meaningfully recommend anything at all to a brand new user with zero prior history) is one of the most commonly asked specific follow-ups here — a good answer combines content-based filtering (using whatever limited signal is actually available, like initial signup preferences) with simply falling back to showing globally popular/trending items until enough genuine behavioral data has accumulated for that specific user.


35. LLM CHAT SYSTEM (CHATGPT-STYLE)

Core requirement: allow users to have multi-turn, real-time conversational interactions with a large language model, maintaining conversation context/history, and streaming the model's response back to the user token-by-token as it's actually generated, rather than making the user wait for the entire, complete response before showing anything at all.

```
 User message ──▶ Context Manager (assembles conversation
                    history, summarizing/trimming if too long
                    for the model's fixed context limit)
                          │
                          ▼
                  Input safety filter (block malicious prompts)
                          │
                          ▼
          Model Serving (batches multiple users' requests
          together on shared GPU hardware for efficiency)
                          │
                          ▼ (token by token)
        Server-Sent Events stream ──▶ Client renders
        incrementally as each token arrives
                          │
                          ▼
                  Output safety filter (before final display)
```

Token streaming is the defining, distinguishing user-experience requirement here — rather than one single request/response cycle (which would otherwise make a user stare at a blank screen for many seconds before seeing any output at all), the model's output is streamed back incrementally, piece by piece, as each new token (a small chunk of generated text, roughly a word or part of a word) is produced. This is naturally implemented using Server-Sent Events — a simple mechanism where the server keeps a connection open and continuously pushes small pieces of data to the client over time, specifically well-suited to exactly this kind of one-directional, incremental, server-to-client streaming pattern.

Conversation context management is a genuine constraint: each new message in an ongoing conversation needs the full relevant prior conversation history included as input context for the model to correctly understand what's being discussed — but large language models have a fundamentally fixed maximum amount of context they can actually process at once, so sufficiently long conversations eventually need a specific strategy (such as periodically summarizing older parts of the conversation into a shorter form, or simply dropping the oldest messages) once that fixed limit is approached.

Model serving at real scale is a genuinely significant operational concern, since large language models are both large and expensive to run — request batching (efficiently grouping multiple different incoming user requests together to process them jointly on the same shared, expensive hardware, rather than handling each one in complete isolation) and careful capacity planning around genuinely expensive specialized compute hardware are central, defining concerns here in a way they simply aren't for most other, more traditional systems.

Safety and guardrails are a standard, essential, expected production requirement: filtering both user inputs (blocking clearly malicious or inappropriate prompts before they ever even reach the model) and the model's own generated outputs (checking generated responses for harmful, biased, or otherwise clearly inappropriate content before actually delivering them back to the user).

Example: "how do you make the response feel fast and real-time despite the model itself taking several seconds to fully generate a complete answer" is the near-universal specific question here — the expected answer centers specifically on token-by-token streaming, rather than simply waiting and returning one single, complete, final response all at once.


36. RAG-BASED KNOWLEDGE ASSISTANT

Core requirement: build a chat-style assistant, similar in overall shape to the LLM Chat System above, that can accurately answer questions about a specific, defined knowledge base (a company's internal documents, a product's documentation), grounding its responses in genuinely real, actually-retrieved source material, rather than relying purely on the underlying language model's own general, pre-existing training knowledge.

```
 Ingestion (offline, ongoing):
   Documents ──▶ Chunk into pieces ──▶ Embedding model
                                              │
                                              ▼
                                     Vector Database

 Query time:
   User question ──▶ Embedding model ──▶ query vector
                                              │
                                              ▼
                          Vector DB: approximate nearest-
                          neighbor search ──▶ top-K relevant chunks
                                              │
                                              ▼
                  Prompt = question + retrieved chunks
                                              │
                                              ▼
                          LLM generates grounded answer,
                          citing which chunks it used
```

The core technique here is called RAG (Retrieval-Augmented Generation), and it solves a genuine, real problem: a general-purpose language model's own training data has some fixed cutoff date and simply doesn't include any given organization's own private, internal documents at all — RAG allows a general-purpose model to correctly, accurately answer questions about current or private information without needing to actually retrain the underlying model itself. Instead, genuinely relevant information is retrieved at the exact moment of the query and explicitly included directly within the model's input prompt.

The process works as follows: a user's query is first converted into a vector embedding — a dense, learned numeric representation that captures a piece of text's underlying meaning, generated by a separate, smaller model specifically trained for this purpose, such that pieces of text with similar meaning end up with mathematically similar-looking embeddings. This resulting query embedding is then used to search a vector database — a specialized database purpose-built for efficiently finding the most similar stored vectors to a given query vector (rather than the exact-match or range-based lookups a traditional database index is built for) — containing embeddings of the organization's own relevant documents, specifically to find the most semantically similar/relevant documents to that particular query. Finding the genuinely closest matches among potentially millions of stored vectors, with mathematically guaranteed exactness, would be too computationally expensive at real scale, so vector databases instead use approximate nearest-neighbor search algorithms, deliberately trading a small, generally acceptable amount of precision for dramatically better real-world speed. These retrieved, relevant documents are then included as additional context directly alongside the user's original query, within the actual final prompt sent to the language model — which then generates its response genuinely grounded in that specific retrieved information, rather than relying purely on its own internal, potentially outdated or simply entirely absent knowledge.

Why RAG is generally preferred over fully retraining or "fine-tuning" a model for many common use cases: fine-tuning is comparatively expensive, and a fine-tuned model's knowledge still becomes stale again the moment the underlying source information later changes, requiring another full, expensive retraining round — RAG instead simply requires updating the vector database directly (a much cheaper, faster, more straightforward operation) to correctly reflect new or changed information, entirely without ever needing to retrain the underlying language model itself at all.

The document ingestion pipeline is the piece genuinely, additionally layered on top of a plain LLM chat system: source documents need to be chunked (split into reasonably-sized, individually retrievable pieces, since full documents are usually far too long to use as a single retrieval unit), converted into embeddings, and loaded into the vector database — initially for all existing documents, and then continuously or incrementally as new or updated documents are added over time.

Retrieval quality directly, critically determines final answer quality: if the retrieval step returns genuinely irrelevant or incomplete source material, the model will generate a confidently-worded but genuinely poor or inaccurate answer, regardless of how capable the underlying model itself actually is — making retrieval evaluation (specifically checking whether the RIGHT relevant chunks are actually being retrieved, distinct from evaluating the quality of the final generated text) a genuinely important, separate part of the overall system's evaluation strategy. Good systems also typically cite exactly which specific source document(s) a given answer was actually grounded in, carrying that source metadata all the way through the pipeline alongside the final generated response.

Example: "the assistant confidently gives a wrong answer — how do you debug whether this is a retrieval problem or a generation problem" is a strong, pointed, and expected follow-up here — the answer is to separately, specifically inspect what chunks were actually retrieved for that particular query first (was the genuinely relevant source document even retrieved at all), before assuming the underlying model itself is simply "hallucinating" (confidently generating plausible-sounding but factually incorrect output) — retrieval quality and generation quality are genuinely separate failure points that need to be debugged independently of each other.


37. MULTI-AGENT AI PLATFORM

Core requirement: coordinate multiple independent, specialized AI agents (each a language-model-powered component with its own specific role — a "research agent," a "coding agent," a "review agent") that collaborate together, potentially across multiple sequential or parallel steps, to accomplish a larger, more complex overall task than any single individual agent could reliably handle entirely on its own.

```
 Orchestration style:
   Orchestrator ──▶ Research Agent ──▶ result
        │      ──▶ Coding Agent    ──▶ result   (explicit,
        │      ──▶ Review Agent    ──▶ result    central control)
        ▼
   Shared task memory (both intermediate results and
   accumulated conversation/task history)

 Choreography style (alternative):
   Agent A publishes event ──▶ Agent B reacts ──▶ publishes event
   ──▶ Agent C reacts ... (no central coordinator, more flexible,
   harder to trace)

 Every agent-to-agent / agent-to-tool call wrapped with:
   timeout + retry-with-backoff + circuit breaker
   (one confused agent shouldn't be able to derail the whole task)
```

There are two general coordination styles, worth knowing by name. In orchestration, a central "orchestrator" component explicitly directs which specific specialized agent handles each particular step of a larger task, and in what order — this is generally simpler to reason about, monitor, and debug, since there's one single, central place tracking the overall flow, but it introduces a genuine central coordination point that everything else depends on. In choreography, agents instead operate independently, each reacting directly to other agents' outputs/events without any single central coordinator directing things — this affords more flexibility and avoids that single central dependency, but is meaningfully harder to trace and reliably debug overall, since there's no single place showing the complete picture.

State and memory management is a genuine consideration: unlike a simple, stateless function call, an individual agent's own useful "state" often includes its accumulated conversational/task history so far, and possibly a working memory of intermediate results it's produced along the way — this commonly uses the same conversation-context-management approach already described under the LLM Chat System above, plus a shared, common data store that multiple different agents can all read from and write to as they progressively, collaboratively work through a larger shared task together.

Agents frequently need to call external tools or APIs (a "research agent" might genuinely need to call a real web search API; a "coding agent" might genuinely need to actually execute code) — this requires a carefully controlled, well-defined interface, since giving an autonomous, language-model-driven agent the genuine ability to actually execute real actions meaningfully raises the security stakes considerably compared to a purely conversational, text-only system. Given how genuinely unpredictable an individual agent step's specific behavior can be (language models are fundamentally probabilistic, not perfectly deterministic), this also requires unusually robust handling of failures around every single individual agent-to-agent or agent-to-tool interaction — using the same general resilience techniques applied throughout many of the earlier case studies: timeouts (never waiting indefinitely on a single step), retries with backoff (giving a struggling step another reasonable chance, waiting progressively longer between attempts), and circuit breakers (temporarily, deliberately stopping calls to a step that's clearly, persistently failing, rather than continuing to retry it indefinitely).

Observability — the ability to understand what's actually happening inside the system by examining the signals it produces — is unusually critical and unusually difficult here specifically: tracing a complex multi-agent task across many sequential or parallel agent steps, each themselves involving a non-deterministic model call, is meaningfully harder than tracing a traditional, fully deterministic request chain through a normal set of backend services, precisely because the exact same input can genuinely, legitimately produce a somewhat different specific execution path or outcome on different individual runs.

Example: "how do you prevent one single malfunctioning or 'confused' agent from derailing an entire complex multi-step task" is a strong, pointed, and genuinely current question here — the expected answer draws directly on the full resilience toolkit described above (timeouts, retries, circuit breakers, isolating each dependency's resources from the others), applied specifically to agent-to-agent calls, plus a clearly defined, sensible fallback or escalation strategy (such as surfacing the issue to a human reviewer, or falling back to a simpler, more constrained workflow) for when an individual agent step genuinely, repeatedly fails or produces low-confidence results.


CLOSING NOTES

A few observations worth internalizing now that you've been through all 37 entries above (a few, like Uber/Ola, Swiggy/Zomato, and Amazon/Flipkart, are genuinely identical designs under different names — bringing the true count of distinct architectures closer to 30).

First: notice how heavily this entire set of case studies reused a comparatively small handful of recurring patterns across very different-sounding products. The fan-out-on-write-vs-read hybrid showed up in Instagram, Facebook, and Twitter. The Saga pattern (breaking a multi-step process spanning several independent parts of a system into individually-committed steps, each with a defined way to undo it) showed up in Uber, Swiggy, Amazon, Payment Gateway, Banking, and Hotel Booking. Geospatial indexing showed up in Uber, Swiggy, and Google Maps. Object storage plus a background processing pipeline showed up in Pastebin, Dropbox, YouTube, Netflix, and Zoom. The two-stage "narrow down candidates, then carefully rank them" pattern showed up in Recommendation Systems and, in spirit, in ranking problems generally.

This is genuinely the actual point of this entire phase: real system design interviews are not testing whether you've memorized 37 separate, distinct system architectures. They're testing whether you can recognize which of a comparatively small number of recurring, genuinely fundamental patterns actually applies to whatever specific problem you're handed, and correctly combine two or three of them together, under real time pressure, while clearly explaining your reasoning out loud.

Second, as ongoing practice: pick any single system from this list, close this document entirely, and give yourself 30-40 minutes to design it completely from scratch, out loud or on paper — clarify requirements first, estimate scale, propose a high-level architecture and sketch your own diagram, then go deep specifically on that system's single hardest sub-problem. Only then compare your own answer back against the write-up above. Doing this repeatedly, across a genuinely wide and varied subset of these systems, is what actually builds real interview-ready fluency — far more than simply re-reading this document passively ever will.
