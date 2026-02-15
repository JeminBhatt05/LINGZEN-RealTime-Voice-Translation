    # Design Document: LINGZEN Real-Time Voice Translation Platform

    ## Overview

    LINGZEN is a real-time voice translation platform built on a microservices architecture that enables seamless multilingual communication during video meetings. The system processes audio through a multi-stage pipeline: audio capture → speech-to-text → translation → text-to-speech → audio delivery, with a target end-to-end latency of less than 3 seconds.

    The platform uses WebSocket connections for real-time bidirectional communication, external AI services (Google Speech-to-Text, DeepL, Google TTS, ElevenLabs) for processing, and a multi-database architecture (PostgreSQL for relational data, Redis for caching and queuing, MongoDB for document storage, S3 for audio files) to optimize for different data access patterns.

    Key design principles:
    - **Low Latency**: Asynchronous processing, caching, and parallel execution to minimize translation delay
    - **Scalability**: Horizontal scaling with Kubernetes, stateless services, and distributed queuing
    - **Reliability**: Retry logic, fallback services, graceful degradation, and comprehensive error handling
    - **Security**: End-to-end encryption, JWT authentication, input validation, and GDPR compliance
    - **Modularity**: Loosely coupled services with clear interfaces for maintainability and extensibility

    ## Architecture

    ### System Architecture Diagram

    ```mermaid
    graph TB
        subgraph "Client Layer"
            WebApp[React Web Application]
            AudioCapture[Audio Capture Module]
            WSClient[WebSocket Client]
        end
        
        subgraph "API Gateway Layer"
            Nginx[Nginx Load Balancer]
            APIGateway[API Gateway]
        end
        
        subgraph "Application Layer"
            AuthService[Authentication Service]
            MeetingService[Meeting Service]
            TranslationService[Translation Orchestrator]
            UserService[User Management Service]
            IntegrationService[Integration Service]
        end
        
        subgraph "Processing Layer"
            STTWorker[Speech-to-Text Worker]
            TranslationWorker[Translation Worker]
            TTSWorker[Text-to-Speech Worker]
            AudioProcessor[Audio Processing Worker]
        end
        
        subgraph "External Services"
            GoogleSTT[Google Speech-to-Text]
            DeepL[DeepL Translation API]
            GoogleTranslate[Google Translate API]
            GoogleTTS[Google TTS]
            ElevenLabs[ElevenLabs TTS]
        end
        
        subgraph "Data Layer"
            PostgreSQL[(PostgreSQL)]
            Redis[(Redis)]
            MongoDB[(MongoDB)]
            S3[(AWS S3)]
        end
        
        subgraph "Infrastructure"
            Celery[Celery Task Queue]
            Prometheus[Prometheus Monitoring]
            K8s[Kubernetes Cluster]
        end
        
        WebApp --> Nginx
        AudioCapture --> WSClient
        WSClient --> Nginx
        Nginx --> APIGateway
        APIGateway --> AuthService
        APIGateway --> MeetingService
        APIGateway --> UserService
        APIGateway --> IntegrationService
        
        MeetingService --> TranslationService
        TranslationService --> Celery
        Celery --> STTWorker
        Celery --> TranslationWorker
        Celery --> TTSWorker
        Celery --> AudioProcessor
        
        STTWorker --> GoogleSTT
        TranslationWorker --> DeepL
        TranslationWorker --> GoogleTranslate
        TTSWorker --> GoogleTTS
        TTSWorker --> ElevenLabs
        
        AuthService --> PostgreSQL
        MeetingService --> PostgreSQL
        UserService --> PostgreSQL
        IntegrationService --> PostgreSQL
        
        TranslationService --> Redis
        MeetingService --> Redis
        AudioProcessor --> S3
        
        TranslationService --> MongoDB
        MeetingService --> MongoDB
        
        K8s -.-> Prometheus
    ```

    ### Translation Pipeline Architecture

    ```mermaid
    sequenceDiagram
        participant Client
        participant WebSocket
        participant AudioQueue
        participant STTWorker
        participant TranslationWorker
        participant TTSWorker
        participant Cache
        participant S3
        participant Clients
        
        Client->>WebSocket: audio_stream (base64, 500ms)
        WebSocket->>S3: Upload audio chunk
        WebSocket->>AudioQueue: Queue audio job
        WebSocket-->>Client: ACK
        
        AudioQueue->>STTWorker: Dequeue audio job
        STTWorker->>S3: Fetch audio
        STTWorker->>GoogleSTT: Transcribe (800ms)
        GoogleSTT-->>STTWorker: Transcript text
        STTWorker->>WebSocket: Broadcast transcript
        WebSocket-->>Clients: transcript event
        
        STTWorker->>TranslationWorker: Queue translation jobs
        TranslationWorker->>Cache: Check translation cache
        alt Cache hit
            Cache-->>TranslationWorker: Cached translation
        else Cache miss
            TranslationWorker->>DeepL: Translate (600ms)
            DeepL-->>TranslationWorker: Translated text
            TranslationWorker->>Cache: Store translation
        end
        
        TranslationWorker->>WebSocket: Broadcast translation
        WebSocket-->>Clients: translation event
        
        TranslationWorker->>TTSWorker: Queue TTS jobs
        TTSWorker->>Cache: Check audio cache
        alt Cache hit
            Cache-->>TTSWorker: Cached audio
        else Cache miss
            TTSWorker->>GoogleTTS: Synthesize (400ms)
            GoogleTTS-->>TTSWorker: Audio data
            TTSWorker->>S3: Store audio
            TTSWorker->>Cache: Store audio reference
        end
        
        TTSWorker->>WebSocket: Send translated audio (300ms)
        WebSocket-->>Clients: translated_audio event
    ```

    ### Deployment Architecture

    ```mermaid
    graph TB
        subgraph "Kubernetes Cluster"
            subgraph "Frontend Pods"
                FE1[Frontend Pod 1]
                FE2[Frontend Pod 2]
                FE3[Frontend Pod N]
            end
            
            subgraph "Backend Pods"
                BE1[Backend Pod 1]
                BE2[Backend Pod 2]
                BE3[Backend Pod N]
            end
            
            subgraph "Worker Pods"
                W1[Worker Pod 1]
                W2[Worker Pod 2]
                W3[Worker Pod N]
            end
            
            Ingress[Ingress Controller]
            HPA[Horizontal Pod Autoscaler]
        end
        
        subgraph "Managed Services"
            RDS[AWS RDS PostgreSQL]
            ElastiCache[AWS ElastiCache Redis]
            DocumentDB[MongoDB Atlas]
            S3Bucket[AWS S3]
        end
        
        subgraph "Monitoring"
            Prometheus[Prometheus]
            Grafana[Grafana]
            CloudWatch[CloudWatch Logs]
        end
        
        Internet[Internet] --> Ingress
        Ingress --> FE1
        Ingress --> FE2
        Ingress --> FE3
        Ingress --> BE1
        Ingress --> BE2
        Ingress --> BE3
        
        HPA -.-> FE1
        HPA -.-> BE1
        HPA -.-> W1
        
        BE1 --> RDS
        BE2 --> RDS
        BE3 --> RDS
        
        BE1 --> ElastiCache
        BE2 --> ElastiCache
        BE3 --> ElastiCache
        
        W1 --> DocumentDB
        W2 --> DocumentDB
        W3 --> DocumentDB
        
        W1 --> S3Bucket
        W2 --> S3Bucket
        W3 --> S3Bucket
        
        BE1 -.-> Prometheus
        W1 -.-> Prometheus
        Prometheus --> Grafana
        BE1 -.-> CloudWatch
    ```

    ### Use Case Diagram

    ```mermaid
    graph TB
        subgraph "Actors"
            User[User]
            Host[Meeting Host]
            Participant[Meeting Participant]
            Admin[System Administrator]
        end
        
        subgraph "Authentication Use Cases"
            UC1[Register Account]
            UC2[Login]
            UC3[Reset Password]
            UC4[Manage Profile]
            UC5[Update Preferences]
        end
        
        subgraph "Meeting Use Cases"
            UC6[Create Meeting]
            UC7[Join Meeting]
            UC8[Leave Meeting]
            UC9[End Meeting]
            UC10[View Meeting History]
            UC11[Export Transcript]
        end
        
        subgraph "Translation Use Cases"
            UC12[Stream Audio]
            UC13[Receive Translations]
            UC14[Change Target Language]
            UC15[View Live Transcript]
            UC16[Toggle Audio On/Off]
        end
        
        subgraph "Subscription Use Cases"
            UC17[Upgrade Subscription]
            UC18[View Usage Statistics]
            UC19[Manage Billing]
        end
        
        subgraph "Integration Use Cases"
            UC20[Connect Zoom]
            UC21[Connect Teams]
            UC22[Connect Google Meet]
            UC23[Disconnect Integration]
        end
        
        subgraph "Admin Use Cases"
            UC24[Monitor System Health]
            UC25[View Error Logs]
            UC26[Manage Quotas]
            UC27[Configure Services]
        end
        
        User --> UC1
        User --> UC2
        User --> UC3
        User --> UC4
        User --> UC5
        User --> UC10
        User --> UC17
        User --> UC18
        User --> UC19
        User --> UC20
        User --> UC21
        User --> UC22
        User --> UC23
        
        Host --> UC6
        Host --> UC9
        Host --> UC11
        
        Participant --> UC7
        Participant --> UC8
        Participant --> UC12
        Participant --> UC13
        Participant --> UC14
        Participant --> UC15
        Participant --> UC16
        
        Admin --> UC24
        Admin --> UC25
        Admin --> UC26
        Admin --> UC27
        
        UC6 -.->|extends| UC7
        UC7 -.->|includes| UC12
        UC12 -.->|includes| UC13
        UC13 -.->|includes| UC15
    ```

    ### Complete Meeting Session Process Flow

    ```mermaid
    flowchart TD
        Start([User Opens LINGZEN]) --> CheckAuth{Authenticated?}
        CheckAuth -->|No| Login[Login/Register]
        Login --> Dashboard
        CheckAuth -->|Yes| Dashboard[View Dashboard]
        
        Dashboard --> CreateOrJoin{Create or Join?}
        CreateOrJoin -->|Create| CreateMeeting[Create Meeting]
        CreateOrJoin -->|Join| EnterMeetingID[Enter Meeting ID]
        
        CreateMeeting --> SelectLanguage[Select Target Language]
        EnterMeetingID --> SelectLanguage
        
        SelectLanguage --> JoinMeeting[Join Meeting]
        JoinMeeting --> EstablishWS[Establish WebSocket Connection]
        
        EstablishWS --> CheckMic{Microphone<br/>Permission?}
        CheckMic -->|Denied| ShowError[Show Permission Error]
        ShowError --> RetryPermission{Retry?}
        RetryPermission -->|Yes| CheckMic
        RetryPermission -->|No| TextOnlyMode[Text-Only Mode]
        
        CheckMic -->|Granted| StartAudioCapture[Start Audio Capture]
        StartAudioCapture --> MeetingActive[Meeting Active]
        TextOnlyMode --> MeetingActive
        
        MeetingActive --> UserAction{User Action}
        
        UserAction -->|Speak| CaptureAudio[Capture Audio Chunk]
        CaptureAudio --> SendToWS[Send via WebSocket]
        SendToWS --> QueueSTT[Queue for STT]
        QueueSTT --> ProcessSTT[Process Speech-to-Text]
        ProcessSTT --> BroadcastTranscript[Broadcast Transcript]
        BroadcastTranscript --> QueueTranslation[Queue Translation]
        QueueTranslation --> ProcessTranslation[Translate Text]
        ProcessTranslation --> QueueTTS[Queue TTS]
        QueueTTS --> ProcessTTS[Synthesize Speech]
        ProcessTTS --> SendAudio[Send Translated Audio]
        SendAudio --> PlayAudio[Play Audio to Participants]
        PlayAudio --> MeetingActive
        
        UserAction -->|Change Language| UpdateLanguage[Update Target Language]
        UpdateLanguage --> NotifyParticipants[Notify Other Participants]
        NotifyParticipants --> MeetingActive
        
        UserAction -->|Toggle Audio| ToggleAudio[Mute/Unmute]
        ToggleAudio --> UpdateStatus[Update Audio Status]
        UpdateStatus --> MeetingActive
        
        UserAction -->|View Transcript| ShowTranscript[Display Live Transcript]
        ShowTranscript --> MeetingActive
        
        UserAction -->|Leave| LeaveMeeting[Leave Meeting]
        LeaveMeeting --> CloseWS[Close WebSocket]
        CloseWS --> RemoveParticipant[Remove from Participant List]
        RemoveParticipant --> NotifyOthers[Notify Other Participants]
        NotifyOthers --> CheckHost{Is Host?}
        
        CheckHost -->|No| ReturnDashboard[Return to Dashboard]
        CheckHost -->|Yes| CheckParticipants{Other<br/>Participants?}
        
        CheckParticipants -->|Yes| TransferHost[Transfer Host Role]
        TransferHost --> ReturnDashboard
        CheckParticipants -->|No| EndMeeting[End Meeting]
        
        UserAction -->|End Meeting<br/>(Host Only)| EndMeeting
        EndMeeting --> CloseAllConnections[Close All Connections]
        CloseAllConnections --> SaveTranscript[Save Transcript to MongoDB]
        SaveTranscript --> CleanupResources[Cleanup Resources]
        CleanupResources --> ReturnDashboard
        
        ReturnDashboard --> End([End])
    ```

    ### Data Flow Diagram

    ```mermaid
    flowchart LR
        subgraph "Client Layer"
            Browser[Web Browser]
            Microphone[Microphone]
            Speaker[Speaker]
        end
        
        subgraph "Edge Layer"
            CDN[CDN - Static Assets]
            LB[Load Balancer]
        end
        
        subgraph "Application Layer"
            API[API Gateway]
            WS[WebSocket Server]
            Auth[Auth Service]
            Meeting[Meeting Service]
        end
        
        subgraph "Processing Layer"
            Queue[Redis Queue]
            STT[STT Worker]
            Trans[Translation Worker]
            TTS[TTS Worker]
        end
        
        subgraph "Storage Layer"
            PG[(PostgreSQL<br/>User Data)]
            Redis[(Redis<br/>Cache/Queue)]
            Mongo[(MongoDB<br/>Transcripts)]
            S3[(S3<br/>Audio Files)]
        end
        
        subgraph "External Services"
            GoogleSTT[Google STT API]
            DeepL[DeepL API]
            GoogleTTS[Google TTS API]
        end
        
        Microphone -->|Raw Audio| Browser
        Browser -->|Base64 Audio| CDN
        CDN --> LB
        LB --> WS
        
        Browser -->|HTTP Requests| CDN
        CDN --> LB
        LB --> API
        
        API --> Auth
        API --> Meeting
        Auth --> PG
        Meeting --> PG
        Meeting --> Redis
        
        WS -->|Audio Chunks| Queue
        Queue --> STT
        STT -->|Audio File| S3
        STT -->|Transcribe Request| GoogleSTT
        GoogleSTT -->|Transcript| STT
        STT -->|Transcript| WS
        STT -->|Queue| Trans
        
        Trans -->|Check Cache| Redis
        Trans -->|Translate Request| DeepL
        DeepL -->|Translation| Trans
        Trans -->|Store Cache| Redis
        Trans -->|Translation| WS
        Trans -->|Queue| TTS
        
        TTS -->|Synthesize Request| GoogleTTS
        GoogleTTS -->|Audio| TTS
        TTS -->|Audio File| S3
        TTS -->|Audio Data| WS
        
        WS -->|Transcript| Browser
        WS -->|Translation| Browser
        WS -->|Audio| Browser
        Browser -->|Decoded Audio| Speaker
        
        Meeting -->|Save Transcript| Mongo
        STT -->|Log Translation| Mongo
        Trans -->|Log Translation| Mongo
    ```

    ### Error Handling Flow Diagram

    ```mermaid
    flowchart TD
        Start([API Request/Event]) --> Process[Process Request]
        Process --> Success{Success?}
        
        Success -->|Yes| LogSuccess[Log Success Metrics]
        LogSuccess --> Return[Return Response]
        Return --> End([End])
        
        Success -->|No| IdentifyError[Identify Error Type]
        
        IdentifyError --> ErrorType{Error Type?}
        
        ErrorType -->|External API| CheckRetries{Retries<br/>Remaining?}
        CheckRetries -->|Yes| Backoff[Exponential Backoff]
        Backoff --> Wait[Wait]
        Wait --> Retry[Retry Request]
        Retry --> Process
        
        CheckRetries -->|No| CheckFallback{Fallback<br/>Available?}
        CheckFallback -->|Yes| UseFallback[Use Fallback Service]
        UseFallback --> Process
        
        CheckFallback -->|No| LogError[Log Error to MongoDB]
        LogError --> NotifyUser[Send Error Event to User]
        NotifyUser --> Degrade[Graceful Degradation]
        
        ErrorType -->|Validation| ValidateInput[Check Input]
        ValidateInput --> Return400[Return 400 Bad Request]
        Return400 --> End
        
        ErrorType -->|Authentication| CheckToken{Token<br/>Valid?}
        CheckToken -->|Expired| RefreshToken{Can<br/>Refresh?}
        RefreshToken -->|Yes| IssueNewToken[Issue New Token]
        IssueNewToken --> Process
        RefreshToken -->|No| Return401[Return 401 Unauthorized]
        Return401 --> End
        
        CheckToken -->|Invalid| Return401
        
        ErrorType -->|Authorization| CheckPermissions[Check User Permissions]
        CheckPermissions --> Return403[Return 403 Forbidden]
        Return403 --> End
        
        ErrorType -->|Rate Limit| CheckQuota{Quota<br/>Exceeded?}
        CheckQuota -->|Yes| Return429[Return 429 Too Many Requests]
        Return429 --> End
        
        ErrorType -->|Database| CheckConnection{Connection<br/>Available?}
        CheckConnection -->|No| Reconnect[Reconnect to Database]
        Reconnect --> CheckSuccess{Success?}
        CheckSuccess -->|Yes| Process
        CheckSuccess -->|No| Return503[Return 503 Service Unavailable]
        Return503 --> End
        
        CheckConnection -->|Yes| UseCache{Cache<br/>Available?}
        UseCache -->|Yes| ReturnCached[Return Cached Data]
        ReturnCached --> End
        UseCache -->|No| Return503
        
        ErrorType -->|WebSocket| CheckWS{Connection<br/>Active?}
        CheckWS -->|No| AttemptReconnect[Attempt Reconnection]
        AttemptReconnect --> ReconnectCount{Attempts<br/>< 5?}
        ReconnectCount -->|Yes| BackoffWS[Exponential Backoff]
        BackoffWS --> WaitWS[Wait]
        WaitWS --> AttemptReconnect
        ReconnectCount -->|No| NotifyDisconnect[Notify User: Disconnected]
        NotifyDisconnect --> End
        
        CheckWS -->|Yes| QueueEvent[Queue Missed Events]
        QueueEvent --> End
        
        Degrade --> DegradeLevel{Degradation<br/>Level?}
        DegradeLevel -->|Text Only| DisableTTS[Disable TTS]
        DisableTTS --> ContinueMeeting[Continue Meeting]
        ContinueMeeting --> End
        
        DegradeLevel -->|Original Language| DisableTranslation[Disable Translation]
        DisableTranslation --> ContinueMeeting
        
        DegradeLevel -->|Critical| EndMeeting[End Meeting]
        EndMeeting --> SaveState[Save Current State]
        SaveState --> End
    ```

    ## Components and Interfaces

    ### Frontend Components

    #### 1. Authentication Module
    - **LoginForm**: Handles user login with email/password, displays validation errors, manages JWT token storage
    - **RegisterForm**: Handles user registration with email/password/name validation, password strength indicator
    - **ForgotPassword**: Initiates password reset flow, sends reset email, validates reset tokens
    - **AuthContext**: React context providing authentication state and methods across the application

    #### 2. Meeting Interface Module
    - **MeetingInterface**: Main container component managing meeting state, WebSocket connection, and child components
    - **ParticipantList**: Displays active participants with name, language, speaking status, and connection quality
    - **TranscriptViewer**: Real-time scrolling transcript with original text and translations, speaker identification
    - **ControlBar**: Meeting controls (mute/unmute, leave, record, settings)
    - **LanguageSelector**: Dropdown for selecting target translation language
    - **AudioVisualizer**: Visual representation of audio input levels and speaking activity
    - **LatencyIndicator**: Displays current translation latency and connection quality
    - **ConnectionIndicator**: Shows WebSocket connection status with reconnection feedback

    #### 3. Dashboard Module
    - **Dashboard**: Main dashboard showing meeting history, usage statistics, and quick actions
    - **StatisticsCard**: Displays usage metrics (translation minutes, meetings attended, etc.)
    - **MeetingHistory**: List of past meetings with join date, duration, participants, and transcript access

    #### 4. Settings Module
    - **Settings**: Container for all settings pages
    - **AccountSettings**: User profile management (name, email, password change)
    - **TranslationPreferences**: Default language preferences, voice selection, quality settings
    - **AudioSettings**: Microphone selection, audio quality, noise suppression settings

    ### Backend Services

    #### 1. Authentication Service
    **Responsibilities**: User registration, login, token management, password reset

    **Endpoints**:
    - `POST /auth/register`: Create new user account
    - `POST /auth/login`: Authenticate user and issue tokens
    - `POST /auth/refresh`: Refresh access token using refresh token
    - `POST /auth/logout`: Invalidate user session
    - `POST /auth/verify-email`: Verify email address with token
    - `POST /auth/forgot-password`: Initiate password reset
    - `POST /auth/reset-password`: Complete password reset with token

    **Dependencies**: PostgreSQL (users table), Redis (session storage), Email service

    #### 2. Meeting Service
    **Responsibilities**: Meeting lifecycle management, participant management, WebSocket connection handling

    **Endpoints**:
    - `POST /meetings/create`: Create new meeting
    - `POST /meetings/{id}/join`: Join existing meeting
    - `GET /meetings/{id}`: Get meeting details
    - `GET /meetings/history`: Get user's meeting history
    - `DELETE /meetings/{id}`: End meeting (host only)

    **WebSocket Events**:
    - Client → Server: `join_meeting`, `audio_stream`, `change_language`, `toggle_audio`, `leave_meeting`, `ping`
    - Server → Client: `connected`, `meeting_joined`, `transcript`, `translation`, `translated_audio`, `participant_joined`, `participant_left`, `participant_updated`, `speaking_status`, `latency_update`, `error`, `meeting_ended`, `pong`

    **Dependencies**: PostgreSQL (meetings, meeting_participants), Redis (active sessions), MongoDB (transcripts), WebSocket manager

    #### 3. Translation Orchestrator Service
    **Responsibilities**: Coordinate translation pipeline, manage worker queues, handle caching

    **Key Methods**:
    - `process_audio_stream(audio_data, meeting_id, user_id)`: Queue audio for STT processing
    - `handle_transcript(transcript, meeting_id)`: Distribute transcript and queue translations
    - `handle_translation(translation, target_language, meeting_id)`: Queue TTS and broadcast
    - `handle_translated_audio(audio_data, target_users)`: Deliver audio to participants

    **Dependencies**: Redis (queues, cache), Celery (task distribution), MongoDB (logs)

    #### 4. User Management Service
    **Responsibilities**: User profile management, preferences, subscription management

    **Endpoints**:
    - `GET /users/me`: Get current user profile
    - `PUT /users/me`: Update user profile
    - `DELETE /users/me`: Delete user account
    - `GET /users/preferences`: Get user preferences
    - `PUT /users/preferences`: Update user preferences

    **Dependencies**: PostgreSQL (users, user_preferences, subscriptions)

    #### 5. Integration Service
    **Responsibilities**: OAuth integration with external platforms (Zoom, Teams, Google Meet)

    **Endpoints**:
    - `GET /integrations`: List user's integrations
    - `POST /integrations/zoom/connect`: Initiate Zoom OAuth flow
    - `POST /integrations/teams/connect`: Initiate Teams OAuth flow
    - `POST /integrations/meet/connect`: Initiate Google Meet OAuth flow
    - `DELETE /integrations/{id}`: Disconnect integration

    **Dependencies**: PostgreSQL (oauth_integrations), External OAuth providers

    ### Worker Services

    #### 1. Speech-to-Text Worker
    **Responsibilities**: Process audio chunks, call Google Speech-to-Text API, handle retries

    **Key Methods**:
    - `process_audio(audio_url, meeting_id, user_id)`: Download audio from S3, call STT API, return transcript
    - `retry_with_backoff(func, max_retries=3)`: Retry failed API calls with exponential backoff

    **Configuration**:
    - Audio format: LINEAR16, 16kHz sample rate
    - Language detection: Automatic with confidence threshold
    - Streaming: Enabled for real-time processing

    **Dependencies**: Google Cloud Speech-to-Text API, S3, Redis (result queue)

    #### 2. Translation Worker
    **Responsibilities**: Translate text between languages, manage cache, handle fallback

    **Key Methods**:
    - `translate_text(text, source_lang, target_lang)`: Translate using DeepL with Google fallback
    - `check_cache(text, source_lang, target_lang)`: Check Redis for cached translation
    - `store_cache(text, translation, source_lang, target_lang)`: Store translation in Redis (24h TTL)

    **Configuration**:
    - Primary: DeepL API (higher quality)
    - Fallback: Google Translate API
    - Cache TTL: 24 hours
    - Supported languages: 50+ languages

    **Dependencies**: DeepL API, Google Translate API, Redis (translation cache)

    #### 3. Text-to-Speech Worker
    **Responsibilities**: Synthesize speech from translated text, manage voice profiles, cache audio

    **Key Methods**:
    - `synthesize_speech(text, language, voice_profile, quality)`: Generate audio using appropriate TTS service
    - `select_tts_service(subscription_tier)`: Choose Google TTS (standard) or ElevenLabs (premium)
    - `check_audio_cache(text, language, voice)`: Check S3 for cached audio

    **Configuration**:
    - Standard: Google TTS, 24kHz sample rate
    - Premium: ElevenLabs, 44.1kHz sample rate
    - Audio format: MP3 for delivery, WAV for storage
    - Cache duration: 1 hour in S3

    **Dependencies**: Google TTS API, ElevenLabs API, S3 (audio storage), Redis (cache index)

    #### 4. Audio Processing Worker
    **Responsibilities**: Audio preprocessing, noise reduction, format conversion, quality analysis

    **Key Methods**:
    - `preprocess_audio(audio_data)`: Apply noise suppression, echo cancellation, gain control
    - `convert_format(audio_data, target_format)`: Convert between audio formats
    - `analyze_quality(audio_data)`: Assess audio quality metrics (SNR, clipping, etc.)

    **Dependencies**: FFmpeg, librosa (audio processing), S3

    ### Custom React Hooks

    #### useAuth
    ```typescript
    interface UseAuth {
    user: User | null;
    isAuthenticated: boolean;
    login: (email: string, password: string) => Promise<void>;
    logout: () => Promise<void>;
    register: (email: string, password: string, name: string) => Promise<void>;
    refreshToken: () => Promise<void>;
    }
    ```

    #### useWebSocket
    ```typescript
    interface UseWebSocket {
    isConnected: boolean;
    latency: number;
    sendAudioStream: (audioData: ArrayBuffer) => void;
    changeLanguage: (language: string) => void;
    toggleAudio: (enabled: boolean) => void;
    leaveMeeting: () => void;
    on: (event: string, handler: Function) => void;
    off: (event: string, handler: Function) => void;
    }
    ```

    #### useAudioCapture
    ```typescript
    interface UseAudioCapture {
    isCapturing: boolean;
    audioLevel: number;
    startCapture: () => Promise<void>;
    stopCapture: () => void;
    getAudioStream: () => MediaStream | null;
    }
    ```

    #### useTranslation
    ```typescript
    interface UseTranslation {
    transcripts: Transcript[];
    translations: Translation[];
    currentSpeaker: string | null;
    addTranscript: (transcript: Transcript) => void;
    addTranslation: (translation: Translation) => void;
    }
    ```

    #### useMeeting
    ```typescript
    interface UseMeeting {
    meeting: Meeting | null;
    participants: Participant[];
    isHost: boolean;
    createMeeting: (name: string, languages: string[]) => Promise<string>;
    joinMeeting: (meetingId: string) => Promise<void>;
    endMeeting: () => Promise<void>;
    }
    ```

    ## Data Models

    ### PostgreSQL Schema

    #### users
    ```sql
    CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
    );

    CREATE INDEX idx_users_email ON users(email);
    CREATE INDEX idx_users_created_at ON users(created_at);
    ```

    #### user_preferences
    ```sql
    CREATE TABLE user_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    default_language VARCHAR(10) DEFAULT 'en',
    voice_profile VARCHAR(50),
    audio_quality VARCHAR(20) DEFAULT 'standard',
    noise_suppression BOOLEAN DEFAULT TRUE,
    auto_join_audio BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id)
    );

    CREATE INDEX idx_user_preferences_user_id ON user_preferences(user_id);
    ```

    #### meetings
    ```sql
    CREATE TABLE meetings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    host_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    started_at TIMESTAMP,
    ended_at TIMESTAMP,
    max_participants INTEGER DEFAULT 20,
    recording_enabled BOOLEAN DEFAULT FALSE
    );

    CREATE INDEX idx_meetings_host_user_id ON meetings(host_user_id);
    CREATE INDEX idx_meetings_status ON meetings(status);
    CREATE INDEX idx_meetings_created_at ON meetings(created_at);
    ```

    #### meeting_participants
    ```sql
    CREATE TABLE meeting_participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID REFERENCES meetings(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    target_language VARCHAR(10) NOT NULL,
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    left_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    audio_enabled BOOLEAN DEFAULT TRUE,
    UNIQUE(meeting_id, user_id)
    );

    CREATE INDEX idx_meeting_participants_meeting_id ON meeting_participants(meeting_id);
    CREATE INDEX idx_meeting_participants_user_id ON meeting_participants(user_id);
    ```

    #### subscriptions
    ```sql
    CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    tier VARCHAR(20) NOT NULL CHECK (tier IN ('free', 'pro', 'enterprise')),
    status VARCHAR(20) DEFAULT 'active',
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    monthly_translation_minutes INTEGER DEFAULT 100,
    used_translation_minutes INTEGER DEFAULT 0,
    stripe_subscription_id VARCHAR(255),
    UNIQUE(user_id)
    );

    CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
    CREATE INDEX idx_subscriptions_tier ON subscriptions(tier);
    ```

    #### api_usage_logs
    ```sql
    CREATE TABLE api_usage_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    meeting_id UUID REFERENCES meetings(id) ON DELETE SET NULL,
    service_type VARCHAR(50) NOT NULL,
    api_call_count INTEGER DEFAULT 1,
    duration_seconds DECIMAL(10, 3),
    cost_usd DECIMAL(10, 4),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX idx_api_usage_logs_user_id ON api_usage_logs(user_id);
    CREATE INDEX idx_api_usage_logs_meeting_id ON api_usage_logs(meeting_id);
    CREATE INDEX idx_api_usage_logs_created_at ON api_usage_logs(created_at);
    ```

    #### oauth_integrations
    ```sql
    CREATE TABLE oauth_integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    platform VARCHAR(50) NOT NULL,
    access_token TEXT NOT NULL,
    refresh_token TEXT,
    token_expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, platform)
    );

    CREATE INDEX idx_oauth_integrations_user_id ON oauth_integrations(user_id);
    ```

    ### Redis Data Structures

    #### Session Management
    ```
    Key: session:{user_id}
    Type: Hash
    Fields:
    - access_token: JWT access token
    - refresh_token: JWT refresh token
    - expires_at: Token expiration timestamp
    - websocket_id: Active WebSocket connection ID
    TTL: 24 hours
    ```

    #### Translation Cache
    ```
    Key: translation:{hash(source_text)}:{source_lang}:{target_lang}
    Type: String
    Value: Translated text
    TTL: 24 hours
    ```

    #### Audio Processing Queue
    ```
    Key: queue:audio_processing
    Type: List
    Value: JSON {meeting_id, user_id, audio_url, timestamp}
    ```

    #### Rate Limiting
    ```
    Key: ratelimit:{user_id}:{endpoint}
    Type: String
    Value: Request count
    TTL: 60 seconds
    ```

    #### Active Meeting Participants
    ```
    Key: meeting:{meeting_id}:participants
    Type: Set
    Value: user_id
    TTL: 24 hours
    ```

    ### MongoDB Collections

    #### meeting_transcripts
    ```javascript
    {
    _id: ObjectId,
    meeting_id: UUID,
    entries: [
        {
        timestamp: ISODate,
        speaker_id: UUID,
        speaker_name: String,
        original_text: String,
        source_language: String,
        confidence: Number,
        translations: [
            {
            target_language: String,
            translated_text: String,
            translation_service: String
            }
        ]
        }
    ],
    created_at: ISODate,
    updated_at: ISODate
    }

    // Indexes
    db.meeting_transcripts.createIndex({ meeting_id: 1 })
    db.meeting_transcripts.createIndex({ "entries.timestamp": 1 })
    ```

    #### translation_logs
    ```javascript
    {
    _id: ObjectId,
    meeting_id: UUID,
    user_id: UUID,
    source_text: String,
    source_language: String,
    target_language: String,
    translated_text: String,
    translation_service: String,
    latency_ms: Number,
    cache_hit: Boolean,
    timestamp: ISODate
    }

    // Indexes
    db.translation_logs.createIndex({ meeting_id: 1, timestamp: 1 })
    db.translation_logs.createIndex({ user_id: 1, timestamp: 1 })
    ```

    #### error_logs
    ```javascript
    {
    _id: ObjectId,
    error_type: String,
    service: String,
    meeting_id: UUID,
    user_id: UUID,
    error_message: String,
    stack_trace: String,
    context: Object,
    timestamp: ISODate
    }

    // Indexes
    db.error_logs.createIndex({ timestamp: -1 })
    db.error_logs.createIndex({ error_type: 1, timestamp: -1 })
    ```

    #### analytics_events
    ```javascript
    {
    _id: ObjectId,
    event_type: String,
    meeting_id: UUID,
    user_id: UUID,
    properties: {
        duration_seconds: Number,
        participant_count: Number,
        total_translations: Number,
        average_latency_ms: Number,
        languages_used: [String]
    },
    timestamp: ISODate
    }

    // Indexes
    db.analytics_events.createIndex({ event_type: 1, timestamp: -1 })
    db.analytics_events.createIndex({ user_id: 1, timestamp: -1 })
    ```

    ### AWS S3 Bucket Structure

    ```
    lingzen-audio/
    ├── original-audio/
    │   └── {meeting_id}/
    │       └── {user_id}/
    │           └── {timestamp}_{chunk_id}.wav
    ├── processed-audio/
    │   └── {meeting_id}/
    │       └── {translation_id}.mp3
    ├── voice-profiles/
    │   └── {user_id}/
    │       └── profile.json
    └── exported-recordings/
        └── {meeting_id}/
            ├── full_recording.mp3
            └── transcript.json
    ```

    ## Correctness Properties


    A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

    ### Authentication and Session Management Properties

    **Property 1: User registration creates encrypted accounts**
    *For any* valid user registration data (email, password, name), creating an account should result in a stored user record with a bcrypt-hashed password (minimum 12 rounds) that cannot be reversed to the original password.
    **Validates: Requirements 1.1, 13.4**

    **Property 2: Invalid passwords are rejected**
    *For any* password that is less than 8 characters or missing required character types, registration attempts should be rejected with a validation error.
    **Validates: Requirements 1.6**

    **Property 3: Login generates valid tokens**
    *For any* registered user with valid credentials, logging in should generate both a JWT access token and refresh token that can be used to authenticate subsequent requests.
    **Validates: Requirements 1.2, 1.3**

    **Property 4: Token refresh round-trip**
    *For any* user session, if the access token expires, using the refresh token should issue a new valid access token that grants the same access as the original.
    **Validates: Requirements 1.4**

    **Property 5: Logout invalidates tokens**
    *For any* authenticated user, after logging out, their access token and refresh token should no longer authenticate requests.
    **Validates: Requirements 1.8**

    **Property 6: Input sanitization prevents injection**
    *For any* user input containing SQL injection patterns, script tags, or other malicious content, the system should sanitize the input before processing or storage.
    **Validates: Requirements 13.7**

    ### Meeting Lifecycle Properties

    **Property 7: Meeting creation generates unique IDs**
    *For any* valid meeting parameters (name, languages), creating a meeting should generate a unique meeting ID and store complete meeting metadata in the database.
    **Validates: Requirements 2.1**

    **Property 8: Joining adds to participant list**
    *For any* valid meeting ID and authenticated user, joining the meeting should add the user to the meeting_participants table and establish a WebSocket connection.
    **Validates: Requirements 2.2**

    **Property 9: Leaving removes from participant list**
    *For any* active participant in a meeting, when they leave, they should be removed from the active participant list and other participants should receive a "participant_left" event.
    **Validates: Requirements 2.4, 9.2**

    **Property 10: Host ending closes all connections**
    *For any* active meeting, when the host ends it, all participant WebSocket connections should be closed and the meeting status should be marked as "completed".
    **Validates: Requirements 2.3**

    **Property 11: Meeting history retrieval**
    *For any* user, requesting meeting history should return all meetings they participated in, with each entry containing timestamps, participant counts, and meeting metadata.
    **Validates: Requirements 2.6**

    ### Audio Processing Pipeline Properties

    **Property 12: Audio encoding for transmission**
    *For any* captured audio data, encoding it for WebSocket transmission should produce valid base64-encoded data that can be decoded back to the original audio format.
    **Validates: Requirements 3.2**

    **Property 13: Audio toggle state management**
    *For any* participant, toggling audio off should immediately stop audio capture and transmission, and toggling back on should resume capture.
    **Validates: Requirements 3.5**

    **Property 14: STT processing workflow**
    *For any* audio chunk dequeued for processing, the system should upload it to S3, call the Speech-to-Text API, and store the resulting transcript with timestamp and speaker ID.
    **Validates: Requirements 4.1, 4.2**

    **Property 15: STT retry with exponential backoff**
    *For any* failed speech-to-text API call, the system should retry up to 3 times with exponentially increasing delays before marking the operation as failed.
    **Validates: Requirements 4.3, 15.1**

    **Property 16: Transcript broadcasting**
    *For any* completed transcription, the system should broadcast a "transcript" event to all active participants in the meeting with speaker name, timestamp, and original text.
    **Validates: Requirements 4.5, 8.1**

    **Property 17: Empty transcription filtering**
    *For any* transcription result that contains only silence or non-speech sounds (empty or whitespace-only text), the system should filter it out and not broadcast it to participants.
    **Validates: Requirements 4.6**

    ### Translation Pipeline Properties

    **Property 18: Translation to all target languages**
    *For any* transcribed text and meeting with N participants, the system should generate N-1 translations (one for each participant's target language different from the source).
    **Validates: Requirements 5.1**

    **Property 19: Translation service fallback**
    *For any* translation request where DeepL API fails or returns an error, the system should automatically fallback to Google Translate API without user intervention.
    **Validates: Requirements 5.2**

    **Property 20: Translation caching**
    *For any* completed translation, the system should store it in Redis with a key based on source text, source language, and target language, with 24-hour expiration.
    **Validates: Requirements 5.4**

    **Property 21: Language change updates routing**
    *For any* participant who changes their target language mid-meeting, all subsequent translations should be delivered in the new target language.
    **Validates: Requirements 5.5**

    **Property 22: Translation routing to correct participants**
    *For any* completed translation, the system should send it only to participants whose target language matches the translation's target language.
    **Validates: Requirements 8.2**

    ### Text-to-Speech Properties

    **Property 23: TTS service selection by tier**
    *For any* user with standard subscription, TTS should use Google TTS, and for any user with pro or enterprise subscription, TTS should use ElevenLabs API.
    **Validates: Requirements 6.1, 6.2, 10.2**

    **Property 24: TTS audio transmission**
    *For any* synthesized speech audio, the system should encode it and transmit via WebSocket to the target participants as a "translated_audio" event.
    **Validates: Requirements 6.3**

    **Property 25: TTS retry and fallback**
    *For any* failed TTS synthesis, the system should retry once, and if that fails, deliver the translation as text-only without audio.
    **Validates: Requirements 6.4**

    **Property 26: TTS audio caching**
    *For any* translated text that has been synthesized within the last hour, requesting TTS for the same text should retrieve the cached audio from S3 instead of re-synthesizing.
    **Validates: Requirements 6.5**

    **Property 27: Voice profile application**
    *For any* participant with custom voice preferences, TTS synthesis should use the selected voice profile from their user preferences.
    **Validates: Requirements 6.6**

    **Property 28: TTS sample rate by tier**
    *For any* standard subscription user, synthesized audio should be at 24kHz sample rate, and for any premium subscription user, audio should be at 44.1kHz sample rate.
    **Validates: Requirements 19.4, 19.5**

    ### WebSocket Communication Properties

    **Property 29: WebSocket connection establishment**
    *For any* participant joining a meeting, a WebSocket connection should be established with JWT authentication, and a "connected" event should be sent with session information.
    **Validates: Requirements 7.1, 7.2**

    **Property 30: Audio stream event processing**
    *For any* "audio_stream" event received via WebSocket, the system should queue it for translation pipeline processing within the audio processing queue.
    **Validates: Requirements 7.3**

    **Property 31: WebSocket reconnection attempts**
    *For any* dropped WebSocket connection, the system should attempt automatic reconnection up to 5 times with exponential backoff before giving up.
    **Validates: Requirements 7.5**

    **Property 32: Ping-pong round-trip**
    *For any* "ping" event sent by a participant, the system should respond with a "pong" event, maintaining connection liveness.
    **Validates: Requirements 7.6**

    ### Transcript Storage and Retrieval Properties

    **Property 33: Transcript persistence with confidence scores**
    *For any* meeting that ends, the complete transcript should be saved to MongoDB with all entries including speaker IDs, timestamps, original text, translations, and confidence scores from the Speech Service.
    **Validates: Requirements 8.4, 8.6**

    **Property 34: Past transcript retrieval**
    *For any* user requesting a past meeting transcript, the system should retrieve it from MongoDB with all original text and translations for all participants.
    **Validates: Requirements 8.5**

    ### Participant Management Properties

    **Property 35: Participant join notifications**
    *For any* participant joining a meeting, all existing participants should receive a "participant_joined" event with the new participant's information.
    **Validates: Requirements 9.1**

    **Property 36: Language change notifications**
    *For any* participant changing their target language, all other participants should receive a "participant_updated" event with the new language preference.
    **Validates: Requirements 9.3**

    **Property 37: Speaking status broadcasts**
    *For any* participant who starts speaking, all other participants should receive a "speaking_status" event indicating the active speaker.
    **Validates: Requirements 9.4**

    **Property 38: Participant list completeness**
    *For any* request for the participant list, the response should include user name, language preference, audio status, and connection quality for all active participants.
    **Validates: Requirements 9.6**

    ### Subscription and Usage Tracking Properties

    **Property 39: Free tier quota enforcement**
    *For any* free tier user who has used 100 or more translation minutes in the current month, additional translation requests should be blocked until the next billing cycle.
    **Validates: Requirements 10.1**

    **Property 40: Enterprise tier participant limits**
    *For any* enterprise tier user creating a meeting, the system should allow up to 50 participants, while standard and pro tiers should be limited to 20 participants.
    **Validates: Requirements 10.3**

    **Property 41: API usage logging**
    *For any* API call made to the system, an entry should be logged to the api_usage_logs table with timestamp, user ID, service type, and duration.
    **Validates: Requirements 10.4, 16.1**

    **Property 42: Usage statistics aggregation**
    *For any* user requesting usage statistics, the system should aggregate data from api_usage_logs and return monthly totals for translation minutes, API calls, and costs.
    **Validates: Requirements 10.5, 16.6**

    **Property 43: Subscription upgrade immediate effect**
    *For any* user upgrading their subscription tier, the new tier limits and features should be applied immediately to their account.
    **Validates: Requirements 10.6**

    **Property 44: Subscription cancellation grace period**
    *For any* user canceling their subscription, access to paid features should be maintained until the end of the current billing period.
    **Validates: Requirements 10.7**

    ### Integration Management Properties

    **Property 45: Integration disconnection cleanup**
    *For any* user disconnecting a platform integration, the system should revoke OAuth tokens and remove the integration record from the database.
    **Validates: Requirements 11.3**

    **Property 46: Integration list retrieval**
    *For any* user requesting their integrations, the system should return all connected platforms with connection status and last sync timestamp.
    **Validates: Requirements 11.4**

    ### Recording and Export Properties

    **Property 47: Recording audio capture**
    *For any* meeting with recording enabled, all audio streams from all participants should be captured and stored in the S3 original-audio bucket organized by meeting ID and user ID.
    **Validates: Requirements 14.1**

    **Property 48: Recording generation on meeting end**
    *For any* meeting that ends with recording enabled, the system should generate a downloadable audio file containing all original and translated audio streams.
    **Validates: Requirements 14.3**

    **Property 49: Transcript export format support**
    *For any* transcript export request, the system should be able to generate the transcript in JSON, PDF, and SRT subtitle formats.
    **Validates: Requirements 14.4**

    **Property 50: Recording retention by tier**
    *For any* stored recording, it should be retained for 90 days for pro tier users and 365 days for enterprise tier users before automatic deletion.
    **Validates: Requirements 14.5**

    ### Error Handling Properties

    **Property 51: Error logging on retry exhaustion**
    *For any* operation that fails after all retry attempts are exhausted, an error entry should be logged to MongoDB error_logs collection with error type, service, context, and stack trace.
    **Validates: Requirements 15.2, 16.5**

    **Property 52: Critical error notifications**
    *For any* critical error that affects meeting functionality, the system should send an "error" event to affected participants with a user-friendly error message.
    **Validates: Requirements 15.3**

    ### Monitoring and Analytics Properties

    **Property 53: Pipeline stage timing metrics**
    *For any* translation pipeline execution, timing metrics should be recorded for each stage (STT, translation, TTS) and stored for performance analysis.
    **Validates: Requirements 16.2**

    **Property 54: Meeting analytics on completion**
    *For any* meeting that ends, analytics events should be stored to MongoDB including meeting duration, participant count, total translations, average latency, and languages used.
    **Validates: Requirements 16.3**

    ### Resource Management Properties

    **Property 55: Audio file cleanup by retention**
    *For any* audio files in S3 that exceed the retention period for their associated subscription tier, the system should delete them to free storage space.
    **Validates: Requirements 17.4**

    ### Language Support Properties

    **Property 56: Automatic language detection**
    *For any* audio transcription where the source language is not specified, the system should use the Translation Service's language detection feature to identify the language.
    **Validates: Requirements 18.1**

    **Property 57: Low confidence language prompts**
    *For any* language detection result with confidence below 80%, the system should prompt the participant to manually confirm or select their language.
    **Validates: Requirements 18.2**

    **Property 58: Target language validation**
    *For any* participant selecting a target language, the system should validate it against the list of supported languages and reject unsupported languages with an error message.
    **Validates: Requirements 18.4, 18.5**

    ### Rate Limiting Properties

    **Property 59: Tier-based rate limiting**
    *For any* user making API requests, the system should enforce rate limits of 100 req/min for free tier, 500 req/min for pro tier, and 2000 req/min for enterprise tier.
    **Validates: Requirements 20.1, 20.2, 20.3**

    **Property 60: Rate limit exceeded response**
    *For any* user exceeding their rate limit, the system should return HTTP 429 status code with a retry-after header indicating when they can retry.
    **Validates: Requirements 20.4**

    **Property 61: Monthly quota enforcement**
    *For any* user exceeding their monthly translation quota, the system should block further translation requests and send a notification about quota exhaustion.
    **Validates: Requirements 20.6**

    ## Error Handling

    ### Error Categories

    #### 1. External Service Failures
    **Speech-to-Text Errors**:
    - API unavailable: Retry with exponential backoff (3 attempts), then notify participants and continue meeting without transcription
    - Invalid audio format: Log error, notify participant about audio quality issues
    - Quota exceeded: Alert administrators, gracefully degrade to text-only mode

    **Translation Service Errors**:
    - DeepL API failure: Automatic fallback to Google Translate API
    - Both services unavailable: Deliver original language transcripts to all participants with error notification
    - Unsupported language pair: Return error with list of supported languages

    **Text-to-Speech Errors**:
    - TTS API failure: Retry once, then fallback to text-only translation delivery
    - Voice profile not found: Use default voice for the target language
    - Audio generation timeout: Cancel TTS, deliver text translation

    #### 2. WebSocket Connection Errors
    **Connection Drops**:
    - Automatic reconnection with exponential backoff (5 attempts)
    - Queue missed events during disconnection
    - Replay missed events on reconnection
    - Notify user of connection status

    **Authentication Failures**:
    - Invalid JWT: Return 401, prompt user to re-authenticate
    - Expired token: Attempt token refresh, fallback to re-authentication
    - Missing token: Reject connection with clear error message

    #### 3. Database Errors
    **PostgreSQL Failures**:
    - Connection timeout: Retry with connection pool, return 503 if exhausted
    - Query timeout: Cancel query, log slow query, return error
    - Constraint violation: Return 400 with specific validation error

    **Redis Failures**:
    - Cache miss: Fetch from primary source (database or API)
    - Connection failure: Continue without cache, queue writes for retry
    - Memory full: Evict LRU entries, alert administrators

    **MongoDB Failures**:
    - Write failure: Queue write for retry, continue operation
    - Read failure: Return cached data if available, otherwise error
    - Connection loss: Reconnect automatically, buffer operations

    #### 4. Resource Exhaustion
    **Memory Limits**:
    - Audio buffer full: Drop oldest chunks, notify participant
    - Queue overflow: Apply backpressure, slow down audio capture
    - Cache full: Evict LRU entries

    **Storage Limits**:
    - S3 quota exceeded: Delete old recordings per retention policy
    - Disk full: Alert administrators, reject new uploads
    - Database storage full: Alert administrators, implement cleanup

    #### 5. Validation Errors
    **Input Validation**:
    - Invalid email format: Return 400 with specific field error
    - Password too weak: Return 400 with password requirements
    - Malicious input detected: Sanitize and log security event

    **Business Logic Validation**:
    - Meeting full: Return 403 with participant limit message
    - Quota exceeded: Return 429 with quota information
    - Unsupported operation: Return 400 with explanation

    ### Error Response Format

    All API errors follow a consistent JSON structure:

    ```json
    {
    "error": {
        "code": "ERROR_CODE",
        "message": "Human-readable error message",
        "details": {
        "field": "specific_field",
        "reason": "detailed_reason"
        },
        "timestamp": "2024-01-15T10:30:00Z",
        "request_id": "uuid"
    }
    }
    ```

    ### Error Logging Strategy

    **Structured Logging**:
    - All errors logged to MongoDB error_logs collection
    - Include: timestamp, error type, service, user ID, meeting ID, stack trace, context
    - Severity levels: DEBUG, INFO, WARNING, ERROR, CRITICAL

    **Alert Thresholds**:
    - Error rate > 5% of requests: Alert on-call engineer
    - External API failure: Alert immediately
    - Database connection failure: Alert immediately
    - Quota approaching limit: Alert administrators

    ### Graceful Degradation

    **Service Degradation Hierarchy**:
    1. Full functionality: All services operational
    2. Text-only mode: STT works, TTS fails → deliver text translations
    3. Original language mode: Translation fails → deliver original transcripts
    4. Transcript-only mode: All AI services fail → basic text chat
    5. Meeting continues: Core WebSocket communication maintained

    ## Testing Strategy

    ### Dual Testing Approach

    The LINGZEN platform requires both unit testing and property-based testing for comprehensive coverage:

    **Unit Tests**: Verify specific examples, edge cases, error conditions, and integration points
    - Focus on concrete scenarios and boundary conditions
    - Test specific error handling paths
    - Verify integration between components
    - Validate specific data transformations

    **Property Tests**: Verify universal properties across all inputs
    - Test behaviors that should hold for all valid inputs
    - Use randomized input generation for comprehensive coverage
    - Validate invariants and round-trip properties
    - Each property test must run minimum 100 iterations

    ### Property-Based Testing Configuration

    **Framework Selection**:
    - Python backend: Use `hypothesis` library for property-based testing
    - TypeScript frontend: Use `fast-check` library for property-based testing

    **Test Configuration**:
    ```python
    # Python example with hypothesis
    from hypothesis import given, settings
    import hypothesis.strategies as st

    @settings(max_examples=100)
    @given(
        email=st.emails(),
        password=st.text(min_size=8, max_size=128),
        name=st.text(min_size=1, max_size=255)
    )
    def test_user_registration_creates_encrypted_accounts(email, password, name):
        """
        Feature: lingzen-translation-platform
        Property 1: User registration creates encrypted accounts
        
        For any valid user registration data, creating an account should
        result in a stored user record with a bcrypt-hashed password.
        """
        # Test implementation
        pass
    ```

    ```typescript
    // TypeScript example with fast-check
    import fc from 'fast-check';

    describe('Feature: lingzen-translation-platform, Property 29: WebSocket connection establishment', () => {
    it('should establish connection for any participant joining', () => {
        fc.assert(
        fc.property(
            fc.uuid(),
            fc.string({ minLength: 1, maxLength: 255 }),
            fc.constantFrom('en', 'es', 'fr', 'de', 'ja'),
            async (userId, userName, targetLanguage) => {
            // Test implementation
            }
        ),
        { numRuns: 100 }
        );
    });
    });
    ```

    **Test Tagging Convention**:
    Every property-based test must include a comment tag referencing the design document property:
    ```
    Feature: lingzen-translation-platform, Property {number}: {property_text}
    ```

    ### Testing Scope by Component

    #### Authentication Service Tests
    **Unit Tests**:
    - Password reset email generation
    - JWT token structure validation
    - Specific invalid email formats
    - Edge cases: empty strings, special characters

    **Property Tests**:
    - Property 1: User registration creates encrypted accounts
    - Property 2: Invalid passwords are rejected
    - Property 3: Login generates valid tokens
    - Property 4: Token refresh round-trip
    - Property 5: Logout invalidates tokens
    - Property 6: Input sanitization prevents injection

    #### Meeting Service Tests
    **Unit Tests**:
    - Meeting creation with minimum/maximum participants
    - Joining meeting at capacity (20 participants)
    - Host permissions validation
    - WebSocket event message formats

    **Property Tests**:
    - Property 7: Meeting creation generates unique IDs
    - Property 8: Joining adds to participant list
    - Property 9: Leaving removes from participant list
    - Property 10: Host ending closes all connections
    - Property 11: Meeting history retrieval

    #### Translation Pipeline Tests
    **Unit Tests**:
    - Empty audio chunk handling
    - Specific language pair translations
    - Cache hit/miss scenarios
    - Fallback service activation

    **Property Tests**:
    - Property 12: Audio encoding for transmission
    - Property 14: STT processing workflow
    - Property 15: STT retry with exponential backoff
    - Property 16: Transcript broadcasting
    - Property 17: Empty transcription filtering
    - Property 18: Translation to all target languages
    - Property 19: Translation service fallback
    - Property 20: Translation caching
    - Property 21: Language change updates routing
    - Property 22: Translation routing to correct participants

    #### TTS Service Tests
    **Unit Tests**:
    - Specific voice profile selection
    - Audio format conversion
    - Cache expiration timing
    - Service selection edge cases

    **Property Tests**:
    - Property 23: TTS service selection by tier
    - Property 24: TTS audio transmission
    - Property 25: TTS retry and fallback
    - Property 26: TTS audio caching
    - Property 27: Voice profile application
    - Property 28: TTS sample rate by tier

    #### WebSocket Communication Tests
    **Unit Tests**:
    - Specific event message parsing
    - Connection timeout scenarios
    - Authentication failure cases
    - Reconnection backoff timing

    **Property Tests**:
    - Property 29: WebSocket connection establishment
    - Property 30: Audio stream event processing
    - Property 31: WebSocket reconnection attempts
    - Property 32: Ping-pong round-trip

    #### Subscription Management Tests
    **Unit Tests**:
    - Billing cycle transitions
    - Specific quota thresholds
    - Tier upgrade/downgrade scenarios
    - Payment webhook handling

    **Property Tests**:
    - Property 39: Free tier quota enforcement
    - Property 40: Enterprise tier participant limits
    - Property 41: API usage logging
    - Property 42: Usage statistics aggregation
    - Property 43: Subscription upgrade immediate effect
    - Property 44: Subscription cancellation grace period

    #### Rate Limiting Tests
    **Unit Tests**:
    - Exact rate limit boundaries
    - Retry-after header calculation
    - Distributed rate limiting consistency

    **Property Tests**:
    - Property 59: Tier-based rate limiting
    - Property 60: Rate limit exceeded response
    - Property 61: Monthly quota enforcement

    ### Integration Testing

    **End-to-End Meeting Flow**:
    1. User registration and login
    2. Meeting creation
    3. Multiple participants joining
    4. Audio streaming and translation
    5. Transcript viewing
    6. Meeting end and cleanup

    **External Service Integration**:
    - Mock Google Speech-to-Text API responses
    - Mock DeepL and Google Translate API responses
    - Mock Google TTS and ElevenLabs API responses
    - Test fallback scenarios

    **Database Integration**:
    - PostgreSQL transaction handling
    - Redis cache consistency
    - MongoDB document structure validation
    - S3 upload/download operations

    ### Performance Testing

    **Load Testing**:
    - 100 concurrent meetings with 20 participants each
    - Sustained audio streaming for 30 minutes
    - Translation pipeline throughput measurement
    - Database query performance under load

    **Latency Testing**:
    - End-to-end latency measurement (target: < 3 seconds)
    - Individual pipeline stage timing
    - WebSocket message delivery latency
    - Cache hit/miss performance comparison

    ### Security Testing

    **Authentication Testing**:
    - JWT token tampering attempts
    - Expired token handling
    - Brute force protection
    - Session hijacking prevention

    **Input Validation Testing**:
    - SQL injection attempts
    - XSS attack vectors
    - Command injection patterns
    - Path traversal attempts

    **Rate Limiting Testing**:
    - Distributed rate limit bypass attempts
    - Token bucket algorithm validation
    - Quota enforcement accuracy

    ### Monitoring and Observability

    **Metrics Collection**:
    - Prometheus metrics for all services
    - Custom metrics: translation latency, cache hit rate, error rate
    - Resource utilization: CPU, memory, network
    - External API call counts and latency

    **Logging**:
    - Structured JSON logs for all services
    - Log levels: DEBUG, INFO, WARNING, ERROR, CRITICAL
    - Correlation IDs for request tracing
    - Sensitive data redaction

    **Alerting**:
    - Error rate thresholds
    - Latency SLA violations
    - External service failures
    - Resource exhaustion warnings
