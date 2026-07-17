# 101 — Design YouTube / Video Streaming
## Category: HLD Case Study

---

## What is this?

YouTube is a system where anyone can upload a giant video file, and then **billions of people can watch that file smoothly** — on a 4K TV plugged into fibre, on a laptop over office WiFi, and on a ₹6,000 Android phone crawling along on 3G in a moving train. All three get a video that plays without stalling.

Think of it like a **restaurant that serves the same dish to every table, but portioned to each diner's appetite**. The kitchen (transcoding) cooks the dish once into six different portion sizes. The waiter (the player) watches how fast you're eating and, between bites, quietly swaps your next bite for a bigger or smaller one. You never notice. You never wait.

The whole design of YouTube falls out of two questions: **how do we turn one arbitrary uploaded file into something every device on earth can play?** (transcoding), and **how do we make sure it never buffers?** (adaptive bitrate streaming). Everything else — the database, the API, the search box — is comparatively easy.

---

## Why does it matter?

Video is the single largest consumer of bandwidth on the internet. If you can design a video platform, you have proven you understand the three things that break at true scale: **enormous storage, enormous egress bandwidth, and long-running asynchronous work**.

**If you don't understand this:**
- You will design a system where the API server receives the uploaded video bytes. At 2 GB per upload, your Node process dies.
- You will transcode the video synchronously inside the upload request. The request times out after 30 seconds; the video is 2 hours long.
- You will serve MP4 files straight from your origin servers, and your cloud bill will be larger than your revenue.
- You will write `UPDATE videos SET views = views + 1` and a viral video will lock that row until your database falls over.

**In interviews:** "Design YouTube" (or Netflix, or TikTok, or Twitch) is one of the top five most-asked HLD questions. The interviewer is not testing whether you know what a load balancer is. They want to hear the words **chunked parallel transcoding**, **manifest**, **HLS/DASH**, **client-side ABR**, and **CDN egress cost**. Say those five things well and you pass.

**At work:** Any product with user-generated media — a fitness app with workout videos, an ed-tech platform, a security-camera SaaS — is a small YouTube. The same pipeline shape applies, whether you build it yourself or wire up AWS MediaConvert / Mux / Cloudflare Stream.

---

## The core idea — explained simply

### The Newspaper Printing Press Analogy

Imagine you run a newspaper read by two billion people.

A journalist hands you a **hand-written manuscript**. It is enormous, messy, and written in her own private shorthand. Nobody can read it as-is. This is the **raw uploaded video**: one giant file, in some arbitrary codec, at some arbitrary bitrate, possibly 40 GB of ProRes footage from a professional camera. Your phone cannot play it. Your network cannot stream it.

So you do what a newspaper does:

1. **You typeset it.** You convert the manuscript into a standard, readable format. → **Transcoding** into H.264/H.265 with standard settings.
2. **You print several editions.** A large-print edition for the elderly, a compact edition for the commuter, a full broadsheet for the reading room. Same content, different sizes. → **Multiple resolutions and bitrates**: 240p, 360p, 480p, 720p, 1080p, 4K.
3. **You don't print it on one endless roll of paper.** You cut it into pages. → **Segmenting** the video into 2–10 second chunks.
4. **You print a table of contents** that says which editions exist and where each page lives. → **The manifest** (`.m3u8` or `.mpd`).
5. **You ship bundles to newsstands in every city** rather than mailing each copy from head office. → **The CDN.**
6. **The reader picks up whichever edition fits in their bag.** Not the printer. Not head office. The *reader*. → **Adaptive bitrate: the client decides.**

That's YouTube. Every technical decision maps back to one of those six steps.

| Newspaper | Video system | Why it exists |
|-----------|--------------|---------------|
| Hand-written manuscript | Raw upload (any codec, huge) | Whatever the creator's camera produced |
| Typesetting | Transcoding to H.264/H.265/VP9/AV1 | Devices only have hardware decoders for a few codecs |
| Multiple editions | 240p → 4K ladder | One size cannot fit both fibre and 3G |
| Cutting into pages | Segmenting (2–10 s chunks) | Lets you switch quality mid-video, and lets you parallelise the work |
| Table of contents | Manifest (`.m3u8` / `.mpd`) | The player needs to know what exists and where |
| Newsstands in every city | CDN edge PoPs | Bytes are expensive and slow to move far |
| Reader picks their edition | Client-side ABR algorithm | Only the client knows the *actual* current network speed |
| Printing takes hours | Video sits in `PROCESSING` | Which is why a fresh upload is 360p-only for a while |

The single most important sentence in this whole document: **the server just serves static files; all the intelligence lives in the player.** That is what makes video infinitely cacheable, and therefore affordable.

---

## Key concepts inside this topic

We'll run the standard 5-step framework from [93 — The HLD Interview Framework], with two extra-deep sections for the transcoding pipeline and ABR.

---

### 1. Requirements

**Clarifying questions to ask the interviewer first** (never skip this — it takes 90 seconds and it's how you show seniority):

- Are we building the whole of YouTube, or the core upload-and-watch path? *(Almost always: core path. Scope down aggressively.)*
- Live streaming, or only video-on-demand (VOD)? *(Say: "I'll assume VOD; live is a different beast — I can cover it at the end if you'd like.")*
- Do we need DRM / paid content? *(Usually: no, out of scope.)*
- Which matters more — upload speed or playback smoothness? *(The answer is always playback. Confirm it out loud.)*

**Functional requirements (in scope):**

| # | Requirement |
|---|-------------|
| F1 | A user can **upload a video** (potentially multi-GB, from a flaky connection) |
| F2 | A user can **watch a video smoothly on any device, on any connection** |
| F3 | A user can **search** for videos by title/description/channel |
| F4 | A user gets **recommendations** (home feed, "up next") |
| F5 | A user can **like, comment**, and see a **view count** |

**Out of scope (say this explicitly):** live streaming, monetisation/ads, Shorts, DRM, moderation UI.

**Non-functional requirements — this is where the design is actually decided:**

| # | Requirement | What it forces |
|---|-------------|----------------|
| N1 | **Extremely read-heavy.** Roughly 1 upload per ~1,000+ views. | Optimise the read path obsessively. Uploads can be slow; playback cannot. |
| N2 | **Playback must start fast (<2 s) and NEVER buffer.** | This is *the* product requirement. Users tolerate a slightly blurry frame. They do not tolerate a spinning wheel. This single line is why ABR exists. |
| N3 | **Works on a 4K TV on fibre AND a cheap phone on 3G.** | One encoding cannot serve both → the bitrate ladder. |
| N4 | **Massive storage** (petabytes/day). | Blob storage + tiering. Never a database. |
| N5 | **Global low latency.** | CDN, non-negotiable. |
| N6 | **High availability** for playback (99.99%+). Upload can be 99.9%. | Asymmetric SLOs: degrade uploads before you degrade watches. |
| N7 | **Cost is a design constraint, not an afterthought.** | Egress bandwidth is the biggest line item on the bill. Design around it. |

> **Interview tip:** Write N2 on the board and circle it. Then say: "Everything I design from here serves this line." That framing alone puts you in the top decile of candidates.

---

### 2. Capacity estimation

Show every line. Round aggressively. Nobody wants precision; they want to see that you can *reason* with magnitudes.

**Given:**
- 2 billion monthly users
- 5 million videos uploaded per day
- 1 billion hours watched per day

**a) Upload write QPS**

```
5,000,000 uploads/day ÷ 86,400 s/day ≈ 58 uploads/sec  (average)
Peak ≈ 3× average                    ≈ 175 uploads/sec
```

58 writes/sec is *nothing*. A single Postgres box could take the metadata. **The upload path is not hard because of QPS — it's hard because of bytes.**

**b) Raw storage per day**

```
Average video length            = 10 minutes
Raw uploaded footage bitrate    ≈ 50 MB per minute   (~6.7 Mbps; a modest 1080p phone capture)

Per video:  10 min × 50 MB/min  = 500 MB
Per day:    5,000,000 × 500 MB  = 2,500,000,000 MB
                                = 2,500,000 GB
                                = 2,500 TB
                                ≈ 2.5 PB/day of ORIGINAL footage
```

**c) Now multiply by transcoding.** This is the step candidates forget.

You do not store one file. You store the *entire ladder*:

```
240p   ≈ 0.10× the source size
360p   ≈ 0.15×
480p   ≈ 0.25×
720p   ≈ 0.50×
1080p  ≈ 1.00×
4K     ≈ 4.00×   (only for videos actually uploaded in 4K — say 10% of them)
+ audio-only tracks, thumbnails, preview sprites  ≈ 0.05×

Sum for a typical non-4K video ≈ 2.05× the source
Plus the ~10% of videos that get a 4K rendition   → pushes the blended factor to ~2.5×
```

```
Transcoded output per day ≈ 2.5 PB × 2.5  ≈ 6 PB/day
Call it 5–7 PB/day. Plus the original, which you keep (for re-transcoding
when a better codec like AV1 ships).

Per year: ~6 PB × 365 ≈ 2,200 PB ≈ 2.2 EXABYTES/year
Over 5 years: ~11 EB
```

**Stop and say this out loud in the interview:** *"That factor of ~2.5× from transcoding is why the architecture looks the way it does. We are not storing videos; we are storing families of videos. Everything downstream — the tiering, the cold storage, the decision to only generate 4K on demand for unpopular videos — exists to fight that multiplier."*

**d) Bandwidth — the real monster**

```
1,000,000,000 hours watched/day
Average streaming bitrate ≈ 5 Mbps  (a blend of 360p phones and 1080p desktops)

Bytes per hour @ 5 Mbps = 5 Mbps × 3,600 s = 18,000 Mb = 18,000 / 8 = 2,250 MB ≈ 2.25 GB/hour

Total egress/day = 1e9 hours × 2.25 GB = 2.25e9 GB = 2,250 PB/day ≈ 2.25 EB/day

Average egress bandwidth:
  2.25e9 GB/day × 8 bits/byte = 1.8e10 Gb/day
  ÷ 86,400 s = ~208,000 Gbps  ≈ 208 Tbps average
Peak (evening, ~3×)           ≈ 600+ Tbps
```

**Now the punchline.** At a *heavily discounted* wholesale CDN rate of, say, $0.005/GB:

```
2.25e9 GB/day × $0.005 = ~$11,000,000 per day  ≈ $4 BILLION per year
```

At standard public cloud egress ($0.05–0.09/GB) it would be **$40+ billion/year** — more than YouTube's entire revenue.

> **This one number explains the whole industry.** It is why a CDN is not optional. It is why Google and Netflix **build their own CDNs and physically install their own servers inside ISP data centres** (Google Global Cache, Netflix Open Connect). If the bytes never traverse a paid transit link — if they hop straight from a box in your ISP's rack to your house — the marginal cost approaches zero. **The bandwidth bill dwarfs storage, compute, and everything else combined.** Say this in the interview and watch the interviewer nod.

**e) Transcoding compute**

```
5M videos/day × 10 min = 50,000,000 video-minutes/day
Transcoding 1 minute of video into the full ladder ≈ 4 CPU-minutes (rough)
= 200,000,000 CPU-minutes/day ÷ 1,440 min/day ≈ 139,000 CPU-cores running flat out
```

~140k cores of transcoding fleet. Large, but a rounding error next to the bandwidth bill. And this is exactly why hardware transcoding ASICs (Google built **Argos VCUs** for this) pay for themselves.

---

### 3. API design

Keep the public API boring. The bytes never go through it.

```
// ---------- Upload ----------
POST   /api/v1/videos
  body: { title, description, tags[], visibility, fileSizeBytes, mimeType }
  →     { videoId, uploadId, uploadUrls: [{ partNumber, url, expiresAt }, ...] }
  // Returns PRE-SIGNED URLs. The client PUTs bytes DIRECTLY to blob storage.
  // Our API server never touches a single video byte.

PUT    <pre-signed blob URL>              // client → blob storage, one per chunk
  →     { ETag }

POST   /api/v1/videos/{videoId}/complete
  body: { uploadId, parts: [{ partNumber, eTag }, ...] }
  →     { videoId, status: "PROCESSING" }

GET    /api/v1/videos/{videoId}/upload-status
  →     { uploadId, receivedParts: [1,2,3,5], missingParts: [4,6] }   // for resume

// ---------- Watch ----------
GET    /api/v1/videos/{videoId}
  →     { videoId, title, description, channel, durationSec, status,
          manifestUrl: "https://cdn.../v/{videoId}/master.m3u8",
          thumbnailUrl, viewCount, likeCount }

GET    https://cdn.../v/{videoId}/master.m3u8        // static file, from CDN
GET    https://cdn.../v/{videoId}/720p/seg_00042.ts  // static file, from CDN

// ---------- Engagement ----------
POST   /api/v1/videos/{videoId}/view      body: { positionSec, sessionId }  // fire-and-forget → Kafka
POST   /api/v1/videos/{videoId}/like
POST   /api/v1/videos/{videoId}/comments  body: { text, parentCommentId? }
GET    /api/v1/videos/{videoId}/comments?cursor=&limit=20

// ---------- Discovery ----------
GET    /api/v1/search?q=&cursor=&limit=20
GET    /api/v1/feed/home?cursor=
```

**Two things to point at:**
1. **Pre-signed URLs** mean the API tier scales with *requests*, not with *gigabytes*. This is the single most important API design decision in the whole system.
2. **Playback URLs point at the CDN, not at us.** Our servers hand out a piece of paper with an address on it; the CDN delivers the parcel.

---

### 4. The upload and TRANSCODING pipeline — centrepiece #1

#### Why transcode at all?

A creator uploads whatever their camera or editor produced. That might be:
- 40 GB of Apple ProRes at 200 Mbps — a codec no phone has a hardware decoder for
- A 4K H.265 file that a 2016 Android phone cannot decode
- A weird VP8-in-WebM file from an old screen recorder

Three separate problems, all fatal:
1. **Compatibility.** Devices only play what their *hardware decoder chip* supports. Software decoding a 4K stream on a phone melts the battery in 20 minutes. You must produce H.264 (universal) plus optionally VP9/AV1 (efficient, for devices that support them).
2. **Bitrate.** A 200 Mbps source cannot stream over a 2 Mbps 3G link. Ever. No amount of buffering saves you.
3. **Structure.** One monolithic file cannot be adapted mid-playback and is miserable to cache.

So: **transcode once, at upload time, into a ladder of playable, segmented renditions.** You pay compute once and save bandwidth and pain forever.

#### The pipeline, step by step

```
┌────────────┐   1. POST /videos (metadata only)   ┌──────────────┐
│            │ ──────────────────────────────────▶ │  Upload API  │
│   Client   │ ◀────── pre-signed part URLs ─────── │   (Node)     │
│  (browser/ │                                      └──────┬───────┘
│    phone)  │                                             │ write row
│            │   2. PUT part 1..N (RESUMABLE, parallel)    │ status=UPLOADING
│            │ ──────────────┐                             ▼
└────────────┘               │                      ┌──────────────┐
                             │                      │  Metadata DB │
                             ▼                      │  (Postgres)  │
                    ┌──────────────────┐            └──────────────┘
                    │   BLOB STORAGE   │  3. "complete" → object event
                    │  (raw/ bucket)   │ ──────────────────────┐
                    └──────────────────┘                       ▼
                                                     ┌───────────────────┐
                                                     │   MESSAGE QUEUE   │
                                                     │  (Kafka / SQS)    │
                                                     │  topic: video.new │
                                                     └─────────┬─────────┘
                                                               │
                                                               ▼
                                                     ┌───────────────────┐
                                                     │  ORCHESTRATOR     │
                                                     │  (builds the DAG) │
                                                     └─────────┬─────────┘
                     ┌──────────────┬──────────────┬───────────┴───────┬──────────────┐
                     ▼              ▼              ▼                   ▼              ▼
              ┌───────────┐  ┌───────────┐  ┌───────────┐      ┌───────────┐  ┌───────────┐
              │ SPLITTER  │  │ TRANSCODE │  │ TRANSCODE │ ...  │ THUMBNAIL │  │  AUDIO    │
              │  (chunk   │  │ WORKER #1 │  │ WORKER #2 │      │  WORKER   │  │ EXTRACTOR │
              │  into 6s) │  │ chunk 1   │  │ chunk 2   │      │           │  │           │
              └─────┬─────┘  └─────┬─────┘  └─────┬─────┘      └─────┬─────┘  └─────┬─────┘
                    │              │              │                  │              │
                    └──────────────┴──────┬───────┴──────────────────┴──────────────┘
                                          ▼
                                 ┌────────────────┐
                                 │  STITCH +      │   concatenate chunks per rendition,
                                 │  PACKAGE       │   write .ts/.m4s segments + manifests
                                 └───────┬────────┘
                                         ▼
                                 ┌────────────────┐
                                 │  BLOB STORAGE  │   (encoded/ bucket, replicated)
                                 │  + CDN WARM-UP │
                                 └───────┬────────┘
                                         ▼
                                 ┌────────────────┐
                                 │  Metadata DB   │   status = READY
                                 │  + notify user │
                                 └────────────────┘
```

**The key insight — parallelism through chunking:**

Transcoding a 2-hour 4K video serially would take **hours**. Real-time-ish transcoding on one core is roughly 1× to 4× the video's duration *per rendition*, and you have six renditions.

But video is **embarrassingly parallel** if you cut it right. Split it at **keyframe (GOP) boundaries** into 6-second chunks:

```
2-hour video  =  7,200 seconds  ÷  6 s  =  1,200 chunks
1,200 chunks × 6 renditions      =  7,200 independent transcode jobs
Throw 1,000 workers at it        →  each does ~7 jobs
Each job ≈ 6 s of video ≈ 10–20 s of CPU
                                 →  the whole thing finishes in ~2 MINUTES
```

Hours → minutes. **That is the single most important sentence in the transcoding section.** You must cut on keyframe boundaries so each chunk decodes independently — otherwise chunk N needs frames from chunk N−1 and the parallelism collapses.

**It's a DAG, not a queue.** Splitting must finish before transcoding starts; all chunks of a rendition must finish before stitching; stitching must finish before packaging; packaging before CDN push. Use a workflow engine (Temporal, Step Functions, Airflow) or roll your own state machine. Every node emits an event; every node can be retried.

**Failures are normal, so every step must be idempotent.** A worker will die mid-chunk — spot instances get reclaimed, ffmpeg segfaults on a corrupt frame. The rule: **a job's output is written to a deterministic path** (`encoded/{videoId}/720p/chunk_00042.ts`). If a worker crashes and the job is redelivered, the retry simply overwrites the same object. No duplicates, no corruption. **Never** use "append to a list" semantics in a retryable pipeline.

**The PROCESSING state — and the real-world proof that this design is correct:**

The video row starts at `status = 'PROCESSING'`. It becomes `READY` when the ladder is done. And here is the observable, real-world detail that proves the architecture:

> When you upload a video to YouTube and open it immediately, **only 360p is available in the quality menu.** The higher resolutions appear over the next several minutes. Why? Because the pipeline **prioritises the low renditions**: 360p is cheap and fast to encode, so YouTube publishes it first so the video is watchable, then backfills 720p, 1080p, and 4K as those jobs complete. It's a *progressive-availability* pipeline, not an all-or-nothing one.

Mention that in an interview and you sound like someone who has actually shipped video.

#### Node.js: the chunked/resumable upload handler

A 2 GB upload over a mobile connection **will** fail partway. Resumability is not a nice-to-have; it is mandatory. The client uploads parts independently, in parallel, and can retry any individual part.

```js
// upload-service.js — the API server never touches video bytes.
// It only mints pre-signed URLs and tracks which parts have landed.

import express from 'express';
import { randomUUID } from 'node:crypto';
import {
  S3Client, CreateMultipartUploadCommand, UploadPartCommand,
  CompleteMultipartUploadCommand, ListPartsCommand, AbortMultipartUploadCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'us-east-1' });
const RAW_BUCKET = 'yt-raw-uploads';
const PART_SIZE = 8 * 1024 * 1024;      // 8 MB parts: small enough that a failed
                                        // part on 3G is cheap to redo, large enough
                                        // that we don't mint 10,000 URLs.
const URL_TTL_SECONDS = 6 * 60 * 60;    // long-lived: a 2 GB upload on a slow link
                                        // can genuinely take hours.

const app = express();
app.use(express.json());

// ---------- STEP 1: initiate ----------
app.post('/api/v1/videos', async (req, res) => {
  const { title, description, fileSizeBytes, mimeType } = req.body;

  if (!/^video\//.test(mimeType ?? '')) {
    return res.status(400).json({ error: 'not a video mime type' });
  }
  if (fileSizeBytes > 256 * 1024 * 1024 * 1024) {   // 256 GB hard cap
    return res.status(413).json({ error: 'file too large' });
  }

  const videoId = randomUUID();
  const key = `raw/${videoId}/source`;

  // Tell blob storage: "a multipart upload is coming."
  const { UploadId } = await s3.send(new CreateMultipartUploadCommand({
    Bucket: RAW_BUCKET, Key: key, ContentType: mimeType,
  }));

  const partCount = Math.ceil(fileSizeBytes / PART_SIZE);
  const uploadUrls = await Promise.all(
    Array.from({ length: partCount }, (_, i) => i + 1).map(async (partNumber) => ({
      partNumber,
      url: await getSignedUrl(s3, new UploadPartCommand({
        Bucket: RAW_BUCKET, Key: key, UploadId, PartNumber: partNumber,
      }), { expiresIn: URL_TTL_SECONDS }),
    })),
  );

  await db.videos.insert({
    id: videoId, user_id: req.user.id, title, description,
    status: 'UPLOADING', s3_upload_id: UploadId, s3_key: key,
    created_at: new Date(),
  });

  // The client now PUTs bytes straight to blob storage. Our process is done.
  res.status(201).json({ videoId, uploadId: UploadId, partSize: PART_SIZE, uploadUrls });
});

// ---------- STEP 2 (resume): which parts already landed? ----------
// The client calls this after a crash / app restart / network death and
// only re-uploads the parts that are missing. THIS is what makes it resumable.
app.get('/api/v1/videos/:videoId/upload-status', async (req, res) => {
  const video = await db.videos.findById(req.params.videoId);
  if (!video || video.user_id !== req.user.id) return res.sendStatus(404);

  const { Parts = [] } = await s3.send(new ListPartsCommand({
    Bucket: RAW_BUCKET, Key: video.s3_key, UploadId: video.s3_upload_id,
  }));

  res.json({
    uploadId: video.s3_upload_id,
    receivedParts: Parts.map((p) => ({ partNumber: p.PartNumber, eTag: p.ETag })),
  });
});

// ---------- STEP 3: complete → publish the event ----------
app.post('/api/v1/videos/:videoId/complete', async (req, res) => {
  const { parts } = req.body;                 // [{ partNumber, eTag }, ...]
  const video = await db.videos.findById(req.params.videoId);
  if (!video || video.user_id !== req.user.id) return res.sendStatus(404);

  await s3.send(new CompleteMultipartUploadCommand({
    Bucket: RAW_BUCKET, Key: video.s3_key, UploadId: video.s3_upload_id,
    MultipartUpload: {
      // S3 requires parts sorted ascending by part number.
      Parts: [...parts].sort((a, b) => a.partNumber - b.partNumber)
        .map((p) => ({ PartNumber: p.partNumber, ETag: p.eTag })),
    },
  }));

  await db.videos.update(video.id, { status: 'PROCESSING' });

  // Hand off to the pipeline. Everything after this is asynchronous.
  // The user gets their HTTP 202 immediately and closes the tab.
  await queue.publish('video.uploaded', {
    videoId: video.id,
    sourceKey: video.s3_key,
    // Idempotency key: if this message is delivered twice, the orchestrator
    // sees the same key and refuses to start a second pipeline.
    idempotencyKey: `transcode:${video.id}:${video.s3_upload_id}`,
  });

  res.status(202).json({ videoId: video.id, status: 'PROCESSING' });
});

export default app;
```

Note what is *absent*: no `multer`, no `req.pipe(fs.createWriteStream(...))`, no video bytes anywhere near the Node event loop. That is the whole trick.

---

### 5. Adaptive Bitrate Streaming (ABR) — centrepiece #2

**The question this answers: "How do you make sure playback NEVER buffers?"**

The honest answer: **you don't try to keep the quality constant — you let the quality vary so the *playback* stays constant.** Users will accept a momentarily softer picture. They will not accept a spinning wheel. ABR trades pixels for continuity, every few seconds, forever.

#### The video is not one file

This is the mental model shift. When you watch a YouTube video you are not downloading `video.mp4`. You are downloading **thousands of tiny files**:

```
/v/abc123/
  master.m3u8                 ← the manifest: "here is what exists"
  240p/  index.m3u8  seg_00000.ts  seg_00001.ts  ...  seg_01199.ts
  360p/  index.m3u8  seg_00000.ts  seg_00001.ts  ...  seg_01199.ts
  480p/  index.m3u8  seg_00000.ts  ...
  720p/  index.m3u8  seg_00000.ts  ...
  1080p/ index.m3u8  seg_00000.ts  ...
  2160p/ index.m3u8  seg_00000.ts  ...
  audio/ index.m3u8  seg_00000.aac ...
```

Every rendition is cut at **exactly the same timestamps**. Segment 42 of 360p covers seconds 252–258, and so does segment 42 of 1080p. That alignment is what makes seamless switching possible: the player can finish playing 1080p segment 41 and start playing 360p segment 42, and the frames line up perfectly. If they weren't aligned, you'd get a visible glitch.

#### The manifest — a real `.m3u8`

**The master playlist** (`master.m3u8`) — the menu of available qualities:

```m3u8
#EXTM3U
#EXT-X-VERSION:6

#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="aud",NAME="English",DEFAULT=YES,URI="audio/index.m3u8"

#EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=426x240,CODECS="avc1.42c015,mp4a.40.2",AUDIO="aud"
240p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360,CODECS="avc1.42c01e,mp4a.40.2",AUDIO="aud"
360p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=854x480,CODECS="avc1.4d401e,mp4a.40.2",AUDIO="aud"
480p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2",AUDIO="aud"
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2",AUDIO="aud"
1080p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=16000000,RESOLUTION=3840x2160,CODECS="avc1.640033,mp4a.40.2",AUDIO="aud"
2160p/index.m3u8
```

`BANDWIDTH` is the crucial field: it tells the player *"you need this many bits per second to sustain this rendition."* The player compares that number to its measured throughput.

**A media playlist** (`720p/index.m3u8`) — the list of segments for one quality:

```m3u8
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD

#EXTINF:6.000,
seg_00000.ts
#EXTINF:6.000,
seg_00001.ts
#EXTINF:6.000,
seg_00002.ts
...
#EXTINF:4.320,
seg_01199.ts
#EXT-X-ENDLIST
```

That's it. **Plain text pointing at plain static files.** No server-side session, no streaming protocol, no stateful connection. Just HTTP GETs for immutable objects — which is *exactly* what a CDN was born to serve. (Recall [60 — CDN]: cacheable, immutable, static = perfect edge content.)

#### The switching timeline — the diagram to draw on the whiteboard

```
Network throughput (Mbps)
 10 │████████████                                    ████████████████
    │            ████                            ████
  5 │                ████                    ████
    │                    ████            ████
  1 │                        ████████████                      ← user walks into a lift
  0 └────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬───▶ time
         │    │    │    │    │    │    │    │    │    │    │    │
Segment: 0    1    2    3    4    5    6    7    8    9    10   11

Quality  ┌────┬────┬────┬────┐
chosen:  │1080│1080│1080│720 │
by the   └────┴────┴────┴────┴────┬────┬────┐
player                            │480 │360 │360 │
                                  └────┴────┴────┴────┬────┬────┬────┐
                                                      │480 │720 │1080│1080│
                                                      └────┴────┴────┴────┘

Buffer   ▓▓▓▓▓▓▓▓▓▓ 30s   ▓▓▓▓▓▓ 18s   ▓▓▓▓ 12s   ▓▓▓▓▓▓ 20s   ▓▓▓▓▓▓▓▓▓▓ 30s
health:      HEALTHY         DRAINING      LOW        REFILLING     HEALTHY

Playback: ████████████████████████████████████████████████████████████████
          NEVER STOPS. Not once. The picture got soft for ~20 seconds. That's all.
```

Read that diagram slowly. The user walked into a lift at segment 3. Bandwidth collapsed from 10 Mbps to under 1 Mbps. A naive player streaming one 5 Mbps MP4 would have drained its buffer and **stalled**. The ABR player instead noticed the buffer draining, stepped down 1080p → 720p → 480p → 360p over the next few segments, kept the buffer alive, and **playback never paused**. When they walked out of the lift, it climbed back up.

#### Where the intelligence lives

**In the client. Not the server.** This is the punchline of the entire section, and it is what candidates most often get backwards.

The player runs a loop, roughly every segment:

```
loop every segment boundary:
    measuredThroughput = bytesDownloaded / downloadTime   (EWMA over last few segments)
    bufferLevel        = seconds of video currently buffered ahead of playhead

    if bufferLevel is LOW           → step DOWN aggressively (panic mode: keep playing!)
    else if measuredThroughput comfortably exceeds the next-higher rendition's BANDWIDTH
         AND buffer is healthy      → step UP one rung (never two — avoid oscillation)
    else                            → stay where you are

    request the chosen rendition's NEXT segment
```

Two signals, throughput and buffer level, and a deliberate asymmetry: **drop fast, climb slow.** Dropping fast prevents a stall. Climbing slowly prevents the flickering quality oscillation that users find more annoying than a permanently lower quality. (Netflix's BOLA and similar buffer-based algorithms formalise exactly this.)

**Why client-side is the *right* answer, not just a convenient one:**

| If the server decided | If the client decides (reality) |
|---|---|
| Server must track per-viewer state → stateful, un-cacheable | Server is stateless; every byte is a cacheable static file |
| Server guesses your bandwidth from afar (badly — it can't see your WiFi) | Client *measures* actual throughput and buffer directly |
| CDN cannot cache a personalised stream | CDN caches segments once, serves them a million times |
| Egress cost explodes | Egress cost collapses |

**The server does nothing clever. That is the design.** All the cleverness is 200 lines of JavaScript in the player (hls.js, Shaka, ExoPlayer, AVPlayer), and the entire delivery tier becomes a dumb, infinitely-scalable static file server. This is the single most impressive thing you can explain in a video-streaming interview.

#### Node.js: the manifest generator

This runs at the end of the transcoding pipeline, once all segments exist in blob storage.

```js
// manifest-generator.js — turns a finished transcode into HLS playlists.

/** The bitrate ladder. BANDWIDTH is the peak bits/sec the player must sustain. */
const LADDER = Object.freeze([
  { name: '240p',  width: 426,  height: 240,  bandwidth: 400_000,    codec: 'avc1.42c015' },
  { name: '360p',  width: 640,  height: 360,  bandwidth: 800_000,    codec: 'avc1.42c01e' },
  { name: '480p',  width: 854,  height: 480,  bandwidth: 1_400_000,  codec: 'avc1.4d401e' },
  { name: '720p',  width: 1280, height: 720,  bandwidth: 2_800_000,  codec: 'avc1.4d401f' },
  { name: '1080p', width: 1920, height: 1080, bandwidth: 5_000_000,  codec: 'avc1.640028' },
  { name: '2160p', width: 3840, height: 2160, bandwidth: 16_000_000, codec: 'avc1.640033' },
]);

const AUDIO_CODEC = 'mp4a.40.2';   // AAC-LC — universally supported
const TARGET_SEGMENT_SEC = 6;

/**
 * Master playlist: the menu. Lists every rendition the player MAY choose from.
 * Only include renditions we actually produced — never advertise a 4K rendition
 * for a 720p source, or the player will 404 the moment it steps up.
 */
export function buildMasterPlaylist({ availableRenditions }) {
  const lines = ['#EXTM3U', '#EXT-X-VERSION:6', ''];

  lines.push(
    '#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="aud",NAME="English",' +
    'DEFAULT=YES,AUTOSELECT=YES,URI="audio/index.m3u8"',
    '',
  );

  // Ascending bandwidth order: players pick the first entry they can sustain
  // for their very first segment, so leading with the cheap rung means a fast start.
  for (const r of LADDER) {
    if (!availableRenditions.includes(r.name)) continue;
    lines.push(
      `#EXT-X-STREAM-INF:BANDWIDTH=${r.bandwidth},` +
      `RESOLUTION=${r.width}x${r.height},` +
      `CODECS="${r.codec},${AUDIO_CODEC}",AUDIO="aud"`,
      `${r.name}/index.m3u8`,
    );
  }

  return lines.join('\n') + '\n';
}

/**
 * Media playlist for ONE rendition: the ordered list of segment files.
 * `segments` is [{ file, durationSec }, ...] as emitted by the packager.
 */
export function buildMediaPlaylist({ segments }) {
  // TARGETDURATION must be >= the longest segment, rounded UP to an integer,
  // or strict players (Safari) will reject the playlist outright.
  const target = Math.ceil(Math.max(...segments.map((s) => s.durationSec)));

  const lines = [
    '#EXTM3U',
    '#EXT-X-VERSION:6',
    `#EXT-X-TARGETDURATION:${target}`,
    '#EXT-X-MEDIA-SEQUENCE:0',
    '#EXT-X-PLAYLIST-TYPE:VOD',   // VOD = the full list is here and will never change,
    '',                           // which is what lets the CDN cache it forever.
  ];

  for (const seg of segments) {
    lines.push(`#EXTINF:${seg.durationSec.toFixed(3)},`, seg.file);
  }

  lines.push('#EXT-X-ENDLIST', '');
  return lines.join('\n');
}

/** Write every playlist for a finished video and warm the CDN. */
export async function publishManifests({ videoId, renditions, blob, cdn }) {
  const base = `encoded/${videoId}`;

  for (const [name, segments] of Object.entries(renditions)) {
    await blob.put(`${base}/${name}/index.m3u8`, buildMediaPlaylist({ segments }), {
      contentType: 'application/vnd.apple.mpegurl',
      // Segments are immutable; the VOD playlist is too. Cache aggressively —
      // this cache header is worth millions of dollars a year at YouTube's scale.
      cacheControl: 'public, max-age=31536000, immutable',
    });
  }

  const master = buildMasterPlaylist({ availableRenditions: Object.keys(renditions) });
  await blob.put(`${base}/master.m3u8`, master, {
    contentType: 'application/vnd.apple.mpegurl',
    // The master is SHORTER-lived: new renditions get appended as higher-quality
    // transcodes finish (this is the "360p first, 1080p later" behaviour).
    cacheControl: 'public, max-age=60',
  });

  // Prefetch the first few segments of the low renditions into the edge, so the
  // very first viewer doesn't eat a cold-cache origin fetch on segment 0.
  await cdn.prefetch([
    `${base}/master.m3u8`,
    `${base}/360p/index.m3u8`,
    `${base}/360p/seg_00000.ts`,
    `${base}/360p/seg_00001.ts`,
  ]);

  return `${base}/master.m3u8`;
}
```

---

### 6. CDN and delivery

Video is the **ultimate CDN use case**. Recall [60 — CDN]: a CDN is a network of caching servers placed physically close to users. It works best when content is:

| CDN wants content that is... | Video segments are... |
|---|---|
| **Static** | Literally files on disk |
| **Immutable** | `seg_00042.ts` never changes. Ever. |
| **Large** | 6 seconds of 1080p ≈ 3–4 MB |
| **Requested by many users** | A hit video: tens of millions of requests for the same segment |
| **Cacheable with no auth logic** | Signed URLs at the edge; no per-user rendering |

Score: 5/5. Nothing else in computing is this perfectly cache-shaped.

**The popularity long tail — the reason you don't push everything:**

```
Views
  │
  │██
  │██
  │███
  │████
  │██████
  │█████████
  │███████████████
  │██████████████████████████████████████████████████████████████████
  └───────────────────────────────────────────────────────────────────▶ Videos
   ◀── ~1% of videos ──▶ ◀────────────── the long tail (~99%) ──────────▶
   ~80-90% of all views              a handful of views each, forever
```

A tiny fraction of videos gets the overwhelming majority of views. So:

- **Hot content (the head):** proactively **pushed** to edge caches — sometimes before the video is even public, based on the channel's subscriber count and predicted demand. A Taylor Swift music video is on every edge server on earth within minutes of release.
- **Cold content (the tail):** **pulled** on demand. First viewer of an obscure 2011 vlog takes a cache miss; the edge fetches from a regional cache, which fetches from origin blob storage. That viewer waits ~300 ms extra. Nobody cares — there are three of them a year.
- **Warm middle:** standard LRU/LFU eviction at the edge sorts itself out.

**Servers inside the ISP — the economics endgame:**

Both **Netflix (Open Connect Appliances)** and **Google (Google Global Cache)** give ISPs free hardware — a rack of boxes stuffed with SSDs — and ask them to plug it into their network. Overnight, during off-peak hours, the boxes are filled with the content predicted to be popular in *that region tomorrow*.

The result: when you press play, the bytes come from a machine **inside your ISP's building**, a few milliseconds away. They never cross a paid transit link. The ISP loves it (their upstream bandwidth bill drops). Netflix/Google love it (their egress bill drops from billions to near-zero for that traffic). You love it (it starts in 500 ms and never buffers).

**That's the endgame of the bandwidth arithmetic from section 2.** When your dominant cost is moving bytes, you eventually stop renting the pipes and start owning the last mile.

---

### 7. Data model

**Here's the thing to say out loud: the metadata database is small and boring. The BYTES are the hard part.** Do not spend 15 minutes on the schema. Spend two, then get back to the pipeline.

**Metadata — PostgreSQL** (relational, modest volume, needs transactions on the upload state machine):

```sql
CREATE TABLE videos (
  id            UUID PRIMARY KEY,
  user_id       UUID NOT NULL,
  title         VARCHAR(200) NOT NULL,
  description   TEXT,
  status        VARCHAR(20) NOT NULL,  -- UPLOADING|PROCESSING|READY|FAILED|BLOCKED
  duration_sec  INT,
  visibility    VARCHAR(10) NOT NULL,  -- PUBLIC|UNLISTED|PRIVATE
  s3_key        TEXT NOT NULL,         -- raw source
  manifest_url  TEXT,                  -- CDN master.m3u8; NULL until READY
  thumbnail_url TEXT,
  created_at    TIMESTAMPTZ NOT NULL,
  published_at  TIMESTAMPTZ
);
CREATE INDEX ON videos (user_id, created_at DESC);   -- a channel's video list

-- Which renditions actually exist. Written incrementally, which is EXACTLY
-- the "360p available first, 1080p appears later" behaviour, in one table.
CREATE TABLE video_renditions (
  video_id      UUID NOT NULL REFERENCES videos(id),
  rendition     VARCHAR(10) NOT NULL,  -- '360p'
  codec         VARCHAR(20) NOT NULL,  -- 'h264' | 'vp9' | 'av1'
  bandwidth_bps INT NOT NULL,
  playlist_url  TEXT NOT NULL,
  segment_count INT NOT NULL,
  ready_at      TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (video_id, rendition, codec)
);

-- Pipeline job state. The DAG's memory. Idempotency lives here.
CREATE TABLE transcode_jobs (
  id            UUID PRIMARY KEY,
  video_id      UUID NOT NULL,
  job_type      VARCHAR(20) NOT NULL,  -- SPLIT|TRANSCODE|STITCH|PACKAGE|THUMBNAIL
  chunk_index   INT,
  rendition     VARCHAR(10),
  status        VARCHAR(20) NOT NULL,  -- PENDING|RUNNING|DONE|FAILED
  attempts      INT NOT NULL DEFAULT 0,
  output_key    TEXT,                  -- deterministic path → retries overwrite
  UNIQUE (video_id, job_type, chunk_index, rendition)   -- the idempotency guard
);
```

**Comments — Cassandra/DynamoDB** (write-heavy, no joins, partition by video):

```
comments:  PK = video_id,  SK = (created_at DESC, comment_id)
           { user_id, text, parent_comment_id, like_count }
```

**Views — never a row in Postgres.** See the deep dive below. Views land in **Kafka**, are aggregated by a stream/batch job, and the rolled-up count is cached in **Redis** and periodically written back.

---

### 8. High-level architecture — the big diagram

```
                                  ┌──────────────────────────────┐
                                  │           CLIENTS            │
                                  │  Web · iOS · Android · TV    │
                                  │  (hls.js / ExoPlayer / AVKit)│
                                  └───┬──────────────────────┬───┘
                                      │ metadata (small)     │ VIDEO BYTES (huge)
                                      ▼                      ▼
                          ┌───────────────────┐    ┌──────────────────────┐
                          │   API GATEWAY     │    │        CDN           │
                          │   (auth, rate     │    │  edge PoPs + caches  │
                          │    limit, LB)     │    │  INSIDE ISP networks │
                          └─────────┬─────────┘    └──────────┬───────────┘
        ┌──────────────┬────────────┼────────────┐            │ cache miss
        ▼              ▼            ▼            ▼            ▼ (~1% of reqs)
 ┌────────────┐ ┌────────────┐ ┌─────────┐ ┌──────────┐  ┌──────────────────┐
 │  UPLOAD    │ │  VIDEO     │ │ SEARCH  │ │  RECO    │  │   BLOB STORAGE   │
 │  SERVICE   │ │  METADATA  │ │ SERVICE │ │ SERVICE  │  │  raw/  encoded/  │
 │ (presigned │ │  SERVICE   │ │(Elastic)│ │          │  │  (multi-region,  │
 │   URLs)    │ │            │ │         │ │          │  │   tiered, EB)    │
 └─────┬──────┘ └─────┬──────┘ └────┬────┘ └────┬─────┘  └────────▲─────────┘
       │              │             │           │                 │
       │        ┌─────▼─────┐  ┌────▼────┐ ┌────▼─────┐           │
       │        │ Postgres  │  │  Index  │ │ Feature  │           │
       │        │(metadata) │  │         │ │  Store   │           │
       │        │  + Redis  │  └─────────┘ └────▲─────┘           │
       │        └───────────┘                   │                 │
       │                                        │                 │
       │  video.uploaded                 ┌──────┴──────┐          │
       ▼                                 │  BATCH /    │          │
 ┌──────────────────────┐                │  STREAM     │          │
 │   MESSAGE QUEUE      │                │  PROCESSING │          │
 │   (Kafka)            │                │ (Spark/Flink│          │
 │  video.uploaded      │◀───────────────│  view counts│          │
 │  video.viewed        │   view events  │  trending   │          │
 └──────────┬───────────┘   from players │  reco model)│          │
            │                            └─────────────┘          │
            ▼                                                     │
 ┌──────────────────────┐                                         │
 │  TRANSCODING         │  split → transcode chunks in PARALLEL   │
 │  ORCHESTRATOR (DAG)  │  → stitch → package → manifest ─────────┘
 │  + worker fleet      │                                    writes encoded/
 │  (~140k cores)       │                                    then warms CDN
 └──────────────────────┘
```

**Read the diagram as two completely separate paths, and say this in the interview:**

- **The WRITE path** (thin, slow, asynchronous, ~58 QPS): client → presigned URL → blob → queue → transcoding DAG → blob → CDN. It can take five minutes. Nobody is waiting.
- **The READ path** (fat, fast, ~99% of everything): client → CDN → done. It must be under 2 seconds to first frame, and it must never stall. It barely touches our servers at all.

The two paths share almost nothing. **That separation is the architecture.**

---

### 9. Deep dives

#### (a) View counting at scale — and the famous "301" bug

**The naive design, which is wrong:**

```js
// DO NOT DO THIS.
await db.query('UPDATE videos SET views = views + 1 WHERE id = $1', [videoId]);
```

A video goes viral. 500,000 people press play in the same minute. Every one of those statements needs a **row-level write lock on the same row**. They serialise. Each transaction waits for the previous one to commit — maybe 1 ms each with replication. You have created a queue with a throughput ceiling of ~1,000 writes/sec on a single row, and 8,000 requests/sec arriving. Connections pile up. The connection pool exhausts. The database becomes unavailable **for every other video too.** One popular cat video has taken down the site. This is called **hot-row contention** and it is the classic trap of this question.

**The real design:**

```
┌────────┐  fire-and-forget   ┌─────────┐   ┌──────────────────┐   ┌─────────┐
│ Player │ ─────────────────▶ │  Kafka  │ ─▶│ Flink / Spark    │ ─▶│  Redis  │
│        │  view event        │ (topic: │   │ Streaming:       │   │ counter │
└────────┘  {videoId, userId, │  views) │   │  windowed        │   └────┬────┘
             sessionId, ts,   └─────────┘   │  aggregation +   │        │ every
             watchedSec}                    │  bot filtering   │        │ 60 s
                                            └──────────────────┘        ▼
                                                                  ┌──────────┐
                                                                  │ Postgres │
                                                                  │ (durable │
                                                                  │  count)  │
                                                                  └──────────┘
```

1. The player **fires an event onto a stream** (Kafka). No database write. No lock. Kafka takes millions of appends per second happily — it's an append-only log.
2. A **stream/batch processing job** (recall [87 — Batch vs Stream Processing]) consumes the topic, **aggregates by videoId over a time window**, and increments a counter in Redis. One million individual events become **one** `INCRBY 1000000`.
3. Periodically (every 30–60 s) the rolled-up count is flushed to Postgres for durability, and the read path serves the count from Redis.

**The trade you're making, and you must name it:** the view count is now **approximate and slightly stale**. It might be 60 seconds behind. It might be off by a few thousand on a viral video. **And that is completely fine** — nobody's business depends on knowing whether a video has 8,412,003 or 8,412,441 views. You gave up exactness and bought unbounded write throughput. In exchange, watches never fail. *Correctness where it matters (billing, likes-by-you), approximation where it doesn't (aggregate counters).*

**The memorable detail — the 301 bug.** For years, YouTube view counts would famously **freeze at exactly "301"** on new videos and stay there for hours before jumping. Why? Because the fast, approximate path was allowed to run free up to ~300 views, and beyond that the count was held while the *slow, careful, anti-fraud* pipeline verified which views were real humans and which were bots inflating the number. The batch job's latency was visible in the UI. It's a perfect, real-world illustration of exactly this design: **an approximate fast path, a slow accurate batch path, and a visible seam between them.** Drop this in an interview and it lands every time.

#### (b) Resumable uploads

Covered in the code above, but the principles are worth naming:

- **Chunk it.** 5–10 MB parts. If part 37 of 250 fails on a train going through a tunnel, you redo 8 MB, not 2 GB.
- **Make parts independent.** They can be uploaded in parallel (3–5 at a time on mobile — more just causes TCP contention) and in any order.
- **Server tracks received parts, not bytes.** The `upload-status` endpoint is the resume point.
- **Client persists the `uploadId` locally.** If the app is killed, on restart it reads the uploadId from local storage, asks the server what landed, and resumes.
- **Expire abandoned uploads.** A lifecycle rule aborts multipart uploads older than 7 days, or you'll pay storage for orphaned parts forever. (This is a real, expensive, commonly-missed AWS bill leak.)

#### (c) Recommendations, conceptually

You will not be asked to design a ranking model. You *will* be asked for the shape. It's a two-stage funnel:

```
      2,000,000,000 videos in the catalogue
                   │
                   ▼  CANDIDATE GENERATION  (offline, BATCH — recall 87)
                   │  cheap, recall-oriented: collaborative filtering,
                   │  co-watch graph, subscriptions, trending in your region.
                   │  Precomputed nightly, stored per-user.
                   ▼
              ~500 candidates
                   │
                   ▼  RANKING  (online, per-request, ~50 ms budget)
                   │  expensive model scores each candidate with rich features:
                   │  watch history, session context, time of day, device,
                   │  predicted watch TIME (not clicks — that's how you avoid
                   │  optimising for clickbait).
                   ▼
              ~20 videos → the home feed
```

The key architectural point: **the expensive stuff runs offline in batch and the online path only ranks a few hundred things.** You could never score two billion videos in 50 ms. You never try.

#### (d) Search over metadata

Recall [79 — Search Systems]. Video *content* isn't searched; the **metadata** is: title, description, tags, channel name, and auto-generated **captions** (which is genuinely how YouTube can find a phrase spoken inside a video).

- On `status → READY`, publish a `video.indexable` event.
- An indexer consumer writes a document into **Elasticsearch**: `{ videoId, title, description, tags, channel, captions, viewCount, publishedAt, durationSec }`.
- Query = full-text relevance (BM25) **blended with popularity and recency signals**. A video that matches your query perfectly but has 3 views ranks below one that matches well and has 3 million.
- The index is a **derived, eventually-consistent view** of Postgres. If it lags 30 seconds, nobody dies. If it's lost entirely, you rebuild it from Postgres.

#### (e) Deduplication and copyright matching (fingerprinting)

Two problems, one technique.

**Dedup:** users re-upload the same video constantly. Storing it twice costs real money at exabyte scale. Compute a content hash of the raw file; if it exists, point the new `videos` row at the existing encoded assets and skip transcoding entirely. Saves both storage and ~140k cores of compute.

**Copyright matching (Content ID):** exact hashing is useless here — a pirate crops the frame, adds a border, shifts the pitch, changes the bitrate. Every byte differs. So instead you compute a **perceptual fingerprint**:

- **Video fingerprint:** sample frames, downscale to a tiny greyscale grid, hash the *structure* (e.g. a perceptual hash per frame). Visually similar frames produce *similar* hashes even after re-encoding.
- **Audio fingerprint:** the Shazam trick — build a spectrogram, find peak frequency landmarks, hash the (frequency, frequency, Δtime) triples. Extremely robust to noise and compression.

Then, on upload, look up the fingerprint sequence against an index of registered rights-holder content. A **matched subsequence** means "this 40-second clip of your upload is a song by X." The rights holder then chooses: block, monetise, or track. This runs asynchronously alongside transcoding, as another branch of the DAG.

---

### 10. Bottlenecks, cost, and failure modes

**Cost is a first-class design concern here.** For most systems you optimise latency and then check the bill. For video, **the bill *is* the design constraint.**

| Cost centre | Rough share | How the design fights it |
|---|---|---|
| **Egress bandwidth** | The overwhelming majority | Own CDN, servers inside ISPs, aggressive edge caching, better codecs (VP9/AV1 cut bytes 30–50% for the same quality), don't serve 1080p to a 5-inch screen |
| **Storage** | Large | Tier cold videos to cheap/archival storage; delete rarely-watched high renditions and re-generate on demand; dedup; don't 4K-encode a video with 12 views |
| **Transcoding compute** | Moderate | Chunked parallelism on **spot/preemptible** instances (jobs are idempotent, so eviction is free); custom ASICs; lazy-encode the top rungs only for videos that get traction |
| **Metadata DB** | Small | Genuinely boring. Read replicas + Redis. Don't over-engineer it. |

**Failure modes and what happens:**

| What fails | Blast radius | Mitigation |
|---|---|---|
| A transcode worker dies mid-chunk | One 6-second chunk | Job is redelivered, writes to a deterministic path, overwrites cleanly. **Idempotency is the whole answer.** |
| ffmpeg cannot decode a corrupt upload | One video | Retry N times, then `status = FAILED`, notify the creator. Do not retry forever — poison-message → DLQ. |
| The transcode queue backs up (viral upload day) | Uploads take hours to publish | Autoscale workers on queue depth. **Prioritise the low rungs** so videos become watchable early. Playback of *existing* videos is 100% unaffected — the paths are separate. |
| CDN edge PoP goes down | Users in one region | Anycast/DNS reroutes to the next-nearest PoP. Slightly higher latency, no outage. |
| **Blob storage region fails** | Catastrophic if unreplicated | Cross-region replication of `encoded/`. The raw source can be single-region + versioned (worst case, you re-transcode). |
| Metadata DB primary fails | No new uploads; watches keep working if the manifest URL is cached | Failover to replica. **Playback degrades gracefully because the CDN doesn't need the DB.** |
| A video goes mega-viral | Origin gets hammered by edge misses | Multi-tier CDN (edge → regional shield → origin) so origin sees a handful of requests, not millions. Pre-push predicted-hot content. |

**How to scale further:**
1. **Better codecs.** AV1 gives roughly the same quality at 30–50% fewer bits than H.264. At a $4B/year bandwidth bill, a 30% reduction is worth more than your entire engineering org. This is why Google poured resources into VP9 and AV1 — it is a *bandwidth cost* project, not a video-quality project.
2. **Per-title / per-shot encoding.** A talking-head video needs far fewer bits than a fireworks display. Netflix pioneered *per-title encoding*: analyse each video and build a **custom bitrate ladder** for it instead of a fixed one. Same perceived quality, meaningfully fewer bytes.
3. **Peer-assisted delivery / edge compute** for further offload.
4. **Lazy transcoding.** Don't build the full ladder for every upload. Encode 360p + 720p immediately; only build 1080p/4K once a video crosses a view threshold. Most videos never get there — you just saved most of your transcode fleet and a large slice of storage.

---

## Visual / Diagram description

The three diagrams you must be able to draw from memory are all above:

1. **The high-level architecture** (section 8). The thing to communicate: **two paths that barely touch.** Draw the write path going *down* the left (client → presigned URL → blob → queue → transcode DAG → blob) and the read path going *right* across the top (client → CDN → done). If your interviewer takes away one image, it should be that the read path — 99% of all traffic — mostly bypasses your servers entirely.

2. **The transcoding pipeline** (section 4). The thing to communicate: **fan-out.** One video → split into 1,200 chunks → 7,200 independent jobs → 1,000 workers → fan back in to stitch. Draw the fan-out explicitly with diverging arrows; that visual *is* the "hours become minutes" argument.

3. **The ABR switching timeline** (section 5). The thing to communicate: **the network line dives, the quality line follows it down, and the playback line never breaks.** Draw three horizontal tracks stacked vertically — bandwidth, chosen quality, playback — and make the playback track a single unbroken bar all the way across. That unbroken bar is the product.

---

## Real world examples

### YouTube

Ingests roughly 500 hours of video every minute. Transcodes every upload into a ladder spanning 144p to 4K/8K, in H.264 plus VP9 and increasingly AV1. Google built custom silicon — the **Argos VCU (Video Coding Unit)** — specifically because software transcoding at their volume was too expensive; Google has publicly stated the VCUs deliver order-of-magnitude efficiency gains over the CPU fleet they replaced. Delivery runs over **Google Global Cache** nodes that Google installs inside ISP networks worldwide. The "only 360p available right after upload" behaviour is directly observable and is exactly the progressive-availability pipeline described above.

### Netflix

The extreme case of "spend compute once, save bandwidth forever." Netflix pioneered **per-title encoding** (a bespoke bitrate ladder per video, since a cartoon and an action film compress completely differently) and later **per-shot encoding** (a different ladder for individual *scenes* within a film). Their **Open Connect Appliances** are physical servers Netflix gives to ISPs for free; they are filled overnight with content predicted to be popular in that region, so during peak hours the bytes originate metres from the subscriber. Netflix has said the overwhelming majority of its traffic is served from these appliances. Their client-side ABR work (notably the **BOLA** buffer-based algorithm) is published research.

### Twitch / live streaming

The instructive *contrast*. Live has the same transcoding-and-ABR shape but with a brutal extra constraint: **latency**. You cannot chunk a 2-hour file in advance because the file doesn't exist yet — you're transcoding a stream that is being created *right now*, and every second of buffering you add for safety is a second of delay between the streamer and their chat. Live systems therefore use much shorter segments (1–2 s, or chunked-transfer LL-HLS), transcode in real time (no chunk-parallelism available), and accept lower quality ceilings. Worth naming in an interview to show you know *why* the VOD design works: **VOD gets to be embarrassingly parallel precisely because the whole file already exists.**

---

## Trade-offs

**Transcode everything up front vs. transcode lazily**

| | Up front (full ladder) | Lazy (low rungs now, high rungs on demand) |
|---|---|---|
| Playback quality available | Immediately, all rungs | Top rungs missing for unpopular videos |
| Compute cost | Huge — you encode 4K for videos with 3 views | Far lower |
| Storage cost | Huge | Far lower |
| Complexity | Simple | Needs a popularity trigger + on-demand encode path |

**Client-side ABR vs. server-side quality selection**

| | Client-side (HLS/DASH — the industry answer) | Server-side |
|---|---|---|
| Cacheability | Perfect — static immutable files | Poor — personalised streams |
| Accuracy of bandwidth estimate | High (measures the real link) | Low (server can't see your WiFi) |
| Server complexity | Near zero | High, and stateful |
| Control over the algorithm | You ship a player and hope | Total |

**Segment length**

| | Short (2 s) | Long (10 s) |
|---|---|---|
| Reaction speed to bandwidth change | Fast — great for mobile | Slow — a stall can happen before you adapt |
| HTTP request overhead | High (5× the requests) | Low |
| Compression efficiency | Worse (more keyframes) | Better |
| Startup latency | Lower | Higher |

**Rule of thumb:** **6-second segments** for VOD (the industry default — good compression, acceptable adaptation speed), **2-second or shorter** for live/low-latency. And when in doubt on quality vs. continuity: **always sacrifice quality to protect continuity.** A soft picture is a minor annoyance; a spinning wheel is a lost user.

**The sweet spot:** transcode the middle rungs (360p/720p) eagerly for everything, build the expensive rungs on demand, ship a proven client player (hls.js / Shaka / ExoPlayer) rather than writing your own ABR algorithm, cache everything at the edge forever, and put your view counts on a stream instead of in a `SET views = views + 1`.

---

## Common interview questions on this topic

### Q1: "Why can't you just store the uploaded MP4 and serve it directly?"

**Hint:** Three reasons, and say all three. **(1) Compatibility** — the source codec/container may be unplayable on most devices, and software decoding kills phone batteries. **(2) Bitrate** — a 200 Mbps source cannot stream over a 2 Mbps mobile link, ever. **(3) Adaptability** — one monolithic file cannot change quality mid-playback, so any bandwidth drop becomes a buffering stall. Transcoding into a segmented ladder solves all three at once, and it makes every byte perfectly CDN-cacheable as a bonus.

### Q2: "A 2-hour video would take hours to transcode. How do you make it take minutes?"

**Hint:** **Chunk it and go embarrassingly parallel.** Split the source at keyframe/GOP boundaries into ~6-second chunks, so each chunk decodes independently. A 2-hour video is ~1,200 chunks × 6 renditions = ~7,200 independent jobs. Throw a thousand workers at them and it's done in a couple of minutes, then stitch and package. Then add: the jobs must be **idempotent** (deterministic output paths) so you can run them on cheap preemptible instances and just retry anything that dies.

### Q3: "How does the player know to drop from 1080p to 360p, and why doesn't it stall?"

**Hint:** **It's client-side ABR.** The video is thousands of small aligned segments encoded at every quality level, plus a manifest (`.m3u8`/`.mpd`) listing each level's `BANDWIDTH` and every segment URL. At each segment boundary the player compares its **measured throughput** and its **buffer level** against the ladder, and requests the next segment at whichever quality it can sustain. Buffer draining → drop fast. Buffer healthy and throughput comfortably above the next rung → climb slowly. Because segments are time-aligned, the switch is seamless. **The server is a dumb static file server; all the intelligence is in the client** — which is precisely what makes it 100% CDN-cacheable.

### Q4: "You have a viral video getting 500k views a minute. How do you count views?"

**Hint:** **Not with `UPDATE videos SET views = views + 1`** — that's hot-row contention and it will take out the database for every other video too. Instead fire view events onto **Kafka**, aggregate them with a **stream/batch job** into windowed counts, `INCRBY` a **Redis** counter, and periodically flush to Postgres. Name the trade explicitly: the count is now **approximate and slightly stale**, and that is a perfectly good trade because nobody's business depends on the last 400 views. Then land the killer detail: this is exactly why YouTube's counter visibly lags — and why it famously froze at **301**.

### Q5: "What's the single biggest cost in this system, and how does that shape the design?"

**Hint:** **Egress bandwidth**, by an enormous margin — on the order of billions of dollars a year at YouTube scale, dwarfing storage and compute combined. This is *why* a CDN isn't optional; *why* Google and Netflix physically install their own caching servers inside ISP networks (Google Global Cache, Netflix Open Connect); *why* they invest so heavily in better codecs (AV1 = 30–50% fewer bytes for the same quality = hundreds of millions of dollars); and *why* per-title encoding exists. In this system, **cost is a design constraint, not an afterthought.**

### Q6: "How would you change this design for live streaming?"

**Hint:** Same shape, one brutal new constraint: **you can't chunk a file that doesn't exist yet.** No chunk-parallelism, so you must transcode in real time (fewer renditions, hardware encoders). Segments shrink to 1–2 s (or LL-HLS chunked transfer) to cut glass-to-glass latency. The manifest becomes a *rolling window* (`#EXT-X-MEDIA-SEQUENCE` advances, old segments drop off) instead of a fixed `VOD` playlist. And the playlist can no longer be cached forever — only for a second or two. Everything else (ABR, CDN, client intelligence) is identical.

---

## Practice exercise

### "The Buffering Postmortem"

Spend 30-40 minutes. Produce a single page (paper or a doc) with four parts:

**Part 1 — The estimate (10 min).** From scratch, without looking back at section 2, derive the daily egress bandwidth for a video platform with **200 million hours watched per day at an average of 4 Mbps**. Show every line of arithmetic. Then price it at $0.01/GB. Write one sentence explaining what that number implies about where your servers must physically live.

**Part 2 — The pipeline (10 min).** Draw the transcoding DAG for a **90-minute** upload. Label: the segment length you chose, the number of chunks, the number of renditions, the total number of parallel jobs, and — assuming 800 workers and 15 seconds of CPU per job — how long the whole transcode takes. Mark clearly which step is the fan-out and which is the fan-in.

**Part 3 — The manifest (10 min).** By hand, write a valid `master.m3u8` advertising exactly **three** renditions (360p, 720p, 1080p) with sensible `BANDWIDTH` and `RESOLUTION` values, and a `720p/index.m3u8` containing the first **five** 6-second segments. It must be syntactically correct — check `#EXT-X-TARGETDURATION` and `#EXT-X-ENDLIST`.

**Part 4 — The trace (10 min).** A user starts at 1080p on office WiFi, then walks into an underground car park where throughput drops to 600 kbps for 30 seconds, then comes back out. Write a segment-by-segment trace (segment number → measured throughput → buffer level → quality chosen → why). **The playback column must never say "STALLED."** If it does, your ABR logic is wrong — fix it and explain what you changed.

**What to produce:** one page containing the arithmetic, the DAG sketch, the two hand-written manifests, and the trace table.

---

## Quick reference cheat sheet

- **The product requirement is "never buffer."** Every design decision descends from that one line. Users forgive a soft picture; they do not forgive a spinning wheel.
- **The API server must never touch video bytes.** Hand out **pre-signed URLs**; the client PUTs directly to blob storage. This is the single most important API decision in the system.
- **Uploads must be resumable.** Multipart, ~8 MB chunks, independent parts, a `upload-status` endpoint to find what's missing. A 2 GB mobile upload *will* fail partway.
- **Transcoding is mandatory**, for three reasons: device **compatibility**, streamable **bitrate**, and the segmented **structure** that makes ABR possible.
- **Chunk the video and transcode chunks in parallel.** ~1,200 chunks × 6 renditions = ~7,200 independent jobs. **Hours of serial work becomes minutes.** Cut on keyframe boundaries.
- **The pipeline is a DAG, and every job must be idempotent** — deterministic output paths so a retry safely overwrites. This is what lets you run the fleet on cheap preemptible instances.
- **`status = PROCESSING`, and low rungs publish first.** That's why a fresh YouTube upload is 360p-only for a few minutes. Real, observable proof the design is right.
- **The video is not one file.** It's thousands of aligned **segments** at multiple qualities, plus a **manifest** (`.m3u8` / `.mpd`) listing every rung's `BANDWIDTH` and every segment URL.
- **ABR intelligence lives in the CLIENT, not the server.** The player measures throughput and buffer level and picks the next segment's quality. **Drop fast, climb slow.**
- **Therefore the server is a dumb static file server** — which is exactly why the whole thing is infinitely CDN-cacheable. Say this sentence in the interview.
- **Egress bandwidth dwarfs everything.** Billions per year. It's why CDNs aren't optional, why Netflix/Google put boxes inside ISPs, and why better codecs (AV1) are a *finance* project.
- **Popularity is a long tail:** ~1% of videos get most of the views. **Push** hot content to the edge; **pull** the cold tail from origin on demand.
- **Never `UPDATE views = views + 1`.** Hot-row contention kills the DB. Events → **Kafka** → stream aggregation → **Redis** counter → periodic flush. Approximate and stale, and that's fine. (See: "301".)
- **The metadata DB is small and boring. The bytes are the hard part.** Don't spend interview time on the schema.
- **Cost is a first-class design constraint here**, not an afterthought. Storage and egress dominate; lazy-encode the expensive rungs.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [100 — Design Instagram](./100-hld-instagram.md) — the other great media system; same blob-storage and CDN spine, but images are trivial next to video, so this doc is Instagram plus the two hard parts (transcoding and ABR) |
| **Next** | [Continue with the next case study in the curriculum] |
| **Related** | [60 — CDN](./60-cdn.md) — video segments are the ultimate CDN content: static, immutable, huge, and requested by millions. This doc is the "why the CDN exists" story told in full |
| **Related** | [78 — Blob Storage](./78-blob-storage.md) — where every raw upload and every encoded segment physically lives, and why the bytes never go through your API server |
| **Related** | [67 — Message Queues](./67-message-queues.md) — the queue between "upload finished" and "transcoding starts" is what makes a five-minute pipeline acceptable behind a two-second API response |
| **Related** | [87 — Batch vs Stream Processing](./87-batch-vs-stream-processing.md) — view counting, trending, and the offline half of the recommendation pipeline all live here. It's the reason your view count is approximate |
