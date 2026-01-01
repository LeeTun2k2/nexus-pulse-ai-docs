# Functional Specifications for AI Content Generation System

## 1. Overview

- Generate content from simple inputs (prompt, text, link, upload).
- Simple and easy to use.
- Web-based platform.

## 2. Key Functional Requirements

- **FR1: Image Generation**
  - Input: Text prompt (e.g., "Cosmetic product photo on white background"), or batch from CSV/product list.
  - Output: High-resolution images (up to 4K) with options like background removal, AI shadows, model try-on.
  - **Optimistic UI**: BlurHash placeholders load instantly before full-resolution images, providing seamless user experience.
  - Sub-features: Batch processing (up to 50 images per batch); E-commerce integration (auto-pull product images).

- **FR2: Video Generation from Prompt**
  - Input: Text prompt (e.g., "Create ad video for red t-shirt product"), product link (e.g., Shopify URL), or upload image/video.
  - Output: Short videos (5-60s) with animation, text overlay, background music.
  - **First Frame Preview**: Must return first frame preview within < 5s for instant user feedback while full video generates.
  - Sub-features: Templates (holiday, product demo); Multi-language support (English, Vietnamese).

- **FR3: Text-to-Voice (Voice-over Generation)**
  - Input: Text script, select avatar (AI-generated face), voice style (male/female, accents).
  - Output: Audio file or direct integration into video (lip-sync).
  - Sub-features: Diverse voices (10+ languages); Custom avatars from user upload.

- **FR4: Additional Tools**
  - Editor: In-app video/image editor (crop, add text, effects).
  - Publishing: Auto-post to social media (TikTok, Facebook, Instagram); Scheduling calendar.
  - Analytics: Track views and engagement for generated content.

## 3. Use Cases

- **UC1: Batch Generate Images**
  - Actor: Marketer.
  - Steps: Upload product CSV → Set prompt template → Generate batch → Review & download.

- **UC2: Create Product Video**
  - Actor: E-commerce seller.
  - Steps: Login → Enter product link → Select template → Generate → Edit → Export/Download.
  - Post-condition: Video ready to share.

- **UC3: Add Voice to Video**
  - Actor: Creator.
  - Steps: Upload text script → Choose avatar/voice → Generate audio → Sync with video.

## 4. Integration Requirements

- API: RESTful APIs for third-party integration (e.g., Shopify webhook to auto-generate content when adding products).
- Authentication: OAuth2, Google/Email login.

## 5. Data Flow

### 5.1. Real-time Progress Updates (SSE)
- User submits generation request → API validates and creates job → **Server-Sent Events (SSE) stream** provides real-time progress updates
- SSE endpoint: `/api/generate/{job_id}/stream`
- Progress events include: status updates, percentage completion, preview frames, error notifications
- **Replaces polling mechanism** for better UX and reduced server load

### 5.2. Media Generation Flow
- User input → AI models (e.g., integrate Stable Diffusion for images, ElevenLabs for voice, or open-source alternatives) → Output storage (cloud bucket) → User download/post.
- **BlurHash Generation**: Worker generates BlurHash for images immediately → Stored in DB → Client loads placeholder instantly → Replaces with full image when ready
- **First Frame Preview**: For video generation, worker extracts first frame → Uploads to S3 → Returns preview URL to client within < 5s → Client displays preview while full video generates

### 5.3. Optimistic UI Pattern
- **Image Lists**: Load BlurHash strings from DB → Display placeholders instantly → Lazy load full images from S3
- **Video Generation**: Show first frame preview immediately → Display progress via SSE → Replace with full video when complete
- **Goal**: Minimize perceived latency, provide instant feedback (< 100ms from click to visual response)

TODO: review
