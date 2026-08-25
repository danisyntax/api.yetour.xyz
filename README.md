# Yetour API

The public API for [Yetour](https://yetour.xyz): search timestamped live performances and create a temporary download link for a selected song clip.

**Base URL:** `https://api.yetour.xyz`

All responses are formatted JSON and include `Access-Control-Allow-Origin: *`, so they can be used from browser projects.

## Quick start

Search for a song:

```sh
curl 'https://api.yetour.xyz/v1/search?song=heartless&year=2026'
```

Each result has an `id`. Request it to receive a temporary download URL:

```sh
curl 'https://api.yetour.xyz/v1/clips/2026.1.0'
```

Open the returned `downloadUrl` within five minutes to begin downloading the clip.

## Endpoints

### Search performances

```text
GET /v1/search?song=<query>&year=<year>
```

| Parameter | Required | Description |
| --- | --- | --- |
| `song` | Yes | Song title or partial title. Matching ignores apostrophe and punctuation differences, and words can be in any order. |
| `year` | No | Catalog year: `2021`, `2023`, `2025`, or `2026`. Defaults to `2026`. |

Example:

```text
https://api.yetour.xyz/v1/search?song=can%27t%20tell%20me%20nothing&year=2026
```

```json
{
  "query": "can't tell me nothing",
  "year": "2026",
  "results": [
    {
      "id": "2026.1.1",
      "song": "Can't Tell Me Nothing",
      "showId": 1,
      "showName": "MEXICO CITY NIGHT 1",
      "startSeconds": 628,
      "endSeconds": 840,
      "durationSeconds": 212,
      "endEstimated": false
    }
  ]
}
```

The clip `id` is stable: catalog year, show ID, and timestamp index.

### Request a temporary clip link

```text
GET /v1/clips/<id>
```

```text
https://api.yetour.xyz/v1/clips/2026.1.1
```

```json
{
  "clip": {
    "id": "2026.1.1",
    "song": "Can't Tell Me Nothing",
    "showName": "MEXICO CITY NIGHT 1",
    "startSeconds": 628,
    "endSeconds": 840
  },
  "downloadUrl": "https://api.yetour.xyz/v1/downloads/2026.1.1?expires=...&signature=...",
  "expiresAt": "2026-08-25T15:32:19.000Z",
  "expiresInSeconds": 300
}
```

`downloadUrl` is signed and valid for five minutes. Start the download before it expires; an already-started download continues streaming.

### Download a clip

```text
GET /v1/downloads/<id>?expires=<unix-timestamp>&signature=<signature>
```

Use the exact `downloadUrl` from the clip endpoint. It streams an MP4 or WebM file with a download filename. Modified or expired links return `403`.

### Health check

```text
GET /healthz
```

```json
{ "status": "ok" }
```

## JavaScript example

```js
const query = encodeURIComponent("Can't Tell Me Nothing");
const search = await fetch(`https://api.yetour.xyz/v1/search?song=${query}&year=2026`)
  .then((response) => response.json());

const firstClip = search.results[0];
const link = await fetch(`https://api.yetour.xyz/v1/clips/${firstClip.id}`)
  .then((response) => response.json());

window.location.href = link.downloadUrl;
```

Always use `encodeURIComponent()` for song queries. Spaces need to be URL-encoded (usually `%20`) and an apostrophe as `%27`.

## Errors

| Status | Meaning |
| --- | --- |
| `400` | A required or valid search parameter was not provided. |
| `403` | A temporary download URL is invalid or expired. Request a fresh link from `/v1/clips/<id>`. |
| `404` | The endpoint, catalog year, or clip ID does not exist. |
| `429` | The clipper is busy. Wait for `Retry-After`, then try again. |
| `503` | The catalog or clip service is temporarily unavailable. |

## Notes

- Clips are capped at ten minutes.
- `endEstimated: true` means the clip is the final timestamp in a source video, so the endpoint uses the ten-minute cap instead of a following timestamp.
- Please link back to [Yetour](https://yetour.xyz) when using the API in a public project.

## Development

This repository is a Cloudflare Worker. Deployment configuration is in [`wrangler.jsonc`](./wrangler.jsonc).
