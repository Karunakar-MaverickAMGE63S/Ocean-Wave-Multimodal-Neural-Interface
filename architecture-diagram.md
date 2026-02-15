# Ocean Wave - AWS Architecture Diagram

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                            USER DEVICE                                      │
│                         (Mobile/Tablet/Desktop)                             │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                                                                       │ │
│  │                    React PWA (Progressive Web App)                    │ │
│  │                                                                       │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │ │
│  │  │   Camera/    │  │  UI Layer    │  │   Audio Player           │  │ │
│  │  │   Video      │  │  (React)     │  │   (Web Audio API)        │  │ │
│  │  │   Capture    │  │              │  │                          │  │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────▲───────────────────┘  │ │
│  │         │                 │                  │                       │ │
│  │         │ Image/Video     │ GraphQL          │ Audio Chunks          │ │
│  │         │ Frames          │ Mutations        │ (Base64 MP3)          │ │
│  │         │                 │                  │                       │ │
│  └─────────┼─────────────────┼──────────────────┼───────────────────────┘ │
│            │                 │                  │                         │
└────────────┼─────────────────┼──────────────────┼─────────────────────────┘
             │                 │                  │
             │                 │                  │
             │    HTTPS/WSS    │                  │
             │    (Secure)     │                  │
             ▼                 ▼                  │
┌─────────────────────────────────────────────────┼─────────────────────────┐
│                                                 │                         │
│                        AWS AMPLIFY              │                         │
│                    (Hosting & CI/CD)            │                         │
│                                                 │                         │
│  • Static Asset Hosting                         │                         │
│  • HTTPS/SSL Termination                        │                         │
│  • CDN Distribution                             │                         │
│  • CI/CD Pipeline                               │                         │
│                                                 │                         │
└─────────────────────────────────────────────────┼─────────────────────────┘
                                                  │
                                                  │
                                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                           AWS APPSYNC                                       │
│                    (GraphQL + WebSocket API)                                │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                                                                       │ │
│  │  GraphQL Schema:                                                      │ │
│  │                                                                       │ │
│  │  Mutations:                          Subscriptions:                  │ │
│  │  • processImage()                    • onVisionResult()              │ │
│  │  • processVideoFrame()               • onAudioStream()               │ │
│  │  • synthesizeSpeech()                                                │ │
│  │                                                                       │ │
│  │  Authentication: API Key                                             │ │
│  │  Protocol: WebSocket (WSS) for real-time streaming                   │ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└──────────────────┬────────────────────────────────────┬─────────────────────┘
                   │                                    │
                   │ Invoke Lambda                      │ Invoke Lambda
                   │ (Vision Processing)                │ (Speech Synthesis)
                   ▼                                    ▼
┌──────────────────────────────────────┐  ┌────────────────────────────────────┐
│                                      │  │                                    │
│    AWS LAMBDA                        │  │    AWS LAMBDA                      │
│    VisionProcessor Function          │  │    SpeechSynthesizer Function      │
│                                      │  │                                    │
│  • Runtime: Python 3.11              │  │  • Runtime: Python 3.11            │
│  • Memory: 1024 MB                   │  │  • Memory: 512 MB                  │
│  • Timeout: 30 seconds               │  │  • Timeout: 15 seconds             │
│                                      │  │                                    │
│  Process:                            │  │  Process:                          │
│  1. Decode base64 image              │  │  1. Receive description text       │
│  2. Build child-friendly prompt      │  │  2. Build SSML with rate control   │
│  3. Call Bedrock API                 │  │  3. Call Polly API                 │
│  4. Parse description                │  │  4. Stream audio in 4KB chunks     │
│  5. Return via AppSync               │  │  5. Return via AppSync             │
│                                      │  │                                    │
│  IAM Role:                           │  │  IAM Role:                         │
│  • bedrock:InvokeModel               │  │  • polly:SynthesizeSpeech          │
│                                      │  │                                    │
└──────────────┬───────────────────────┘  └────────────┬───────────────────────┘
               │                                       │
               │ API Call                              │ API Call
               │                                       │
               ▼                                       ▼
┌──────────────────────────────────────┐  ┌────────────────────────────────────┐
│                                      │  │                                    │
│       AMAZON BEDROCK                 │  │       AMAZON POLLY                 │
│    Claude 3.5 Sonnet v2              │  │       Neural Engine                │
│                                      │  │                                    │
│  Model ID:                           │  │  Voices:                           │
│  anthropic.claude-3-5-sonnet-        │  │  • Joanna (US English)             │
│  20241022-v2:0                       │  │  • Matthew (US English)            │
│                                      │  │  • Ivy (US English - Child)        │
│  Capabilities:                       │  │  • Aditi (Indian English)          │
│  • Multimodal vision analysis        │  │  • Raveena (Indian English)        │
│  • Scene understanding               │  │                                    │
│  • Object detection                  │  │  Features:                         │
│  • Spatial relationships             │  │  • Neural TTS engine               │
│  • Child-friendly descriptions       │  │  • Expressive, natural voices      │
│                                      │  │  • SSML support                    │
│  Input:                              │  │  • Speech rate control             │
│  • Base64 encoded image              │  │  • MP3 output format               │
│  • Text prompt                       │  │                                    │
│                                      │  │  Output:                           │
│  Output:                             │  │  • Streaming audio chunks          │
│  • Scene description text            │  │  • Base64 encoded MP3              │
│  • Confidence score                  │  │                                    │
│                                      │  │                                    │
└──────────────────────────────────────┘  └────────────────────────────────────┘
```

## Data Flow Sequences

### 1. Image Analysis Flow (Sighted Guide)

```
User → Camera Capture → React PWA
                           │
                           │ 1. Capture image
                           │ 2. Validate (format, size)
                           │ 3. Convert to base64
                           │
                           ▼
                    AWS AppSync (WSS)
                           │
                           │ Mutation: processImage()
                           │ - image: base64 string
                           │ - detailLevel: SIMPLE/MODERATE/DETAILED
                           │ - childFriendly: true
                           │
                           ▼
              Lambda: VisionProcessor
                           │
                           │ 1. Decode base64
                           │ 2. Build prompt based on detail level
                           │ 3. Invoke Bedrock API
                           │
                           ▼
              Amazon Bedrock (Claude 3.5 Sonnet)
                           │
                           │ Vision Analysis:
                           │ - Identify objects
                           │ - Detect people & actions
                           │ - Understand spatial relationships
                           │ - Generate child-friendly description
                           │
                           ▼
              Lambda: VisionProcessor
                           │
                           │ Return: VisionResult
                           │ - description: "A happy dog playing..."
                           │ - confidence: 0.95
                           │ - processingTime: 1.2s
                           │
                           ▼
                    AWS AppSync (WSS)
                           │
                           │ Subscription: onVisionResult()
                           │ Push to client in real-time
                           │
                           ▼
                      React PWA
                           │
                           │ 1. Display description
                           │ 2. Trigger speech synthesis
                           │
                           └──────────────────────────────────┐
                                                              │
                                                              ▼
                                                    (Continue to Speech Flow)
```

### 2. Video Streaming Flow

```
User → Activate Video Mode → React PWA
                                 │
                                 │ 1. Start MediaStream
                                 │ 2. Capture frames at 1-5 fps
                                 │ 3. Throttle processing
                                 │
                                 ▼
                          AWS AppSync (WSS)
                                 │
                                 │ Mutation: processVideoFrame()
                                 │ (Continuous stream)
                                 │ - frame: base64 string
                                 │ - frameNumber: int
                                 │ - detailLevel: SIMPLE
                                 │
                                 ▼
                    Lambda: VisionProcessor
                                 │
                                 │ Process each frame
                                 │ (Same as image flow)
                                 │
                                 ▼
                    Amazon Bedrock
                                 │
                                 │ Real-time analysis
                                 │ < 500ms latency target
                                 │
                                 ▼
                          AWS AppSync (WSS)
                                 │
                                 │ Subscription: onVisionResult()
                                 │ Stream descriptions continuously
                                 │
                                 ▼
                            React PWA
                                 │
                                 │ 1. Display real-time descriptions
                                 │ 2. Show connection quality metrics
                                 │ 3. Trigger speech for each frame
```

### 3. Speech Synthesis Flow (Eloquent Voice)

```
React PWA (receives description)
         │
         │ Automatic trigger
         │
         ▼
  AWS AppSync (WSS)
         │
         │ Mutation: synthesizeSpeech()
         │ - text: "A happy dog playing in the park"
         │ - voiceId: "Joanna" (or Ivy for child voice)
         │ - engine: NEURAL
         │ - rate: NORMAL/SLOW/FAST
         │
         ▼
Lambda: SpeechSynthesizer
         │
         │ 1. Build SSML with rate control
         │ 2. Invoke Polly API
         │
         ▼
  Amazon Polly (Neural Engine)
         │
         │ Text-to-Speech Synthesis:
         │ - Neural voice processing
         │ - Expressive, natural output
         │ - Child-friendly tone
         │
         ▼
Lambda: SpeechSynthesizer
         │
         │ 1. Receive audio stream
         │ 2. Chunk into 4KB pieces
         │ 3. Base64 encode each chunk
         │ 4. Return with metadata
         │
         ▼
  AWS AppSync (WSS)
         │
         │ Subscription: onAudioStream()
         │ Stream audio chunks in real-time
         │ - chunk 1/10
         │ - chunk 2/10
         │ - ... (streaming)
         │
         ▼
    React PWA
         │
         │ 1. Buffer first 2 chunks
         │ 2. Start playback (Web Audio API)
         │ 3. Continue streaming remaining chunks
         │ 4. Display playback controls
         │
         ▼
    Audio Output (Speaker)
         │
         │ User hears natural, expressive voice
         │ describing the scene
```

## Connection Management

### WebSocket Lifecycle

```
React PWA Startup
       │
       │ 1. Initialize AppSync client
       │ 2. Establish WSS connection
       │
       ▼
AWS AppSync (Connected)
       │
       │ Subscribe to:
       │ - onVisionResult
       │ - onAudioStream
       │
       ▼
Active Session
       │
       ├─── Normal Operation ───┐
       │                        │
       │                        ▼
       │              Send mutations
       │              Receive subscriptions
       │              < 500ms latency
       │
       ├─── Connection Drop ────┐
       │                        │
       │                        ▼
       │              Automatic Reconnection
       │              - Attempt 1 (1s delay)
       │              - Attempt 2 (2s delay)
       │              - Attempt 3 (4s delay)
       │                        │
       │                        ├─── Success ──> Resume
       │                        │
       │                        └─── Failure ──> HTTP Polling Fallback
       │
       └─── Video Mode Failure ─┐
                                │
                                ▼
                      Graceful Degradation
                      - Fall back to image-only mode
                      - Notify user
                      - Continue operation
```

## Security & Privacy Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Security Layers                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Transport Security                                          │
│     • HTTPS for all HTTP requests                               │
│     • WSS (WebSocket Secure) for real-time communication        │
│     • TLS 1.2+ encryption                                       │
│                                                                 │
│  2. Authentication & Authorization                              │
│     • AppSync API Key authentication                            │
│     • Lambda IAM roles (least privilege)                        │
│     • No user authentication required (public app)              │
│                                                                 │
│  3. Data Privacy                                                │
│     • No permanent storage of images/video                      │
│     • In-memory processing only                                 │
│     • Temporary data deleted within 60 seconds                  │
│     • No PII collection                                         │
│                                                                 │
│  4. Input Validation                                            │
│     • Client-side: file size, format validation                 │
│     • Server-side: image content validation                     │
│     • Rate limiting to prevent abuse                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Monitoring & Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS CloudWatch                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Logs:                                                          │
│  • Lambda function execution logs                               │
│  • AppSync request/response logs                                │
│  • Application error logs                                       │
│                                                                 │
│  Metrics:                                                       │
│  • WebSocket connection count                                   │
│  • Message throughput (messages/second)                         │
│  • Lambda invocation counts & durations                         │
│  • Error rates by type                                          │
│  • Bedrock API latency                                          │
│  • Polly API latency                                            │
│                                                                 │
│  Alarms:                                                        │
│  • High error rate (> 5%)                                       │
│  • High latency (> 5s vision, > 3s speech)                      │
│  • Lambda throttling                                            │
│  • WebSocket connection failures                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Scalability & Performance

```
Component              Auto-Scaling Strategy           Target Performance
─────────────────────────────────────────────────────────────────────────
AWS Amplify            CDN edge locations              < 2s page load
AWS AppSync            Automatic (serverless)          < 500ms latency
Lambda (Vision)        Concurrent executions           < 3s per image
Lambda (Speech)        Concurrent executions           < 2s per synthesis
Amazon Bedrock         Managed service                 < 2s inference
Amazon Polly           Managed service                 < 1s synthesis
```

## Cost Optimization

```
Service              Pricing Model                    Optimization Strategy
─────────────────────────────────────────────────────────────────────────
AWS Amplify          Build minutes + hosting          Static assets, CDN caching
AWS AppSync          Requests + data transfer         WebSocket reuse, batching
Lambda               Invocations + duration           Right-sized memory, timeout
Amazon Bedrock       Tokens processed                 Optimized prompts, caching
Amazon Polly         Characters synthesized           Text optimization, reuse
```

## Deployment Pipeline

```
Developer → Git Push → GitHub
                         │
                         │ Webhook trigger
                         │
                         ▼
                  AWS Amplify CI/CD
                         │
                         ├─── Build Phase
                         │    • npm ci
                         │    • npm run build
                         │    • Run tests
                         │
                         ├─── Deploy Phase
                         │    • Deploy to CDN
                         │    • Update environment variables
                         │    • Invalidate cache
                         │
                         └─── Post-Deploy
                              • Health checks
                              • Smoke tests
                              • Notify team
```

---

## Architecture Highlights

1. **Serverless & Scalable**: All compute resources auto-scale based on demand
2. **Real-Time Streaming**: WebSocket-based architecture for < 500ms latency
3. **Child-Centered Design**: Optimized for children with Aphasia
4. **Privacy-First**: No permanent storage, in-memory processing only
5. **Fault Tolerant**: Automatic reconnection, graceful degradation
6. **Cost-Effective**: Pay-per-use pricing, no idle resources
7. **Secure**: End-to-end encryption, IAM-based access control
8. **Observable**: Comprehensive logging, metrics, and alarms

---

**Generated for**: Ocean Wave - AI for Bharat Submission
**Architecture**: AWS Serverless (Amplify + AppSync + Lambda + Bedrock + Polly)
**Target Users**: Children with Aphasia
**Core Features**: Sighted Guide (Vision) + Eloquent Voice (TTS)
