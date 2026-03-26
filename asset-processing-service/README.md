# Asset Processing Service

This service is the asynchronous worker for the AI marketing platform. It polls the Next.js API for pending asset-processing jobs, reads uploaded files, extracts or transcribes content, and writes the processed content back to the main app.

## Responsibilities

- Poll for pending and failed asset-processing jobs
- Detect stale jobs by heartbeat timeout
- Fetch asset metadata and the uploaded file itself
- Read text and markdown files directly
- Extract audio from video files with FFmpeg
- Split large audio files into chunks
- Transcribe audio chunks with OpenAI Whisper
- Update the asset content and token count in the Next.js app
- Mark jobs as completed, failed, or max-attempts-exceeded

## Runtime Flow

1. `main.py` starts a fetcher task plus a configurable number of worker tasks.
2. The fetcher polls `/api/asset-processing-job` and queues eligible jobs.
3. Each worker locks a job id, updates its status to `in_progress`, and starts a heartbeat loop.
4. `job_processor.py` fetches the associated asset and file contents.
5. Depending on file type:
   - text and markdown files are decoded directly
   - audio files are split and transcribed
   - video files have audio extracted first, then split and transcribed
6. The worker updates `/api/asset` with extracted content and token count.
7. The worker marks the job complete or failed.

## Requirements

- Python 3.9+
- Poetry
- FFmpeg installed and available on `PATH`
- Reachable Next.js API server
- Matching `SERVER_API_KEY` shared with the Next.js app
- OpenAI API key for transcription

## Installation

```bash
cd asset-processing-service
poetry install
```

## Environment Variables

Create `asset-processing-service/.env`.

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `SERVER_API_KEY` | Yes | None | Bearer token for protected Next.js worker routes |
| `OPENAI_API_KEY` | Yes | None | Used by the Python OpenAI SDK for audio transcription |
| `API_BASE_URL` | No | `http://localhost:3000/api` | Base URL for the Next.js API |
| `STUCK_JOB_THRESHOLD_SECONDS` | No | `30` | Maximum age of heartbeat before the job is treated as stuck |
| `MAX_JOB_ATTEMPTS` | No | `3` | Number of times a failed job may be retried |
| `MAX_NUM_WORKERS` | No | `2` | Number of concurrent worker coroutines |
| `HEARTBEAT_INTERVAL_SECONDS` | No | `10` | How often a running job posts a heartbeat |
| `MAX_CHUNK_SIZE_BYTES` | No | `25165824` | Maximum chunk size for audio transcription |
| `OPENAI_MODEL` | No | `whisper-1` | OpenAI transcription model |

Example:

```env
SERVER_API_KEY=replace-me
OPENAI_API_KEY=replace-me
API_BASE_URL=http://localhost:3000/api
STUCK_JOB_THRESHOLD_SECONDS=30
MAX_JOB_ATTEMPTS=3
MAX_NUM_WORKERS=2
HEARTBEAT_INTERVAL_SECONDS=10
MAX_CHUNK_SIZE_BYTES=25165824
OPENAI_MODEL=whisper-1
```

## Running

```bash
cd asset-processing-service
poetry run asset-processing-service
```

## Build

```bash
cd asset-processing-service
poetry build
```

## Service-to-App API Contract

The worker calls these Next.js endpoints:

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/asset-processing-job` | Fetch non-terminal jobs |
| `PATCH` | `/api/asset-processing-job?jobId=...` | Update status, attempts, heartbeat, and error state |
| `GET` | `/api/asset?assetId=...` | Fetch asset metadata |
| `PATCH` | `/api/asset?assetId=...` | Save extracted content and token counts |

All of these calls use `Authorization: Bearer <SERVER_API_KEY>`.

## File Processing Details

- Text and markdown: file bytes are decoded as UTF-8.
- Audio: files are converted to MP3 when needed, split by estimated duration, then transcribed chunk by chunk.
- Video: audio is extracted with FFmpeg before the same chunking and transcription process.

## Key Modules

- `asset_processing_service/main.py`: event loop, fetcher, and workers
- `asset_processing_service/job_processor.py`: job orchestration and status updates
- `asset_processing_service/api_client.py`: HTTP client for the Next.js API
- `asset_processing_service/media_processor.py`: FFmpeg and OpenAI transcription pipeline
- `asset_processing_service/config.py`: environment loading and runtime settings
- `asset_processing_service/models.py`: Pydantic models for API payloads

## Operational Notes

- The service assumes the Next.js app is already running and reachable at `API_BASE_URL`.
- If FFmpeg is missing, video and most audio processing will fail.
- If `OPENAI_API_KEY` is missing, transcription requests will fail when the worker tries to process audio or video.
- Jobs are retried until `MAX_JOB_ATTEMPTS` is exceeded.
