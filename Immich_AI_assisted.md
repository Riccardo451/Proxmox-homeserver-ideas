# AI-Assisted Photo Curation with Immich on Proxmox

## Goal

Turn a large Apple Photos archive (~1.5 TB) into a curated collection without manually reviewing every image.

Objectives:

* Remove duplicates and near-duplicates
* Identify blurry and low-quality photos
* Surface the best photo from similar groups
* Build "Life Highlights" and "Top Photos" albums
* Never automatically delete originals

---

# Architecture

```text
Apple Photos (Master Archive)
        |
        v
Export Originals
        |
        v
Immich
        |
        +--> Face Recognition
        +--> Smart Search
        +--> Duplicate Detection
        |
        v
AI Pipeline
        |
        +--> CLIP Embeddings
        +--> Similarity Clustering
        +--> Aesthetic Scoring
        +--> Blur Detection
        |
        v
Generated Albums
        |
        +--> Life Highlights
        +--> Top 5%
        +--> Duplicate Candidates
        +--> Blur Candidates
```

---

# Hardware

## Host

Lenovo ThinkCentre M720q

Recommended:

* Intel i5/i7 8th or 9th Gen
* 32 GB RAM
* SSD for VMs
* Large HDD/NAS storage for photos

## Hypervisor

* Proxmox VE 8.x

---

# VM Layout

## VM 101 - Immich

Purpose:

* Photo storage
* Search
* Face recognition
* Duplicate detection

Resources:

```text
CPU: 4-6 vCPU
RAM: 8-16 GB
Disk: 100 GB SSD
Storage: mounted photo dataset
```

---

## VM 102 - AI Processing

Purpose:

* CLIP embeddings
* Similarity search
* Aesthetic scoring
* Blur detection

Resources:

```text
CPU: 4 vCPU
RAM: 16 GB
Disk: 50 GB SSD
```

---

# Step 1 - Deploy Immich

Install Debian 12.

Install Docker:

```bash
apt update
apt install docker.io docker-compose-plugin -y
```

Clone Immich:

```bash
mkdir /opt/immich
cd /opt/immich
```

Download current compose files from Immich documentation.

Start:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Open:

```text
http://IMMICH_IP:2283
```

Create admin account.

---

# Step 2 - Configure Machine Learning

Enable:

* Facial Recognition
* Smart Search
* Duplicate Detection

Admin Settings → Machine Learning

Allow all jobs to complete before importing additional photos.

---

# Step 3 - Import Photos

DO NOT import all 1.5 TB immediately.

Start with:

```text
Recent Year
or
50,000 Photos
```

Verify:

* Storage works
* Thumbnails generate
* ML jobs complete

Then import remaining years.

---

# Step 4 - Install Immich-Deduper

Purpose:

* Find visually similar images
* Select preferred images
* Move duplicates to trash

Project:

https://github.com/agnet/immich-deduper

Run against Immich API.

Configure:

```yaml
delete: false
trash_duplicates: true
```

Never permanently delete.

---

# Step 5 - Build AI Processing VM

Install Debian 12.

Install packages:

```bash
apt update
apt install python3 python3-pip git -y
```

Create virtual environment:

```bash
python3 -m venv ~/venv
source ~/venv/bin/activate
```

Install:

```bash
pip install torch torchvision
pip install open_clip_torch
pip install pillow
pip install numpy
pip install pandas
pip install qdrant-client
pip install opencv-python
pip install transformers
```

---

# Step 6 - Deploy Qdrant

Qdrant stores image vectors.

Run:

```bash
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  qdrant/qdrant
```

Verify:

```text
http://AI_VM_IP:6333/dashboard
```

---

# Step 7 - Generate CLIP Embeddings

For every image:

```python
embedding = clip.encode_image(image)
```

Store:

```text
image_id
embedding_vector
```

inside Qdrant.

Benefits:

* Similarity search
* Duplicate discovery
* Event clustering

---

# Step 8 - Cluster Similar Images

Example:

```text
IMG_1001
IMG_1002
IMG_1003
IMG_1004
```

All belong to the same burst.

Create cluster:

```text
Cluster_001
```

Select best candidate later.

---

# Step 9 - Aesthetic Scoring

Use:

* CLIP aesthetic models
* LAION aesthetic predictor

Example output:

```text
IMG_1001 = 8.4
IMG_1002 = 7.1
IMG_1003 = 9.2
IMG_1004 = 5.9
```

Store scores in database.

---

# Step 10 - Blur Detection

Using OpenCV:

```python
variance = cv2.Laplacian(
    image,
    cv2.CV_64F
).var()
```

Low value:

```text
candidate_blur = true
```

Add to review list.

Never delete automatically.

---

# Step 11 - Select Winners

Inside each similarity cluster:

```text
IMG_1001 7.5
IMG_1002 8.9
IMG_1003 6.8
IMG_1004 5.2
```

Winner:

```text
IMG_1002
```

Other images:

```text
Review Candidate
```

---

# Step 12 - Create Albums

Generate:

```text
Life Highlights
```

Top 1%

```text
Top Photos
```

Top 5%

```text
Best Family
```

People-focused

```text
Best Travel
```

Location-focused

```text
Blur Candidates
```

Technical rejects

```text
Duplicate Candidates
```

Review queue

---

# Step 13 - Human Review

This is mandatory.

Rules:

* Never auto-delete
* Never trust scores blindly
* Never delete family photos solely due to low score

Workflow:

```text
Life Highlights
    ->
Favorite

Duplicate Candidates
    ->
Review
    ->
Trash

Blur Candidates
    ->
Review
    ->
Trash
```

---

# Recommended Workflow

Phase 1

* Import archive
* Run Immich ML jobs

Phase 2

* Run Immich-Deduper
* Remove obvious duplicates

Phase 3

* Generate embeddings
* Build similarity clusters

Phase 4

* Score aesthetics

Phase 5

* Generate albums

Phase 6

* Manually review top candidates

---

# Important Rule

Treat Apple Photos as the master archive for at least 6 months.

Do not permanently delete originals until:

* Immich migration is stable
* Duplicates have been reviewed
* Curated albums have been validated

Storage is cheaper than recovering lost memories.
