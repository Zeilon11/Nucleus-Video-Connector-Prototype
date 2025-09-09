---
# Nucleus Video Connector

A blueprint for an integration application to streamline video workflows between Nucleus and an Online Video Platform (OVP). This repository provides a two-step API retrieval method for video assets and their metadata, along with potential ingestion workflows.

### Introduction

Nucleus, developed by BAFTA, is a management system for awards, grants, and initiatives. This repository explores potential methods for connecting it to an OVP to automate the transfer of video files submitted through Nucleus. The proposed solution is based on a secure, two-step API retrieval process that fetches video files and their associated metadata.

---
## API Blueprint

Both API steps require authentication using a Nucleus Base URL, an Award ID, an API Key ID, and a generated signature with an expiration timestamp.

### 1. Retrieve Entry Metadata

The first step is to use the `/api/entries.php` endpoint to retrieve entry details. This API call provides vital information, including the `entryTitle` and, crucially, the `mediaItem` ID for the video. This unique ID links the entry to its corresponding video file within the Nucleus system.

```bash
curl -X POST "<YOUR_NUCLEUS_BASE_URL>/api/entries.php" \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "awardId=<YOUR_AWARD_ID>&entryId=<YOUR_ENTRY_ID>&expires=<GENERATED_EXPIRES_TIMESTAMP>&keyId=<YOUR_API_KEY_ID>&signature=<GENERATED_SIGNATURE>"
```

The JSON response will contain a field, such as `data.Film file`, which holds the format `mediaItem:video:<MEDIA_ITEM_ID>`. The `MEDIA_ITEM_ID` must be extracted for the subsequent step.

### 2. Get the MP4 Video URL

Once the `mediaItem` ID is obtained, use the `/api/mediaItem.php` endpoint. By specifying `type=download`, the direct URL to the transcoded MP4 video file can be retrieved. This call also necessitates a `viewerId` for logging and permissions.

```bash
curl -X POST "<YOUR_NUCLEUS_BASE_URL>/api/mediaItem.php" \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "mediaItemId=<MEDIA_ITEM_ID>&type=download&viewerId=<YOUR_VIEWER_ID>&expires=<GENERATED_EXPIRES_TIMESTAMP>&keyId=<YOUR_API_KEY_ID>&signature=<GENERATED_SIGNATURE>"
```

The response is a JSON object containing the video URL, for example: `https://d3cszrd24af8ic.cloudfront.net/path/to/your/video.mp4`.

---
## Ingestion Workflows

An integration application—a dedicated script, for example—can serve as the bridge between Nucleus and the OVP. Once the video URL is retrieved, two primary methods can be used to ingest the video into the OVP.

### Option A: Push via TUS Upload

The integration application downloads the video file stream from the Nucleus API and then uses a **resumable upload protocol**, such as TUS, to upload the video to the OVP's API. This method is reliable for large files and ensures the upload can be resumed if interrupted.

### Option B: Pull via Direct URL Ingestion

This method is simpler if the OVP supports it. The integration application requests the OVP to ingest the video directly from the Nucleus URL. The OVP's servers then pull the asset directly from the Nucleus API. This approach offloads the file transfer burden from the integration application.

I've previously documented both methods here:

* Velazquez, J. (2025). *TUS-resumable File Uploads (TUS)*. https://doi.org/10.5281/zenodo.15034989
* Velazquez, J. (2025). *Understanding video pull uploads by referencing an external URL*. https://doi.org/10.5281/zenodo.15286575

---
## Workflow Diagram

<img width="1025" height="739" alt="image" src="https://github.com/user-attachments/assets/45f55413-0438-478b-afd8-a6140cb684fa" />


The workflow logic for this integration could follow these steps:

1.  An admin or scheduler triggers a sync.
2.  The integration application queries the Nucleus API for new or updated assets and their metadata.
3.  For each identified asset, the application maps the Nucleus metadata to the required OVP format.
4.  The application ingests the video using either a **Push (TUS)** or **Pull (Direct URL)** method.
5.  After successful ingestion, the application uses further API calls to apply additional metadata or settings (e.g., privacy controls).
6.  Once all assets have been processed, the integration reports its status.
