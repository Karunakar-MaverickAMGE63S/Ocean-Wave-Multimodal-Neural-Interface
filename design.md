# Design Document: Ocean Wave

## Overview

Ocean Wave is a serverless, AWS-native assistive technology application that helps children with Aphasia understand their visual environment through real-time AI-powered vision analysis and expressive speech synthesis. The application leverages a modern WebSocket-based streaming architecture for low-latency, interactive experiences.

### Core Modules

1. **Sighted Guide**: Real-time vision analysis using Amazon Bedrock Claude 3.5 Sonnet for multimodal reasoning
2. **Eloquent Voice**: Expressive text-to-speech using Amazon Polly Neural Engine with localized voice support

### Technology Stack

- **Frontend**: React-based progressive web app (PWA)
- **Hosting**: AWS Amplify with CI/CD pipeline
- **Real-Time Communication**: AWS AppSync with WebSocket subscriptions
- **Compute**: AWS Lambda for serverless processing
- **Vision AI**: Amazon Bedrock (Claude 3.5 Sonnet) for high-speed multimodal reasoning
- **Speech AI**: Amazon Polly Neural Engine for expressive, natural-sounding speech
- **Streaming**: WebSocket-based bidirectional streaming for audio and video

### Design Principles

- **Real-Time First**: Sub-500ms latency for interactive experiences
- **Simplicity First**: Minimize cognitive load with clear visual cues and minimal text
- **Streaming Architecture**: Continuous processing for video streams and audio output
- **Fault Tolerance**: Graceful degradation with automatic reconnection
- **Child-Centered**: Design for independent use by children with Aphasia
- **Privacy-Conscious**: In-memory processing without permanent storage
- **Serverless Scalability**: Auto-scaling Lambda functions and AppSync connections

## Architecture

### System Architecture

The application follows a serverless, event-driven architecture with WebSocket-based real-time communication:

```
┌──────────────────────────────────────────────────────────────┐
│                  Client (React PWA)                          │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Camera/    │  │  UI Layer    │  │  Audio/Video     │   │
│  │  Video      │  │  (React)     │  │  Player          │   │
│  │  Capture    │  │              │  │                  │   │
│  └──────┬──────┘  └──────┬───────┘  └────────▲─────────┘   │
│         │                │                    │             │
│         │                │                    │             │
└─────────┼────────────────┼────────────────────┼─────────────┘
          │                │                    │
          │    WebSocket   │                    │
          │    (WSS)       │                    │
          ▼                ▼                    │
┌──────────────────────────────────────────────┼─────────────┐
│              AWS AppSync                      │             │
│         (GraphQL + WebSocket)                 │             │
│  ┌────────────────────────────────────────┐  │             │
│  │  Subscriptions & Mutations             │  │             │
│  │  - onVisionResult                      │  │             │
│  │  - onAudioStream                       │  │             │
│  │  - processImage / processVideoFrame    │  │             │
│  └────────────┬───────────────────────────┘  │             │
└───────────────┼──────────────────────────────┼─────────────┘
                │                              │
                │ Invoke                       │ Stream
                ▼                              │
┌───────────────────────────┐    ┌────────────┴──────────────┐
│   Lambda: VisionProcessor │    │ Lambda: SpeechSynthesizer │
│   - Receive image/frame   │    │ - Receive description     │
│   - Call Bedrock          │    │ - Call Polly Neural       │
│   - Return description    │    │ - Stream audio chunks     │
└──────────┬────────────────┘    └────────────┬──────────────┘
           │                                  │
           ▼                                  ▼
┌───────────────────────┐         ┌──────────────────────────┐
│   Amazon Bedrock      │         │   Amazon Polly           │
│   Claude 3.5 Sonnet   │         │   Neural Engine          │
│   - Multimodal AI     │         │   - Expressive TTS       │
│   - Vision reasoning  │         │   - Localized voices     │
└───────────────────────┘         └──────────────────────────┘
```

### Data Flow

#### Image Analysis Flow
1. User captures image → Client validates → Sends via AppSync mutation
2. AppSync triggers VisionProcessor Lambda
3. Lambda invokes Bedrock Claude 3.5 Sonnet with image
4. Bedrock returns description → Lambda publishes to AppSync
5. AppSync pushes description to client via subscription
6. Client triggers speech synthesis

#### Video Streaming Flow
1. User activates video mode → Client establishes WebSocket connection
2. Client captures video frames → Sends frames via AppSync at configured rate
3. AppSync triggers VisionProcessor Lambda for each frame
4. Lambda processes frame with Bedrock → Returns description
5. AppSync streams descriptions back to client in real-time
6. Client displays descriptions and triggers speech synthesis

#### Speech Synthesis Flow
1. Client receives description → Sends to SpeechSynthesizer via AppSync
2. Lambda invokes Polly Neural Engine
3. Polly generates audio → Lambda streams audio chunks
4. AppSync pushes audio chunks to client via subscription
5. Client plays audio with streaming playback

### WebSocket Connection Management

```typescript
// AppSync WebSocket lifecycle
1. Connect: Client establishes WSS connection with AppSync
2. Subscribe: Client subscribes to onVisionResult and onAudioStream
3. Mutate: Client sends processImage or processVideoFrame mutations
4. Receive: Client receives real-time updates via subscriptions
5. Reconnect: Automatic reconnection on connection drop (up to 3 attempts)
6. Fallback: HTTP polling if WebSocket fails
```

## Components and Interfaces

### Frontend Components

#### 1. MediaCaptureComponent
**Responsibility**: Handle image and video input from camera or file upload

**Interface**:
```typescript
interface MediaCaptureComponent {
  captureImage(): Promise<ImageData>
  startVideoStream(): Promise<VideoStream>
  stopVideoStream(): void
  uploadFromFile(file: File): Promise<ImageData>
  validateMedia(media: ImageData | VideoFrame): ValidationResult
  previewMedia(media: ImageData | VideoStream): void
}

interface ImageData {
  data: Blob
  format: 'jpeg' | 'png' | 'webp'
  size: number
  timestamp: Date
}

interface VideoStream {
  stream: MediaStream
  frameRate: number
  resolution: { width: number; height: number }
}

interface VideoFrame {
  data: Blob
  timestamp: Date
  frameNumber: number
}

interface ValidationResult {
  isValid: boolean
  error?: string
}
```

#### 2. AppSyncClient
**Responsibility**: Manage WebSocket connection and real-time communication with AWS AppSync

**GraphQL Schema**:
```graphql
type Mutation {
  processImage(
    image: String!  # Base64 encoded
    detailLevel: DetailLevel!
    childFriendly: Boolean!
  ): String!  # Returns requestId
  
  processVideoFrame(
    frame: String!  # Base64 encoded
    frameNumber: Int!
    detailLevel: DetailLevel!
  ): String!  # Returns requestId
  
  synthesizeSpeech(
    text: String!
    voiceId: String!
    engine: Engine!
    rate: SpeechRate!
  ): String!  # Returns requestId
}

type Subscription {
  onVisionResult(requestId: String!): VisionResult
    @aws_subscribe(mutations: ["processImage", "processVideoFrame"])
  
  onAudioStream(requestId: String!): AudioChunk
    @aws_subscribe(mutations: ["synthesizeSpeech"])
}

type VisionResult {
  requestId: String!
  description: String!
  confidence: Float!
  processingTime: Float!
  timestamp: AWSDateTime!
}

type AudioChunk {
  requestId: String!
  data: String!  # Base64 encoded audio
  chunkIndex: Int!
  totalChunks: Int!
  format: AudioFormat!
}

enum DetailLevel { SIMPLE MODERATE DETAILED }
enum Engine { NEURAL STANDARD }
enum SpeechRate { SLOW NORMAL FAST }
enum AudioFormat { MP3 PCM }
```

### Backend Components

#### 3. VisionProcessorLambda
**Responsibility**: Process images and video frames using Amazon Bedrock

**Configuration**:
- Runtime: Python 3.11
- Memory: 1024 MB
- Timeout: 30 seconds
- Model: `anthropic.claude-3-5-sonnet-20241022-v2:0`

**Handler**:
```python
import boto3
import json
import base64
from datetime import datetime

bedrock = boto3.client('bedrock-runtime')

def lambda_handler(event, context):
    image_data = base64.b64decode(event['image'])
    detail_level = event.get('detailLevel', 'simple')
    
    prompt = build_child_friendly_prompt(detail_level)
    
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 300,
            'messages': [{
                'role': 'user',
                'content': [
                    {'type': 'image', 'source': {
                        'type': 'base64',
                        'media_type': 'image/jpeg',
                        'data': base64.b64encode(image_data).decode()
                    }},
                    {'type': 'text', 'text': prompt}
                ]
            }]
        })
    )
    
    result = json.loads(response['body'].read())
    return {
        'requestId': event['requestId'],
        'description': result['content'][0]['text'],
        'confidence': 0.95,
        'processingTime': (datetime.now() - start_time).total_seconds()
    }
```

#### 4. SpeechSynthesizerLambda
**Responsibility**: Convert text to speech using Amazon Polly Neural Engine

**Configuration**:
- Runtime: Python 3.11
- Memory: 512 MB
- Timeout: 15 seconds
- Engine: Neural
- Voices: Joanna, Matthew, Ivy (English US), Aditi, Raveena (English IN)

**Handler**:
```python
import boto3
import base64

polly = boto3.client('polly')

def lambda_handler(event, context):
    text = event['text']
    voice_id = event.get('voiceId', 'Joanna')
    rate = event.get('rate', 'normal')
    
    ssml = f'<speak><prosody rate="{rate}">{text}</prosody></speak>'
    
    response = polly.synthesize_speech(
        Text=ssml,
        TextType='ssml',
        OutputFormat='mp3',
        VoiceId=voice_id,
        Engine='neural'
    )
    
    # Stream in 4KB chunks
    chunks = []
    while chunk := response['AudioStream'].read(4096):
        chunks.append(base64.b64encode(chunk).decode())
    
    return {
        'requestId': event['requestId'],
        'chunks': [
            {'data': chunk, 'chunkIndex': i, 'totalChunks': len(chunks)}
            for i, chunk in enumerate(chunks)
        ]
    }
```

## Data Models

### Client-Side Models

```typescript
interface ImageModel {
  id: string
  data: Blob
  format: 'jpeg' | 'png' | 'webp'
  size: number
  dimensions: { width: number; height: number }
  captureMethod: 'camera' | 'upload'
  timestamp: Date
}

interface VideoStreamModel {
  id: string
  stream: MediaStream
  frameRate: number
  resolution: { width: number; height: number }
  startTime: Date
  framesProcessed: number
  isActive: boolean
}

interface DescriptionModel {
  id: string
  requestId: string
  mediaId: string
  mediaType: 'image' | 'video-frame'
  text: string
  detailLevel: 'simple' | 'moderate' | 'detailed'
  confidence: number
  generatedAt: Date
  processingTime: number
  frameNumber?: number
}

interface AudioModel {
  id: string
  requestId: string
  descriptionId: string
  chunks: AudioChunk[]
  totalDuration: number
  format: 'mp3' | 'pcm'
  voiceId: string
  engine: 'neural' | 'standard'
  speechRate: 'slow' | 'normal' | 'fast'
  generatedAt: Date
  isComplete: boolean
}

interface SessionState {
  connectionState: 'connecting' | 'connected' | 'disconnected' | 'reconnecting'
  streamingMode: 'image' | 'video'
  currentMedia?: ImageModel | VideoStreamModel
  currentDescription?: DescriptionModel
  currentAudio?: AudioModel
  processingStatus: ProcessingStatus
  connectionQuality: ConnectionQuality
  errorState?: ErrorInfo
  settings: UserSettings
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Image Format Validation
*For any* uploaded file, the system should accept the file if and only if it has a valid image format (JPEG, PNG, or WebP) and reject all other formats with an appropriate error message.
**Validates: Requirements 1.3**

### Property 2: Image Size Validation
*For any* image input, the system should accept images under 5MB and reject images at or above 5MB with a clear error message prompting for a new image.
**Validates: Requirements 1.4, 1.5**

### Property 3: Media Preview Display
*For any* successfully validated image or video stream, loading the media should result in a preview being displayed to the user.
**Validates: Requirements 1.6**

### Property 4: WebSocket Connection Establishment
*For any* application start, the system should establish a WebSocket connection to AppSync and transition to connected state within 5 seconds.
**Validates: Requirements 3.1**

### Property 5: Vision API Invocation via AppSync
*For any* valid image or video frame input, the system should send the data through the AppSync WebSocket connection with the configured detail level.
**Validates: Requirements 2.1, 2.2, 3.2**

### Property 6: Real-Time Description Streaming
*For any* vision processing request, the system should receive the description via AppSync subscription within 3 seconds for images and continuously for video frames.
**Validates: Requirements 2.6, 3.3**

### Property 7: Description Content Completeness
*For any* generated scene description, the text should contain identifiable references to visual elements (objects, colors, or spatial relationships).
**Validates: Requirements 2.4**

### Property 8: Child-Friendly Language Complexity
*For any* generated scene description, the average word length should be ≤ 6 characters and average sentence length should be ≤ 12 words to ensure child-appropriate language.
**Validates: Requirements 2.5**

### Property 9: Speech Synthesis Triggering
*For any* successfully generated scene description, the system should automatically invoke the Speech Synthesizer via AppSync to convert the text to audio.
**Validates: Requirements 4.1, 4.4**

### Property 10: Audio Streaming Playback
*For any* speech synthesis request, the system should receive audio chunks via AppSync subscription and begin playback after buffering the first 2 chunks.
**Validates: Requirements 4.4**

### Property 11: Speech Synthesis Fallback
*For any* speech synthesis failure, the system should display the text description as a fallback to ensure the user can still access the content.
**Validates: Requirements 4.7**

### Property 12: WebSocket Reconnection
*For any* WebSocket connection drop, the system should attempt automatic reconnection up to 3 times with exponential backoff before falling back to HTTP polling.
**Validates: Requirements 3.4, 11.6**

### Property 13: Video Streaming Latency
*For any* active video stream, the latency between frame capture and description delivery should be less than 500ms for 95% of frames.
**Validates: Requirements 3.5**

### Property 14: Button Accessibility Standards
*For any* primary action button in the interface, the button should have a minimum touch target size of 44x44 pixels, include both an icon and label, and provide visual feedback on interaction.
**Validates: Requirements 5.1, 5.3, 5.6**

### Property 15: Color Contrast Compliance
*For any* text and background color combination in the interface, the contrast ratio should meet or exceed WCAG AA standards (4.5:1 for normal text, 3:1 for large text).
**Validates: Requirements 5.2**

### Property 16: Error Communication Format
*For any* error that occurs, the error display should include both a visual icon and simple text explanation, and for recoverable errors, should include a retry action.
**Validates: Requirements 5.5, 11.1, 11.3**

### Property 17: Processing State Feedback
*For any* asynchronous operation (image analysis, video streaming, or speech synthesis), the system should display a loading indicator while processing and a success indicator upon completion.
**Validates: Requirements 6.1, 6.2, 6.3**

### Property 18: Duplicate Submission Prevention
*For any* operation in progress, attempting to submit another request should be prevented (buttons disabled) until the current operation completes.
**Validates: Requirements 6.4**

### Property 19: Connection Quality Indicators
*For any* active video streaming session, the system should display real-time connection quality metrics (latency, frame rate) updated at least once per second.
**Validates: Requirements 5.7, 6.5**

### Property 20: Settings Persistence Round-Trip
*For any* valid settings configuration, saving the settings and then reloading the application should restore the exact same settings values including streaming mode and frame rate.
**Validates: Requirements 7.8, 8.5**

### Property 21: Responsive Layout Adaptation
*For any* viewport width, the application layout should adapt appropriately with mobile-optimized layouts for widths < 768px and desktop layouts for widths ≥ 768px.
**Validates: Requirements 8.4**

### Property 22: API Retry Logic
*For any* AppSync mutation that fails with a retryable error, the system should retry the request with exponential backoff up to 3 attempts before displaying an error.
**Validates: Requirements 9.4**

### Property 23: API Error Handling
*For any* AWS service call failure, the system should log the error details (timestamp, service, error code) and display a user-friendly error message without exposing technical details.
**Validates: Requirements 9.5**

### Property 24: Data Privacy - No Permanent Storage
*For any* processed image or video frame, the data should not be written to any persistent storage (database, file system) and temporary data should be deleted within 60 seconds of processing completion.
**Validates: Requirements 10.2, 10.3, 10.6**

### Property 25: PII Collection Prevention
*For any* data model or API request, the system should not include fields for personally identifiable information (name, email, phone, address) unless explicitly required and consented.
**Validates: Requirements 10.4**

### Property 26: Secure Transmission
*For any* data transmission, the system should use secure protocols (HTTPS for HTTP requests, WSS for WebSocket connections).
**Validates: Requirements 10.1**

### Property 27: Network Error Recovery
*For any* network connectivity failure, the system should display a clear connectivity error message and provide a retry option.
**Validates: Requirements 11.2**

### Property 28: Escalated Error Messaging
*For any* sequence of 3 or more consecutive failed operations, the system should display an enhanced error message suggesting to check internet connection or contact support.
**Validates: Requirements 11.4**

### Property 29: Error Logging Completeness
*For any* unexpected error, the system should log the error with timestamp, error type, error message, stack trace, and user action context.
**Validates: Requirements 11.5**

### Property 30: Video Streaming Fallback
*For any* video streaming session where WebSocket reconnection fails after 3 attempts, the system should gracefully fall back to image-only mode and notify the user.
**Validates: Requirements 11.7**

## Error Handling

### Error Categories

1. **User Input Errors**: Invalid image format, oversized files, corrupted images
2. **Service Errors**: AWS API failures, rate limiting, timeouts
3. **Network Errors**: Connectivity loss, WebSocket disconnections
4. **Application Errors**: Unexpected exceptions, state inconsistencies

### Error Handling Strategy

#### WebSocket Connection Errors
```typescript
const RECONNECTION_CONFIG = {
  maxAttempts: 3,
  baseDelay: 1000,
  maxDelay: 10000,
  backoffMultiplier: 2
}

async function reconnectWithBackoff(): Promise<void> {
  for (let attempt = 1; attempt <= RECONNECTION_CONFIG.maxAttempts; attempt++) {
    try {
      await connect()
      return
    } catch (error) {
      if (attempt === RECONNECTION_CONFIG.maxAttempts) {
        enableHttpPollingMode()
        throw error
      }
      const delay = Math.min(
        RECONNECTION_CONFIG.baseDelay * Math.pow(RECONNECTION_CONFIG.backoffMultiplier, attempt - 1),
        RECONNECTION_CONFIG.maxDelay
      )
      await sleep(delay)
    }
  }
}
```

#### Service Errors
- Automatic retry with exponential backoff for rate limits (429) and temporary failures (503)
- Fallback to text display for speech synthesis failures
- Graceful degradation from video to image mode on persistent failures

#### User Feedback
- Clear error messages with visual icons
- Retry buttons for recoverable errors
- Escalated messaging after 3 consecutive failures
- Connection quality indicators during streaming

## Testing Strategy

### Dual Testing Approach

Ocean Wave requires both unit testing and property-based testing:

- **Unit Tests**: Verify specific examples, edge cases, and integration points
- **Property Tests**: Verify universal properties across randomized inputs

### Property-Based Testing

**Library**: fast-check (JavaScript/TypeScript)
**Configuration**: Minimum 100 iterations per property test

**Example Property Tests**:
```typescript
import fc from 'fast-check'

// Feature: ocean-wave, Property 2: Image Size Validation
describe('Property 2: Image Size Validation', () => {
  it('should accept images under 5MB and reject images >= 5MB', () => {
    fc.assert(
      fc.property(
        fc.integer({ min: 0, max: 10 * 1024 * 1024 }),
        (size) => {
          const image = createMockImage(size)
          const result = validateImage(image)
          
          if (size < 5 * 1024 * 1024) {
            expect(result.isValid).toBe(true)
          } else {
            expect(result.isValid).toBe(false)
            expect(result.error).toBeDefined()
          }
        }
      ),
      { numRuns: 100 }
    )
  })
})

// Feature: ocean-wave, Property 12: WebSocket Reconnection
describe('Property 12: WebSocket Reconnection', () => {
  it('should attempt reconnection up to 3 times before fallback', () => {
    fc.assert(
      fc.property(
        fc.integer({ min: 1, max: 5 }),
        async (failureCount) => {
          const mockConnection = createMockConnection(failureCount)
          const manager = new ConnectionManager(mockConnection)
          
          await manager.connect()
          
          if (failureCount <= 3) {
            expect(manager.getState()).toBe('connected')
            expect(mockConnection.attempts).toBe(failureCount)
          } else {
            expect(manager.getState()).toBe('fallback')
            expect(mockConnection.attempts).toBe(3)
          }
        }
      ),
      { numRuns: 100 }
    )
  })
})

// Feature: ocean-wave, Property 20: Settings Persistence Round-Trip
describe('Property 20: Settings Persistence Round-Trip', () => {
  it('should restore settings after save and reload', () => {
    fc.assert(
      fc.property(
        fc.record({
          voiceId: fc.constantFrom('Joanna', 'Matthew', 'Ivy'),
          speechRate: fc.constantFrom('slow', 'normal', 'fast'),
          volume: fc.integer({ min: 0, max: 100 }),
          detailLevel: fc.constantFrom('simple', 'moderate', 'detailed'),
          streamingMode: fc.constantFrom('image', 'video'),
          videoFrameRate: fc.integer({ min: 1, max: 5 })
        }),
        (settings) => {
          settingsManager.updateSettings(settings)
          const restored = settingsManager.getSettings()
          expect(restored).toEqual(settings)
        }
      ),
      { numRuns: 100 }
    )
  })
})
```

### Unit Testing

**Framework**: Jest with React Testing Library
**Coverage Target**: 80% minimum

**Focus Areas**:
- WebSocket connection lifecycle
- AppSync mutation and subscription handling
- Video frame processing and throttling
- Audio chunk buffering and playback
- Error recovery and fallback mechanisms

### Integration Testing

**Tools**: Playwright for browser automation

**Test Scenarios**:
1. End-to-end: Capture → Analyze → Speak
2. Video streaming: Start stream → Process frames → Display descriptions
3. Error recovery: Connection drop → Reconnect → Resume
4. Fallback: WebSocket failure → HTTP polling mode

## Deployment Architecture

### AWS Amplify Hosting

**Configuration**:
```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - '**/*'
```

**Environment Variables**:
- `VITE_APPSYNC_URL`: AppSync GraphQL endpoint
- `VITE_APPSYNC_API_KEY`: AppSync API key
- `VITE_AWS_REGION`: AWS region

### AWS AppSync

**API Type**: GraphQL with WebSocket subscriptions
**Authentication**: API Key (for public access)
**Data Sources**: Lambda functions

**Resolvers**:
- `Mutation.processImage` → VisionProcessorLambda
- `Mutation.processVideoFrame` → VisionProcessorLambda
- `Mutation.synthesizeSpeech` → SpeechSynthesizerLambda

### Lambda Functions

**VisionProcessorLambda**:
- Runtime: Python 3.11
- Memory: 1024 MB
- Timeout: 30 seconds
- IAM Role: Bedrock invoke permissions

**SpeechSynthesizerLambda**:
- Runtime: Python 3.11
- Memory: 512 MB
- Timeout: 15 seconds
- IAM Role: Polly synthesize permissions

### CloudWatch

**Logging**:
- Lambda function logs
- AppSync request logs
- Application error logs

**Metrics**:
- WebSocket connection count
- Message throughput
- Lambda invocation counts and durations
- Error rates by type

**Alarms**:
- High error rate (> 5%)
- High latency (> 5s for vision, > 3s for speech)
- Lambda throttling

## Security Considerations

### Authentication and Authorization
- AppSync uses API keys for public access
- Lambda functions use IAM roles with least-privilege permissions
- No user authentication required (public application)

### Data Security
- All transmission over HTTPS/WSS (TLS 1.2+)
- No permanent storage of images or video
- Temporary data deleted within 60 seconds
- No PII collection without consent

### Input Validation
- Client-side validation for file size and format
- Server-side validation for image content
- Rate limiting to prevent abuse

## Future Enhancements

1. **Multi-language Support**: Regional Indian languages
2. **Offline Mode**: Cache descriptions for offline playback
3. **Custom Vocabulary**: Caregiver-defined words/phrases
4. **Progress Tracking**: Usage analytics (with consent)
5. **Voice Customization**: Fine-tune voice characteristics
6. **Batch Processing**: Multiple images in sequence
