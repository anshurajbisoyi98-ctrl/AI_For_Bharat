# Design Document: MargMitra (The Civic Neural Grid) - MERN + India Stack Hybrid

## Overview

MargMitra is a civic safety platform that combines the MERN stack (MongoDB, Express.js, React, Node.js) with India Stack protocols to create a scalable, accessible civic technology solution. The hybrid architecture leverages modern web technologies for the application layer while integrating government-backed infrastructure for linguistic inclusion and decentralized coordination.

**Technology Stack**:
- **Frontend**: React 18 Progressive Web App (PWA)
- **Backend**: Node.js 18+ with Express.js 4.x
- **Application Database**: MongoDB 6.0+ (users, sessions, observations)
- **Spatial Database**: PostgreSQL 14+ with PostGIS extension (safety hexagons, routes)
- **Cache**: Redis 7.0+ (API responses, session data)
- **Real-time**: Socket.IO 4.x (WebSocket connections)
- **Maps**: Leaflet.js with PMTiles on Cloudflare R2
- **Voice**: Bhashini API (primary) + Web Speech API (fallback)
- **NER**: IndicBERT via REST API
- **Discovery**: Beckn Protocol for responder coordination
- **Spatial Indexing**: Uber H3 (Resolution 9/10)
- **Authentication**: JWT with bcrypt + optional Aadhaar

**Key Design Principles**:
- **MERN for Application**: React UI, Node.js services, MongoDB for flexible data
- **India Stack for Civic Infrastructure**: Bhashini, Beckn, PostGIS for government integration
- **Progressive Enhancement**: Core features work without India Stack, enhanced when available
- **Offline-First**: Service workers + IndexedDB for offline capability
- **Privacy by Design**: H3 anonymization, HTTPS encryption
- **Multilingual**: Voice-first in 10+ Indian languages

## Architecture

### System Architecture Diagram

```mermaid
flowchart TB
    subgraph Frontend["React PWA"]
        A[React Components]
        B[Redux Toolkit]
        C[Service Worker]
        D[IndexedDB]
        E[Leaflet Maps]
    end
    
    subgraph Backend["Node.js + Express"]
        F[Express REST API]
        G[Socket.IO Server]
        H[JWT Auth]
        I[Bhashini Client]
        J[IndicBERT Client]
        K[Beckn Client]
        L[H3 Service]
    end
    
    subgraph Databases["Hybrid DB"]
        M[MongoDB]
        N[PostgreSQL PostGIS]
        O[Redis]
    end
    
    subgraph IndiaStack["India Stack"]
        P[Bhashini API]
        Q[Beckn Gateway]
    end
    
    subgraph External["External"]
        R[Cloudflare R2]
        S[OSM Data]
    end
    
    A --> F
    B --> C
    C --> D
    E --> R
    F --> G
    F --> H
    F --> I
    F --> J
    F --> K
    F --> L
    I --> P
    K --> Q
    F --> M
    F --> N
    F --> O
    G --> A
```

### Layer Descriptions

**Frontend Layer (React PWA)**: Single-page application with offline capabilities via service workers. Redux Toolkit manages state, IndexedDB caches data locally, Leaflet renders interactive maps with PMTiles.

**Backend Layer (Node.js + Express)**: RESTful API server with microservice architecture. Express handles HTTP requests, Socket.IO provides real-time bidirectional communication. Services integrate with India Stack APIs.

**Database Layer (Hybrid)**: MongoDB stores user accounts, sessions, and crowdsourced observations. PostgreSQL with PostGIS handles spatial queries for safety hexagons and route calculation. Redis caches frequently accessed data.

**India Stack Layer**: Bhashini provides multilingual voice ASR/TTS. Beckn Protocol enables decentralized responder discovery. Both are accessed via REST/WebSocket APIs from Node.js backend.

**External Services**: Cloudflare R2 hosts PMTiles for serverless vector map delivery. OpenStreetMap provides base map data.

## Components and Interfaces

### 1. Frontend Components (React)

**Component Structure**:

```
src/
├── components/
│   ├── Map/
│   │   ├── MapView.tsx          # Main map component with Leaflet
│   │   ├── SafetyOverlay.tsx    # H3 hexagon safety visualization
│   │   └── RouteDisplay.tsx     # Safe route rendering
│   ├── Voice/
│   │   ├── VoiceInput.tsx       # Voice command interface
│   │   └── LanguageSelector.tsx # Language preference
│   ├── SOS/
│   │   ├── SOSButton.tsx        # Emergency broadcast trigger
│   │   └── ResponderList.tsx    # Available responders
│   ├── Safety/
│   │   ├── ObservationForm.tsx  # Crowdsource safety reports
│   │   └── SafetyScore.tsx      # Display hexagon safety scores
│   └── Auth/
│       ├── Login.tsx
│       └── Register.tsx
├── store/
│   ├── slices/
│   │   ├── authSlice.ts
│   │   ├── safetySlice.ts
│   │   ├── sosSlice.ts
│   │   └── mapSlice.ts
│   └── store.ts
├── services/
│   ├── api.ts                   # Axios API client
│   ├── socket.ts                # Socket.IO client
│   └── storage.ts               # IndexedDB wrapper
└── utils/
    ├── h3.ts                    # H3 utilities
    └── geolocation.ts           # Browser geolocation
```

**Key Interfaces**:

```typescript
// types/safety.ts
export interface SafetyHexagon {
  h3Index: string
  resolution: 9 | 10
  safetyScore: number
  factors: {
    crime: number
    streetlights: number
    crowd: number
    police: number
  }
  lastUpdated: Date
}

export interface Route {
  coordinates: [number, number][]
  distance: number
  estimatedTime: number
  averageSafetyScore: number
  minimumSafetyScore: number
}

export interface SOSBroadcast {
  id: string
  location: [number, number]
  emergencyType: 'MEDICAL' | 'SAFETY_THREAT' | 'ACCIDENT' | 'FIRE' | 'OTHER'
  severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL'
  status: 'PENDING' | 'RESPONDED' | 'RESOLVED' | 'CANCELLED'
  createdAt: Date
}
```

**Redux Store Setup**:

```typescript
// store/store.ts
import { configureStore } from '@reduxjs/toolkit'
import authReducer from './slices/authSlice'
import safetyReducer from './slices/safetySlice'
import sosReducer from './slices/sosSlice'
import mapReducer from './slices/mapSlice'

export const store = configureStore({
  reducer: {
    auth: authReducer,
    safety: safetyReducer,
    sos: sosReducer,
    map: mapReducer
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['map/setMapInstance'],
        ignoredPaths: ['map.instance']
      }
    })
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch
```

**Service Worker (PWA)**:

```javascript
// public/service-worker.js
const CACHE_NAME = 'margmitra-v1'
const RUNTIME_CACHE = 'margmitra-runtime'

// Cache static assets on install
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll([
        '/',
        '/index.html',
        '/static/js/main.js',
        '/static/css/main.css',
        '/manifest.json',
        '/offline.html'
      ])
    })
  )
})

// Network-first strategy for API calls, cache-first for assets
self.addEventListener('fetch', (event) => {
  const { request } = event
  const url = new URL(request.url)

  // API requests: network-first with cache fallback
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(
      fetch(request)
        .then((response) => {
          const responseClone = response.clone()
          caches.open(RUNTIME_CACHE).then((cache) => {
            cache.put(request, responseClone)
          })
          return response
        })
        .catch(() => caches.match(request))
    )
    return
  }

  // Static assets: cache-first
  event.respondWith(
    caches.match(request).then((response) => {
      return response || fetch(request)
    })
  )
})
```


### 2. Backend API (Node.js + Express)

**Project Structure**:

```
server/
├── src/
│   ├── routes/
│   │   ├── auth.routes.ts
│   │   ├── safety.routes.ts
│   │   ├── sos.routes.ts
│   │   ├── responder.routes.ts
│   │   └── voice.routes.ts
│   ├── controllers/
│   │   ├── auth.controller.ts
│   │   ├── safety.controller.ts
│   │   ├── sos.controller.ts
│   │   └── voice.controller.ts
│   ├── services/
│   │   ├── bhashini.service.ts
│   │   ├── indicbert.service.ts
│   │   ├── beckn.service.ts
│   │   ├── h3.service.ts
│   │   └── safety.service.ts
│   ├── models/
│   │   ├── mongodb/
│   │   │   ├── User.model.ts
│   │   │   ├── Observation.model.ts
│   │   │   └── Session.model.ts
│   │   └── postgres/
│   │       ├── SafetyHexagon.model.ts
│   │       ├── SOSBroadcast.model.ts
│   │       └── Responder.model.ts
│   ├── middleware/
│   │   ├── auth.middleware.ts
│   │   ├── validation.middleware.ts
│   │   └── rateLimit.middleware.ts
│   ├── utils/
│   │   ├── jwt.util.ts
│   │   ├── encryption.util.ts
│   │   └── logger.util.ts
│   ├── config/
│   │   ├── mongodb.config.ts
│   │   ├── postgres.config.ts
│   │   └── redis.config.ts
│   └── app.ts
├── package.json
└── tsconfig.json
```

**API Routes**:

```typescript
// routes/api.routes.ts
import express from 'express'
import { authMiddleware } from '../middleware/auth.middleware'
import { validateRequest } from '../middleware/validation.middleware'
import { rateLimiter } from '../middleware/rateLimit.middleware'

const router = express.Router()

// Authentication
router.post('/auth/register', validateRequest, registerUser)
router.post('/auth/login', validateRequest, loginUser)
router.post('/auth/refresh', refreshToken)
router.post('/auth/logout', authMiddleware, logoutUser)

// Safety Data
router.get('/safety/hexagons', authMiddleware, getSafetyHexagons)
router.get('/safety/hexagons/:h3Index', authMiddleware, getHexagonDetails)
router.post('/safety/observations', authMiddleware, validateRequest, submitObservation)
router.get('/safety/route', authMiddleware, calculateSafeRoute)

// Emergency SOS
router.post('/sos/broadcast', authMiddleware, rateLimiter, broadcastSOS)
router.get('/sos/:id', authMiddleware, getSOSDetails)
router.put('/sos/:id/respond', authMiddleware, respondToSOS)
router.put('/sos/:id/resolve', authMiddleware, resolveS OS)

// Voice Processing
router.post('/voice/transcribe', authMiddleware, transcribeAudio)
router.post('/voice/synthesize', authMiddleware, synthesizeSpeech)
router.post('/voice/extract-entities', authMiddleware, extractEntities)

// Responders
router.post('/responders/register', authMiddleware, registerResponder)
router.get('/responders/nearby', authMiddleware, getNearbyResponders)
router.put('/responders/:id/availability', authMiddleware, updateAvailability)

export default router
```

**Express Server Setup**:

```typescript
// app.ts
import express from 'express'
import cors from 'cors'
import helmet from 'helmet'
import compression from 'compression'
import { createServer } from 'http'
import { Server as SocketIOServer } from 'socket.io'
import mongoose from 'mongoose'
import { Pool } from 'pg'
import Redis from 'ioredis'
import apiRoutes from './routes/api.routes'
import { errorHandler } from './middleware/error.middleware'
import { setupSocketIO } from './socket/socket.handler'

const app = express()
const httpServer = createServer(app)
const io = new SocketIOServer(httpServer, {
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true
  }
})

// Middleware
app.use(helmet())
app.use(cors({ origin: process.env.FRONTEND_URL, credentials: true }))
app.use(compression())
app.use(express.json({ limit: '10mb' }))
app.use(express.urlencoded({ extended: true }))

// Database connections
mongoose.connect(process.env.MONGODB_URI!)
  .then(() => console.log('MongoDB connected'))
  .catch((err) => console.error('MongoDB connection error:', err))

export const pgPool = new Pool({
  connectionString: process.env.POSTGRES_URI,
  max: 20
})

export const redisClient = new Redis(process.env.REDIS_URI!)

// Routes
app.use('/api', apiRoutes)

// Socket.IO setup
setupSocketIO(io)

// Error handling
app.use(errorHandler)

const PORT = process.env.PORT || 5000
httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`)
})
```

**Bhashini Integration Service**:

```typescript
// services/bhashini.service.ts
import axios from 'axios'
import WebSocket from 'ws'

export class BhashiniService {
  private apiKey: string
  private baseURL: string

  constructor() {
    this.apiKey = process.env.BHASHINI_API_KEY!
    this.baseURL = process.env.BHASHINI_BASE_URL!
  }

  // Transcribe audio using Bhashini ASR
  async transcribeAudio(
    audioBuffer: Buffer,
    sourceLanguage: string
  ): Promise<{ text: string; translatedText: string; confidence: number }> {
    try {
      const response = await axios.post(
        `${this.baseURL}/asr`,
        {
          audio: audioBuffer.toString('base64'),
          sourceLanguage,
          targetLanguage: 'en'
        },
        {
          headers: {
            'Authorization': `Bearer ${this.apiKey}`,
            'Content-Type': 'application/json'
          }
        }
      )

      return {
        text: response.data.originalText,
        translatedText: response.data.translatedText,
        confidence: response.data.confidence
      }
    } catch (error) {
      console.error('Bhashini ASR error:', error)
      throw new Error('Voice transcription failed')
    }
  }

  // Synthesize speech using Bhashini TTS
  async synthesizeSpeech(
    text: string,
    targetLanguage: string
  ): Promise<Buffer> {
    try {
      const response = await axios.post(
        `${this.baseURL}/tts`,
        {
          text,
          targetLanguage
        },
        {
          headers: {
            'Authorization': `Bearer ${this.apiKey}`,
            'Content-Type': 'application/json'
          },
          responseType: 'arraybuffer'
        }
      )

      return Buffer.from(response.data)
    } catch (error) {
      console.error('Bhashini TTS error:', error)
      throw new Error('Speech synthesis failed')
    }
  }

  // Create WebSocket connection for real-time streaming
  createStreamingConnection(
    sourceLanguage: string,
    onTranscription: (text: string) => void
  ): WebSocket {
    const ws = new WebSocket(`${this.baseURL}/stream`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    })

    ws.on('open', () => {
      ws.send(JSON.stringify({ sourceLanguage, targetLanguage: 'en' }))
    })

    ws.on('message', (data) => {
      const result = JSON.parse(data.toString())
      if (result.type === 'transcription') {
        onTranscription(result.text)
      }
    })

    return ws
  }
}
```

**Beckn Protocol Integration**:

```typescript
// services/beckn.service.ts
import axios from 'axios'
import { v4 as uuidv4 } from 'uuid'

interface BecknSearchIntent {
  context: {
    domain: string
    action: string
    transaction_id: string
    message_id: string
  }
  message: {
    intent: {
      category: { descriptor: { code: string } }
      fulfillment: {
        type: string
        start: {
          location: { gps: string; h3_index: string }
          time: { timestamp: string }
        }
        tags: Array<{ descriptor: { code: string }; list: any[] }>
      }
    }
  }
}

export class BecknService {
  private gatewayURL: string
  private bapId: string
  private bapURI: string

  constructor() {
    this.gatewayURL = process.env.BECKN_GATEWAY_URL!
    this.bapId = process.env.BECKN_BAP_ID!
    this.bapURI = process.env.BECKN_BAP_URI!
  }

  // Broadcast SOS as Beckn search intent
  async broadcastSOS(sosData: {
    location: [number, number]
    h3Index: string
    emergencyType: string
    severity: string
  }): Promise<string> {
    const transactionId = uuidv4()
    const messageId = uuidv4()

    const searchIntent: BecknSearchIntent = {
      context: {
        domain: 'emergency-response',
        action: 'search',
        transaction_id: transactionId,
        message_id: messageId
      },
      message: {
        intent: {
          category: {
            descriptor: { code: 'SOS' }
          },
          fulfillment: {
            type: 'EMERGENCY_RESPONSE',
            start: {
              location: {
                gps: `${sosData.location[0]},${sosData.location[1]}`,
                h3_index: sosData.h3Index
              },
              time: {
                timestamp: new Date().toISOString()
              }
            },
            tags: [
              {
                descriptor: { code: 'emergency_type' },
                list: [{ descriptor: { code: sosData.emergencyType }, value: 'true' }]
              },
              {
                descriptor: { code: 'severity' },
                list: [{ descriptor: { code: sosData.severity }, value: 'true' }]
              }
            ]
          }
        }
      }
    }

    try {
      await axios.post(`${this.gatewayURL}/search`, searchIntent, {
        headers: { 'Content-Type': 'application/json' }
      })

      return transactionId
    } catch (error) {
      console.error('Beckn broadcast error:', error)
      throw new Error('SOS broadcast failed')
    }
  }

  // Listen for responder responses (on_search)
  async getResponderResponses(transactionId: string): Promise<any[]> {
    try {
      const response = await axios.get(
        `${this.bapURI}/on_search?transaction_id=${transactionId}`
      )

      return response.data.responses || []
    } catch (error) {
      console.error('Beckn response retrieval error:', error)
      return []
    }
  }

  // Confirm responder selection
  async confirmResponder(
    transactionId: string,
    responderId: string
  ): Promise<void> {
    const messageId = uuidv4()

    const confirmIntent = {
      context: {
        domain: 'emergency-response',
        action: 'confirm',
        transaction_id: transactionId,
        message_id: messageId
      },
      message: {
        order: {
          provider: { id: responderId }
        }
      }
    }

    try {
      await axios.post(`${this.gatewayURL}/confirm`, confirmIntent)
    } catch (error) {
      console.error('Beckn confirm error:', error)
      throw new Error('Responder confirmation failed')
    }
  }
}
```


### 3. H3 Spatial Indexing Service

**Purpose**: Convert geographic coordinates to H3 hexagonal indices for safety bucketing.

```typescript
// services/h3.service.ts
import { cellToBoundary, latLngToCell, cellToLatLng, gridDisk } from 'h3-js'

export class H3Service {
  // Convert coordinates to H3 index
  coordinatesToH3(lat: number, lng: number, resolution: 9 | 10): string {
    return latLngToCell(lat, lng, resolution)
  }

  // Get hexagon boundary coordinates for map rendering
  getHexagonBoundary(h3Index: string): [number, number][] {
    return cellToBoundary(h3Index, true) as [number, number][]
  }

  // Get center coordinates of hexagon
  getHexagonCenter(h3Index: string): [number, number] {
    const [lat, lng] = cellToLatLng(h3Index)
    return [lat, lng]
  }

  // Get neighboring hexagons (k-ring)
  getNeighbors(h3Index: string, k: number = 1): string[] {
    return gridDisk(h3Index, k)
  }

  // Determine resolution based on area type
  getAppropriateResolution(populationDensity: number): 9 | 10 {
    // High density (>5000 people/km²) uses resolution 10 (66m edge)
    // Urban areas use resolution 9 (174m edge)
    return populationDensity > 5000 ? 10 : 9
  }

  // Batch convert multiple coordinates
  batchCoordinatesToH3(
    coordinates: [number, number][],
    resolution: 9 | 10
  ): string[] {
    return coordinates.map(([lat, lng]) => this.coordinatesToH3(lat, lng, resolution))
  }
}
```

### 4. IndicBERT NER Service

**Purpose**: Extract named entities (locations, times, emergency types) from vernacular text.

```typescript
// services/indicbert.service.ts
import axios from 'axios'

interface Entity {
  type: 'LOCATION' | 'PERSON' | 'TIME' | 'EMERGENCY_TYPE'
  value: string
  confidence: number
  startIndex: number
  endIndex: number
}

export class IndicBERTService {
  private apiURL: string
  private apiKey: string

  constructor() {
    this.apiURL = process.env.INDICBERT_API_URL!
    this.apiKey = process.env.INDICBERT_API_KEY!
  }

  // Extract entities from text
  async extractEntities(text: string, language: string): Promise<Entity[]> {
    try {
      const response = await axios.post(
        `${this.apiURL}/ner`,
        {
          text,
          language
        },
        {
          headers: {
            'Authorization': `Bearer ${this.apiKey}`,
            'Content-Type': 'application/json'
          }
        }
      )

      return response.data.entities.map((entity: any) => ({
        type: entity.label,
        value: entity.text,
        confidence: entity.score,
        startIndex: entity.start,
        endIndex: entity.end
      }))
    } catch (error) {
      console.error('IndicBERT NER error:', error)
      return []
    }
  }

  // Classify user intent from text and entities
  classifyIntent(text: string, entities: Entity[]): {
    type: 'SOS' | 'NAVIGATE' | 'QUERY_SAFETY' | 'REPORT_INCIDENT'
    confidence: number
  } {
    const lowerText = text.toLowerCase()

    // SOS keywords
    if (lowerText.includes('help') || lowerText.includes('emergency') || 
        lowerText.includes('मदद') || lowerText.includes('உதவி')) {
      return { type: 'SOS', confidence: 0.95 }
    }

    // Navigation keywords
    if (lowerText.includes('route') || lowerText.includes('navigate') ||
        lowerText.includes('रास्ता') || lowerText.includes('வழி')) {
      return { type: 'NAVIGATE', confidence: 0.90 }
    }

    // Safety query keywords
    if (lowerText.includes('safe') || lowerText.includes('safety') ||
        lowerText.includes('सुरक्षित') || lowerText.includes('பாதுகாப்பு')) {
      return { type: 'QUERY_SAFETY', confidence: 0.85 }
    }

    // Report incident keywords
    if (lowerText.includes('report') || lowerText.includes('incident') ||
        lowerText.includes('रिपोर्ट') || lowerText.includes('அறிக்கை')) {
      return { type: 'REPORT_INCIDENT', confidence: 0.80 }
    }

    // Default to query if uncertain
    return { type: 'QUERY_SAFETY', confidence: 0.50 }
  }

  // Resolve location entity to coordinates using geocoding
  async resolveLocation(
    locationEntity: string,
    userContext: { city?: string; state?: string }
  ): Promise<[number, number] | null> {
    // This would integrate with a geocoding service
    // For now, returning null to indicate unresolved
    try {
      const response = await axios.get(
        `https://nominatim.openstreetmap.org/search`,
        {
          params: {
            q: `${locationEntity}, ${userContext.city}, ${userContext.state}, India`,
            format: 'json',
            limit: 1
          }
        }
      )

      if (response.data.length > 0) {
        return [parseFloat(response.data[0].lat), parseFloat(response.data[0].lon)]
      }

      return null
    } catch (error) {
      console.error('Geocoding error:', error)
      return null
    }
  }
}
```

## Data Models

### MongoDB Schemas (Mongoose)

```typescript
// models/mongodb/User.model.ts
import mongoose, { Schema, Document } from 'mongoose'

export interface IUser extends Document {
  email: string
  passwordHash: string
  name: string
  phone: string
  languagePreference: string
  role: 'USER' | 'RESPONDER' | 'ADMIN'
  aadhaarVerified: boolean
  createdAt: Date
  lastActive: Date
}

const UserSchema = new Schema<IUser>({
  email: { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true },
  name: { type: String, required: true },
  phone: { type: String, required: true },
  languagePreference: { type: String, default: 'en' },
  role: { type: String, enum: ['USER', 'RESPONDER', 'ADMIN'], default: 'USER' },
  aadhaarVerified: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now },
  lastActive: { type: Date, default: Date.now }
})

export const User = mongoose.model<IUser>('User', UserSchema)
```

```typescript
// models/mongodb/Observation.model.ts
import mongoose, { Schema, Document } from 'mongoose'

export interface IObservation extends Document {
  userId: mongoose.Types.ObjectId
  location: {
    type: 'Point'
    coordinates: [number, number]
  }
  h3Index: string
  observationType: 'INCIDENT' | 'SAFETY_CONCERN' | 'SAFE_AREA' | 'INFRASTRUCTURE'
  safetyRating: number
  description: string
  photoUrls: string[]
  verificationStatus: 'PENDING' | 'VERIFIED' | 'REJECTED'
  contributionWeight: number
  createdAt: Date
}

const ObservationSchema = new Schema<IObservation>({
  userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  location: {
    type: { type: String, enum: ['Point'], required: true },
    coordinates: { type: [Number], required: true }
  },
  h3Index: { type: String, required: true, index: true },
  observationType: {
    type: String,
    enum: ['INCIDENT', 'SAFETY_CONCERN', 'SAFE_AREA', 'INFRASTRUCTURE'],
    required: true
  },
  safetyRating: { type: Number, min: 1, max: 5 },
  description: { type: String, required: true },
  photoUrls: [{ type: String }],
  verificationStatus: {
    type: String,
    enum: ['PENDING', 'VERIFIED', 'REJECTED'],
    default: 'PENDING'
  },
  contributionWeight: { type: Number, default: 1.0 },
  createdAt: { type: Date, default: Date.now }
})

// Geospatial index for location queries
ObservationSchema.index({ location: '2dsphere' })

export const Observation = mongoose.model<IObservation>('Observation', ObservationSchema)
```

### PostgreSQL Tables (PostGIS)

```sql
-- Safety hexagons with spatial data
CREATE TABLE safety_hexagons (
  h3_index VARCHAR(15) PRIMARY KEY,
  resolution SMALLINT NOT NULL CHECK (resolution IN (9, 10)),
  geometry GEOMETRY(POLYGON, 4326) NOT NULL,
  
  -- Safety factors (0-100 scale)
  crime_score NUMERIC(5,2) NOT NULL DEFAULT 50.0,
  streetlight_score NUMERIC(5,2) NOT NULL DEFAULT 50.0,
  crowd_score NUMERIC(5,2) NOT NULL DEFAULT 50.0,
  police_presence_score NUMERIC(5,2) NOT NULL DEFAULT 50.0,
  
  -- Aggregated score
  overall_safety_score NUMERIC(5,2) NOT NULL,
  confidence_level NUMERIC(3,2) NOT NULL DEFAULT 0.5,
  
  -- Metadata
  last_updated TIMESTAMP NOT NULL DEFAULT NOW(),
  update_count INTEGER NOT NULL DEFAULT 0,
  
  CONSTRAINT valid_scores CHECK (
    crime_score BETWEEN 0 AND 100 AND
    streetlight_score BETWEEN 0 AND 100 AND
    crowd_score BETWEEN 0 AND 100 AND
    police_presence_score BETWEEN 0 AND 100 AND
    overall_safety_score BETWEEN 0 AND 100
  )
);

CREATE INDEX idx_safety_hexagons_geom ON safety_hexagons USING GIST(geometry);
CREATE INDEX idx_safety_hexagons_score ON safety_hexagons(overall_safety_score);
```

```sql
-- SOS broadcasts
CREATE TABLE sos_broadcasts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id VARCHAR(24) NOT NULL, -- MongoDB ObjectId
  
  -- Location
  location GEOMETRY(POINT, 4326) NOT NULL,
  h3_index VARCHAR(15) NOT NULL,
  
  -- Emergency details
  emergency_type VARCHAR(50) NOT NULL,
  severity VARCHAR(20) NOT NULL,
  description TEXT,
  
  -- Beckn protocol tracking
  beckn_transaction_id VARCHAR(100) UNIQUE,
  beckn_state VARCHAR(20) NOT NULL DEFAULT 'search',
  
  -- Response tracking
  responder_id UUID,
  response_time_seconds INTEGER,
  resolution_status VARCHAR(20) DEFAULT 'PENDING',
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  responded_at TIMESTAMP,
  resolved_at TIMESTAMP,
  
  CONSTRAINT valid_beckn_state CHECK (
    beckn_state IN ('search', 'on_search', 'init', 'confirm', 'complete', 'cancelled')
  ),
  CONSTRAINT valid_severity CHECK (
    severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')
  )
);

CREATE INDEX idx_sos_location ON sos_broadcasts USING GIST(location);
CREATE INDEX idx_sos_h3 ON sos_broadcasts(h3_index);
CREATE INDEX idx_sos_created ON sos_broadcasts(created_at);
```

```sql
-- Responders
CREATE TABLE responders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id VARCHAR(24) NOT NULL UNIQUE, -- MongoDB ObjectId
  
  -- Responder details
  responder_type VARCHAR(30) NOT NULL,
  organization_name VARCHAR(200),
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  
  -- Capabilities
  capabilities JSONB NOT NULL DEFAULT '[]'::jsonb,
  service_radius_km NUMERIC(6,2) NOT NULL DEFAULT 5.0,
  
  -- Location
  base_location GEOMETRY(POINT, 4326),
  coverage_area GEOMETRY(POLYGON, 4326),
  
  -- Performance metrics
  average_response_time_minutes NUMERIC(6,2),
  total_responses INTEGER NOT NULL DEFAULT 0,
  successful_responses INTEGER NOT NULL DEFAULT 0,
  reputation_score NUMERIC(5,4) NOT NULL DEFAULT 0.5000,
  
  -- Metadata
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  last_active TIMESTAMP,
  
  CONSTRAINT valid_responder_type CHECK (
    responder_type IN ('VOLUNTEER', 'NGO', 'PRIVATE_SECURITY', 'GOVERNMENT')
  )
);

CREATE INDEX idx_responders_location ON responders USING GIST(base_location);
CREATE INDEX idx_responders_coverage ON responders USING GIST(coverage_area);
```

```sql
-- Reputation scores
CREATE TABLE reputation_scores (
  user_id VARCHAR(24) PRIMARY KEY, -- MongoDB ObjectId
  eigentrust_value NUMERIC(5,4) NOT NULL DEFAULT 0.5000,
  
  -- Historical accuracy tracking
  total_contributions INTEGER NOT NULL DEFAULT 0,
  verified_accurate INTEGER NOT NULL DEFAULT 0,
  verified_inaccurate INTEGER NOT NULL DEFAULT 0,
  
  -- Trust vector history (for EigenTrust iteration)
  trust_vector JSONB NOT NULL DEFAULT '[]'::jsonb,
  
  -- Metadata
  last_calculated TIMESTAMP NOT NULL DEFAULT NOW(),
  calculation_iterations INTEGER NOT NULL DEFAULT 0,
  
  CONSTRAINT valid_eigentrust CHECK (eigentrust_value BETWEEN 0 AND 1),
  CONSTRAINT valid_counts CHECK (
    total_contributions >= 0 AND
    verified_accurate >= 0 AND
    verified_inaccurate >= 0 AND
    verified_accurate + verified_inaccurate <= total_contributions
  )
);

CREATE INDEX idx_reputation_eigentrust ON reputation_scores(eigentrust_value);
```


## Algorithm Specifications

### 1. Analytic Hierarchy Process (AHP) for Route Safety Weighting

**Purpose**: Calculate mathematically consistent weights for safety factors.

```typescript
// services/ahp.service.ts
export class AHPService {
  private randomIndex = {
    3: 0.58,
    4: 0.90,
    5: 1.12,
    6: 1.24,
    7: 1.32
  }

  // Calculate AHP weights from pairwise comparison matrix
  calculateWeights(pairwiseMatrix: number[][]): {
    weights: number[]
    consistencyRatio: number
  } {
    const n = pairwiseMatrix.length

    // Step 1: Normalize matrix
    const normalizedMatrix = this.normalizeMatrix(pairwiseMatrix)

    // Step 2: Calculate priority vector (weights)
    const weights = normalizedMatrix.map(row =>
      row.reduce((sum, val) => sum + val, 0) / n
    )

    // Step 3: Calculate consistency
    const weightedSum = this.multiplyMatrixVector(pairwiseMatrix, weights)
    const lambdaMax = weightedSum.reduce((sum, val, i) => sum + val / weights[i], 0) / n
    const consistencyIndex = (lambdaMax - n) / (n - 1)
    const consistencyRatio = consistencyIndex / this.randomIndex[n as keyof typeof this.randomIndex]

    // Step 4: Validate consistency
    if (consistencyRatio > 0.1) {
      throw new Error(`Inconsistent pairwise comparisons (CR=${consistencyRatio.toFixed(3)})`)
    }

    return { weights, consistencyRatio }
  }

  private normalizeMatrix(matrix: number[][]): number[][] {
    const n = matrix.length
    const columnSums = Array(n).fill(0)

    // Calculate column sums
    for (let i = 0; i < n; i++) {
      for (let j = 0; j < n; j++) {
        columnSums[j] += matrix[i][j]
      }
    }

    // Normalize
    return matrix.map(row =>
      row.map((val, j) => val / columnSums[j])
    )
  }

  private multiplyMatrixVector(matrix: number[][], vector: number[]): number[] {
    return matrix.map(row =>
      row.reduce((sum, val, i) => sum + val * vector[i], 0)
    )
  }

  // Get default weights if AHP fails
  getDefaultWeights(): number[] {
    return [0.4, 0.3, 0.2, 0.1] // Crime, Streetlights, Crowd, Police
  }
}
```

### 2. EigenTrust Algorithm for Reputation Scoring

**Purpose**: Calculate transitive trust scores to prevent Sybil attacks.

```typescript
// services/eigentrust.service.ts
export class EigenTrustService {
  private readonly ALPHA = 0.15 // Pre-trust weight
  private readonly MAX_ITERATIONS = 50
  private readonly CONVERGENCE_THRESHOLD = 0.001

  // Calculate EigenTrust scores
  calculateTrust(
    trustMatrix: number[][],
    preTrustVector: number[]
  ): number[] {
    const n = trustMatrix.length
    let trustVector = this.normalize(preTrustVector)
    let iteration = 0

    while (iteration < this.MAX_ITERATIONS) {
      // Transitive trust propagation: t_new = (1-α) * C^T * t_old + α * p
      const transitiveTrust = this.multiplyMatrixVector(
        this.transpose(trustMatrix),
        trustVector
      )

      const newTrustVector = transitiveTrust.map((val, i) =>
        (1 - this.ALPHA) * val + this.ALPHA * preTrustVector[i]
      )

      // Check convergence
      const delta = this.l1Norm(this.subtract(newTrustVector, trustVector))
      if (delta < this.CONVERGENCE_THRESHOLD) {
        return newTrustVector
      }

      trustVector = newTrustVector
      iteration++
    }

    return trustVector
  }

  // Build trust matrix from user interactions
  buildTrustMatrix(
    userIds: string[],
    verifications: Array<{
      verifierId: string
      contributorId: string
      outcome: 'ACCURATE' | 'INACCURATE'
    }>
  ): number[][] {
    const n = userIds.length
    const userIndexMap = new Map(userIds.map((id, i) => [id, i]))
    const trustMatrix = Array(n).fill(0).map(() => Array(n).fill(0))

    // Build raw trust scores
    for (const verification of verifications) {
      const verifierIdx = userIndexMap.get(verification.verifierId)
      const contributorIdx = userIndexMap.get(verification.contributorId)

      if (verifierIdx !== undefined && contributorIdx !== undefined) {
        if (verification.outcome === 'ACCURATE') {
          trustMatrix[verifierIdx][contributorIdx] += 1.0
        } else {
          trustMatrix[verifierIdx][contributorIdx] -= 0.5
        }
      }
    }

    // Normalize rows to sum to 1
    for (let i = 0; i < n; i++) {
      const rowSum = trustMatrix[i].reduce((sum, val) => sum + Math.max(0, val), 0)
      if (rowSum > 0) {
        for (let j = 0; j < n; j++) {
          trustMatrix[i][j] = Math.max(0, trustMatrix[i][j]) / rowSum
        }
      } else {
        // No trust relationships; distribute uniformly
        for (let j = 0; j < n; j++) {
          trustMatrix[i][j] = 1.0 / n
        }
      }
    }

    return trustMatrix
  }

  private transpose(matrix: number[][]): number[][] {
    return matrix[0].map((_, i) => matrix.map(row => row[i]))
  }

  private multiplyMatrixVector(matrix: number[][], vector: number[]): number[] {
    return matrix.map(row =>
      row.reduce((sum, val, i) => sum + val * vector[i], 0)
    )
  }

  private subtract(a: number[], b: number[]): number[] {
    return a.map((val, i) => val - b[i])
  }

  private l1Norm(vector: number[]): number {
    return vector.reduce((sum, val) => sum + Math.abs(val), 0)
  }

  private normalize(vector: number[]): number[] {
    const sum = vector.reduce((s, v) => s + v, 0)
    return vector.map(v => v / sum)
  }
}
```

### 3. Safe Route Calculation with A* and Safety Weighting

**Purpose**: Find routes optimized for safety rather than just speed.

```typescript
// services/routing.service.ts
import { pgPool } from '../app'
import { H3Service } from './h3.service'

export class RoutingService {
  private h3Service: H3Service

  constructor() {
    this.h3Service = new H3Service()
  }

  // Calculate safe route using pgRouting with safety weights
  async calculateSafeRoute(
    origin: [number, number],
    destination: [number, number],
    safetyWeight: number = 0.7 // 0 = fast, 1 = safe
  ): Promise<{
    coordinates: [number, number][]
    distance: number
    estimatedTime: number
    averageSafetyScore: number
    minimumSafetyScore: number
  }> {
    // Find nearest road network nodes
    const originNode = await this.findNearestNode(origin)
    const destNode = await this.findNearestNode(destination)

    // Calculate route using pgRouting with custom cost function
    const query = `
      WITH route AS (
        SELECT * FROM pgr_dijkstra(
          'SELECT id, source, target, 
           cost * (1 + ${1 - safetyWeight} * time_factor + ${safetyWeight} * safety_factor) as cost,
           reverse_cost * (1 + ${1 - safetyWeight} * time_factor + ${safetyWeight} * safety_factor) as reverse_cost
           FROM road_network',
          $1, $2, directed := false
        )
      )
      SELECT 
        ST_AsGeoJSON(rn.geometry) as geometry,
        rn.length_meters as distance,
        rn.estimated_time_seconds as time,
        sh.overall_safety_score as safety_score
      FROM route r
      JOIN road_network rn ON r.edge = rn.id
      LEFT JOIN safety_hexagons sh ON ST_Intersects(rn.geometry, sh.geometry)
      ORDER BY r.seq
    `

    const result = await pgPool.query(query, [originNode, destNode])

    // Process results
    const coordinates: [number, number][] = []
    let totalDistance = 0
    let totalTime = 0
    const safetyScores: number[] = []

    for (const row of result.rows) {
      const geom = JSON.parse(row.geometry)
      coordinates.push(...geom.coordinates)
      totalDistance += row.distance
      totalTime += row.time
      if (row.safety_score) {
        safetyScores.push(row.safety_score)
      }
    }

    return {
      coordinates,
      distance: totalDistance,
      estimatedTime: totalTime,
      averageSafetyScore: safetyScores.reduce((a, b) => a + b, 0) / safetyScores.length,
      minimumSafetyScore: Math.min(...safetyScores)
    }
  }

  private async findNearestNode(location: [number, number]): Promise<number> {
    const query = `
      SELECT id
      FROM road_network_nodes
      ORDER BY geometry <-> ST_SetSRID(ST_MakePoint($1, $2), 4326)
      LIMIT 1
    `
    const result = await pgPool.query(query, [location[1], location[0]])
    return result.rows[0].id
  }

  // Calculate BOTH fastest and safest routes for comparison
  async calculateComparativeRoutes(
    origin: [number, number],
    destination: [number, number]
  ): Promise<{
    fastestRoute: {
      coordinates: [number, number][]
      distance: number
      estimatedTime: number
      averageSafetyScore: number
    }
    safestRoute: {
      coordinates: [number, number][]
      distance: number
      estimatedTime: number
      averageSafetyScore: number
    }
    safetyDifferential: {
      timeDifferenceMinutes: number
      safetyScoreImprovement: number
      percentageSafer: number
    }
  }> {
    // Calculate fastest route (safetyWeight = 0)
    const fastestRoute = await this.calculateSafeRoute(origin, destination, 0.0)

    // Calculate safest route (safetyWeight = 1)
    const safestRoute = await this.calculateSafeRoute(origin, destination, 1.0)

    // Calculate differential metrics
    const timeDifferenceMinutes = (safestRoute.estimatedTime - fastestRoute.estimatedTime) / 60
    const safetyScoreImprovement = safestRoute.averageSafetyScore - fastestRoute.averageSafetyScore
    const percentageSafer = (safetyScoreImprovement / fastestRoute.averageSafetyScore) * 100

    return {
      fastestRoute: {
        coordinates: fastestRoute.coordinates,
        distance: fastestRoute.distance,
        estimatedTime: fastestRoute.estimatedTime,
        averageSafetyScore: fastestRoute.averageSafetyScore
      },
      safestRoute: {
        coordinates: safestRoute.coordinates,
        distance: safestRoute.distance,
        estimatedTime: safestRoute.estimatedTime,
        averageSafetyScore: safestRoute.averageSafetyScore
      },
      safetyDifferential: {
        timeDifferenceMinutes,
        safetyScoreImprovement,
        percentageSafer
      }
    }
  }
}
```

### 4. Data Ingestion Engine (ETL Pipeline)

**Purpose**: Automate ingestion of official government data sources to supplement crowdsourced safety intelligence with authoritative crime statistics and infrastructure data.

**Data Sources**:
- **NCRB (National Crime Records Bureau)**: PDF reports with district-level crime statistics
- **Census API**: Population density, demographic data for H3 resolution assignment
- **OpenStreetMap Overpass API**: Infrastructure points (police stations, hospitals, petrol pumps)

```typescript
// services/ingestion.service.ts
import axios from 'axios'
import { parse as parsePDF } from 'pdf-parse'
import { H3Service } from './h3.service'
import { pgPool } from '../app'

export class DataIngestionService {
  private h3Service: H3Service

  constructor() {
    this.h3Service = new H3Service()
  }

  // Scrape NCRB PDF reports and extract crime statistics
  async ingestNCRBData(pdfUrl: string, year: number): Promise<void> {
    try {
      const response = await axios.get(pdfUrl, { responseType: 'arraybuffer' })
      const pdfBuffer = Buffer.from(response.data)
      const pdfData = await parsePDF(pdfBuffer)
      const crimeData = this.parseNCRBText(pdfData.text)

      for (const record of crimeData) {
        const h3Index = this.h3Service.coordinatesToH3(
          record.latitude,
          record.longitude,
          9
        )
        await this.updateCrimeScore(h3Index, record.crimeRate, year)
      }

      console.log(`Ingested ${crimeData.length} NCRB records for ${year}`)
    } catch (error) {
      console.error('NCRB ingestion error:', error)
      throw new Error('Failed to ingest NCRB data')
    }
  }

  // Fetch Census API data for population density
  async ingestCensusData(state: string, district: string): Promise<void> {
    try {
      const response = await axios.get(
        `https://api.census.gov.in/v1/data`,
        {
          params: { state, district, metrics: 'population_density,demographics' },
          headers: { 'Authorization': `Bearer ${process.env.CENSUS_API_KEY}` }
        }
      )

      const censusData = response.data.records
      for (const record of censusData) {
        const h3Index = this.h3Service.coordinatesToH3(
          record.centroid.lat,
          record.centroid.lng,
          this.h3Service.getAppropriateResolution(record.population_density)
        )
        await this.updatePopulationData(h3Index, record)
      }

      console.log(`Ingested Census data for ${district}, ${state}`)
    } catch (error) {
      console.error('Census ingestion error:', error)
      throw new Error('Failed to ingest Census data')
    }
  }

  // Ingest infrastructure data from OpenStreetMap Overpass API
  async ingestOSMInfrastructure(bbox: [number, number, number, number]): Promise<void> {
    const query = `
      [out:json];
      (
        node["amenity"="police"](${bbox[0]},${bbox[1]},${bbox[2]},${bbox[3]});
        node["amenity"="hospital"](${bbox[0]},${bbox[1]},${bbox[2]},${bbox[3]});
        node["amenity"="fuel"](${bbox[0]},${bbox[1]},${bbox[2]},${bbox[3]});
      );
      out body;
    `

    try {
      const response = await axios.post(
        'https://overpass-api.de/api/interpreter',
        query,
        { headers: { 'Content-Type': 'text/plain' } }
      )

      const elements = response.data.elements
      for (const element of elements) {
        await this.storeInfrastructure({
          type: element.tags.amenity,
          name: element.tags.name || 'Unnamed',
          location: [element.lat, element.lon],
          h3Index: this.h3Service.coordinatesToH3(element.lat, element.lon, 10)
        })
      }

      console.log(`Ingested ${elements.length} infrastructure points from OSM`)
    } catch (error) {
      console.error('OSM ingestion error:', error)
      throw new Error('Failed to ingest OSM infrastructure data')
    }
  }

  private parseNCRBText(text: string): Array<{
    district: string
    latitude: number
    longitude: number
    crimeRate: number
  }> {
    const records: any[] = []
    const lines = text.split('\n')
    for (const line of lines) {
      const match = line.match(/(\w+)\s+(\d+\.\d+)\s+(\d+\.\d+)\s+(\d+)/)
      if (match) {
        records.push({
          district: match[1],
          latitude: parseFloat(match[2]),
          longitude: parseFloat(match[3]),
          crimeRate: parseInt(match[4])
        })
      }
    }
    return records
  }

  private async updateCrimeScore(h3Index: string, crimeRate: number, year: number): Promise<void> {
    const crimeScore = Math.max(0, 100 - (crimeRate / 10))
    await pgPool.query(
      `INSERT INTO safety_hexagons (h3_index, crime_score, last_updated)
       VALUES ($1, $2, NOW())
       ON CONFLICT (h3_index) 
       DO UPDATE SET crime_score = $2, last_updated = NOW()`,
      [h3Index, crimeScore]
    )
  }

  private async updatePopulationData(h3Index: string, censusRecord: any): Promise<void> {
    await pgPool.query(
      `INSERT INTO safety_hexagons (h3_index, crowd_score, resolution)
       VALUES ($1, $2, $3)
       ON CONFLICT (h3_index)
       DO UPDATE SET crowd_score = $2`,
      [h3Index, Math.min(100, censusRecord.population_density / 100), censusRecord.population_density > 5000 ? 10 : 9]
    )
  }

  private async storeInfrastructure(data: {
    type: string
    name: string
    location: [number, number]
    h3Index: string
  }): Promise<void> {
    await pgPool.query(
      `INSERT INTO infrastructure (type, name, location, h3_index, created_at)
       VALUES ($1, $2, ST_SetSRID(ST_MakePoint($3, $4), 4326), $5, NOW())
       ON CONFLICT DO NOTHING`,
      [data.type, data.name, data.location[1], data.location[0], data.h3Index]
    )
  }
}
```

**PostgreSQL Table for Infrastructure**:

```sql
-- Infrastructure points (police stations, hospitals, petrol pumps)
CREATE TABLE infrastructure (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(20) NOT NULL CHECK (type IN ('police', 'hospital', 'fuel')),
  name VARCHAR(200) NOT NULL,
  location GEOMETRY(POINT, 4326) NOT NULL,
  h3_index VARCHAR(15) NOT NULL,
  address TEXT,
  phone VARCHAR(20),
  operating_hours VARCHAR(100),
  verified BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_infrastructure_location ON infrastructure USING GIST(location);
CREATE INDEX idx_infrastructure_type ON infrastructure(type);
CREATE INDEX idx_infrastructure_h3 ON infrastructure(h3_index);
```

**Scheduled ETL Jobs**:

```typescript
// jobs/etl.jobs.ts
import cron from 'node-cron'
import { DataIngestionService } from '../services/ingestion.service'

const ingestionService = new DataIngestionService()

// Run NCRB ingestion monthly
cron.schedule('0 0 1 * *', async () => {
  console.log('Starting monthly NCRB data ingestion...')
  const currentYear = new Date().getFullYear()
  await ingestionService.ingestNCRBData(
    `https://ncrb.gov.in/reports/${currentYear}/crime-data.pdf`,
    currentYear
  )
})

// Run Census ingestion quarterly
cron.schedule('0 0 1 */3 *', async () => {
  console.log('Starting quarterly Census data ingestion...')
  const states = ['Delhi', 'Maharashtra', 'Karnataka', 'Tamil Nadu']
  for (const state of states) {
    await ingestionService.ingestCensusData(state, 'all')
  }
})

// Run OSM infrastructure ingestion weekly
cron.schedule('0 0 * * 0', async () => {
  console.log('Starting weekly OSM infrastructure ingestion...')
  const majorCities = [
    [28.4, 77.0, 28.9, 77.4], // Delhi
    [18.9, 72.8, 19.3, 73.0], // Mumbai
    [12.8, 77.5, 13.1, 77.7]  // Bangalore
  ]
  for (const bbox of majorCities) {
    await ingestionService.ingestOSMInfrastructure(bbox as [number, number, number, number])
  }
})
```

### 5. Proactive Sentinel Service (Background Intelligence)

**Purpose**: Monitor user location and safety context to proactively trigger check-ins when users are stationary in high-risk areas.

**Logic**: `IF (User_Location == High_Risk_Hexagon) AND (Speed < 1km/h) AND (Time > 5 mins) THEN Trigger_Check_In`

```typescript
// services/sentinel.service.ts
import { H3Service } from './h3.service'
import { SafetyService } from './safety.service'
import { pgPool } from '../app'
import { io } from '../app'

interface UserLocationState {
  userId: string
  currentH3Index: string
  lastMovementTime: Date
  speed: number
  safetyScore: number
}

export class SentinelService {
  private h3Service: H3Service
  private safetyService: SafetyService
  private userStates: Map<string, UserLocationState>
  private readonly STATIONARY_THRESHOLD_KMH = 1.0
  private readonly TIME_THRESHOLD_MS = 5 * 60 * 1000
  private readonly HIGH_RISK_THRESHOLD = 40

  constructor() {
    this.h3Service = new H3Service()
    this.safetyService = new SafetyService()
    this.userStates = new Map()
    this.startMonitoring()
  }

  async updateUserLocation(
    userId: string,
    location: [number, number],
    speed: number
  ): Promise<void> {
    const h3Index = this.h3Service.coordinatesToH3(location[0], location[1], 9)
    const safetyScore = await this.safetyService.getSafetyScore(h3Index)
    const previousState = this.userStates.get(userId)
    const now = new Date()
    const hasMoved = previousState?.currentH3Index !== h3Index

    const newState: UserLocationState = {
      userId,
      currentH3Index: h3Index,
      lastMovementTime: hasMoved ? now : (previousState?.lastMovementTime || now),
      speed,
      safetyScore
    }

    this.userStates.set(userId, newState)
    await this.checkSentinelConditions(newState)
  }

  private async checkSentinelConditions(state: UserLocationState): Promise<void> {
    const isStationary = state.speed < this.STATIONARY_THRESHOLD_KMH
    const isHighRisk = state.safetyScore < this.HIGH_RISK_THRESHOLD
    const timeStationary = Date.now() - state.lastMovementTime.getTime()
    const exceededTimeThreshold = timeStationary > this.TIME_THRESHOLD_MS

    if (isStationary && isHighRisk && exceededTimeThreshold) {
      await this.triggerSafetyCheckIn(state)
    }
  }

  private async triggerSafetyCheckIn(state: UserLocationState): Promise<void> {
    console.log(`Sentinel triggered for user ${state.userId} in high-risk zone`)

    await pgPool.query(
      `INSERT INTO sentinel_events (user_id, h3_index, safety_score, triggered_at)
       VALUES ($1, $2, $3, NOW())`,
      [state.userId, state.currentH3Index, state.safetyScore]
    )

    io.to(state.userId).emit('sentinel:check-in', {
      message: 'You have been stationary in a high-risk area for 5 minutes. Are you safe?',
      safetyScore: state.safetyScore,
      location: state.currentH3Index,
      actions: ['I_AM_SAFE', 'NEED_HELP', 'FALSE_ALARM']
    })

    setTimeout(async () => {
      const response = await this.checkUserResponse(state.userId)
      if (!response) {
        await this.escalateToEmergency(state)
      }
    }, 60000)
  }

  private async checkUserResponse(userId: string): Promise<boolean> {
    const result = await pgPool.query(
      `SELECT response FROM sentinel_events 
       WHERE user_id = $1 
       ORDER BY triggered_at DESC 
       LIMIT 1`,
      [userId]
    )
    return result.rows[0]?.response !== null
  }

  private async escalateToEmergency(state: UserLocationState): Promise<void> {
    console.log(`Escalating to emergency for user ${state.userId}`)

    await pgPool.query(
      `INSERT INTO sos_broadcasts (
        user_id, location, h3_index, emergency_type, severity, 
        description, created_at
       )
       VALUES (
         $1, 
         ST_SetSRID(ST_MakePoint(0, 0), 4326), 
         $2, 
         'SAFETY_THREAT', 
         'HIGH',
         'Automatic escalation: User unresponsive in high-risk zone',
         NOW()
       )`,
      [state.userId, state.currentH3Index]
    )

    io.to(state.userId).emit('sentinel:escalated', {
      message: 'Emergency services have been notified',
      sosId: 'auto-generated'
    })
  }

  private startMonitoring(): void {
    setInterval(() => {
      for (const [userId, state] of this.userStates.entries()) {
        this.checkSentinelConditions(state)
      }
    }, 30000)
  }

  async handleCheckInResponse(
    userId: string,
    response: 'I_AM_SAFE' | 'NEED_HELP' | 'FALSE_ALARM'
  ): Promise<void> {
    await pgPool.query(
      `UPDATE sentinel_events 
       SET response = $1, responded_at = NOW()
       WHERE user_id = $2 
       AND triggered_at = (
         SELECT MAX(triggered_at) FROM sentinel_events WHERE user_id = $2
       )`,
      [response, userId]
    )

    if (response === 'NEED_HELP') {
      const state = this.userStates.get(userId)
      if (state) {
        await this.escalateToEmergency(state)
      }
    }
  }
}
```

**PostgreSQL Table for Sentinel Events**:

```sql
CREATE TABLE sentinel_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id VARCHAR(24) NOT NULL,
  h3_index VARCHAR(15) NOT NULL,
  safety_score NUMERIC(5,2) NOT NULL,
  triggered_at TIMESTAMP NOT NULL DEFAULT NOW(),
  response VARCHAR(20),
  responded_at TIMESTAMP,
  escalated BOOLEAN DEFAULT FALSE,
  speed_kmh NUMERIC(5,2),
  time_stationary_seconds INTEGER,
  CONSTRAINT valid_response CHECK (
    response IN ('I_AM_SAFE', 'NEED_HELP', 'FALSE_ALARM')
  )
);

CREATE INDEX idx_sentinel_user ON sentinel_events(user_id, triggered_at DESC);
CREATE INDEX idx_sentinel_h3 ON sentinel_events(h3_index);
```

### 6. Wearable Bridge (Hardware Integration)

**Purpose**: Integrate smartwatch biometric data (heart rate, accelerometer) for fall detection and physiological stress monitoring.

**Supported Devices**: Apple Watch, Wear OS, Fitbit

```typescript
// services/wearable.service.ts
import { io } from '../app'
import { pgPool } from '../app'

interface BiometricData {
  userId: string
  heartRate: number
  accelerometerX: number
  accelerometerY: number
  accelerometerZ: number
  timestamp: Date
}

interface FallDetectionResult {
  isFall: boolean
  confidence: number
  impactMagnitude: number
}

export class WearableService {
  private readonly FALL_THRESHOLD_G = 2.5
  private readonly ELEVATED_HR_THRESHOLD = 120
  private readonly STRESS_HR_THRESHOLD = 140

  async processBiometricData(data: BiometricData): Promise<void> {
    await this.storeBiometricData(data)

    const fallResult = this.detectFall(data)
    if (fallResult.isFall) {
      await this.handleFallDetected(data.userId, fallResult)
    }

    const isStressed = this.detectStress(data)
    if (isStressed) {
      await this.handleStressDetected(data.userId, data.heartRate)
    }
  }

  private detectFall(data: BiometricData): FallDetectionResult {
    const magnitude = Math.sqrt(
      data.accelerometerX ** 2 +
      data.accelerometerY ** 2 +
      data.accelerometerZ ** 2
    )
    const gForce = magnitude / 9.81
    const isFall = gForce > this.FALL_THRESHOLD_G
    const confidence = Math.min(1.0, gForce / (this.FALL_THRESHOLD_G * 2))

    return { isFall, confidence, impactMagnitude: gForce }
  }

  private detectStress(data: BiometricData): boolean {
    return data.heartRate > this.STRESS_HR_THRESHOLD
  }

  private async handleFallDetected(
    userId: string,
    fallResult: FallDetectionResult
  ): Promise<void> {
    console.log(`Fall detected for user ${userId} (confidence: ${fallResult.confidence})`)

    await pgPool.query(
      `INSERT INTO wearable_events (
        user_id, event_type, confidence, metadata, detected_at
       )
       VALUES ($1, 'FALL', $2, $3, NOW())`,
      [userId, fallResult.confidence, JSON.stringify({ impactMagnitude: fallResult.impactMagnitude })]
    )

    io.to(userId).emit('wearable:fall-detected', {
      message: 'Fall detected. Are you okay?',
      confidence: fallResult.confidence,
      actions: ['I_AM_OK', 'NEED_HELP'],
      autoEscalateIn: 30
    })

    setTimeout(async () => {
      const response = await this.checkFallResponse(userId)
      if (!response) {
        await this.escalateFallToSOS(userId)
      }
    }, 30000)
  }

  private async handleStressDetected(userId: string, heartRate: number): Promise<void> {
    console.log(`Elevated heart rate detected for user ${userId}: ${heartRate} BPM`)

    await pgPool.query(
      `INSERT INTO wearable_events (
        user_id, event_type, metadata, detected_at
       )
       VALUES ($1, 'STRESS', $2, NOW())`,
      [userId, JSON.stringify({ heartRate })]
    )

    io.to(userId).emit('wearable:stress-detected', {
      message: 'Elevated heart rate detected. Everything okay?',
      heartRate,
      suggestions: ['BREATHING_EXERCISE', 'CONTACT_FRIEND', 'IGNORE']
    })
  }

  private async checkFallResponse(userId: string): Promise<boolean> {
    const result = await pgPool.query(
      `SELECT response FROM wearable_events 
       WHERE user_id = $1 AND event_type = 'FALL'
       ORDER BY detected_at DESC 
       LIMIT 1`,
      [userId]
    )
    return result.rows[0]?.response !== null
  }

  private async escalateFallToSOS(userId: string): Promise<void> {
    console.log(`Escalating fall event to SOS for user ${userId}`)

    const locationResult = await pgPool.query(
      `SELECT location, h3_index FROM user_locations 
       WHERE user_id = $1 
       ORDER BY updated_at DESC 
       LIMIT 1`,
      [userId]
    )

    if (locationResult.rows.length > 0) {
      await pgPool.query(
        `INSERT INTO sos_broadcasts (
          user_id, location, h3_index, emergency_type, severity, 
          description, created_at
         )
         VALUES ($1, $2, $3, 'MEDICAL', 'CRITICAL', 
                 'Automatic: Fall detected, user unresponsive', NOW())`,
        [userId, locationResult.rows[0].location, locationResult.rows[0].h3_index]
      )
    }
  }

  private async storeBiometricData(data: BiometricData): Promise<void> {
    await pgPool.query(
      `INSERT INTO biometric_data (
        user_id, heart_rate, accelerometer_x, accelerometer_y, 
        accelerometer_z, recorded_at
       )
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [data.userId, data.heartRate, data.accelerometerX, data.accelerometerY, data.accelerometerZ, data.timestamp]
    )
  }
}
```

**PostgreSQL Tables for Wearable Data**:

```sql
CREATE TABLE biometric_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id VARCHAR(24) NOT NULL,
  heart_rate INTEGER NOT NULL,
  accelerometer_x NUMERIC(8,4),
  accelerometer_y NUMERIC(8,4),
  accelerometer_z NUMERIC(8,4),
  recorded_at TIMESTAMP NOT NULL,
  CONSTRAINT valid_heart_rate CHECK (heart_rate BETWEEN 30 AND 250)
);

CREATE INDEX idx_biometric_user ON biometric_data(user_id, recorded_at DESC);

CREATE TABLE wearable_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id VARCHAR(24) NOT NULL,
  event_type VARCHAR(20) NOT NULL CHECK (event_type IN ('FALL', 'STRESS')),
  confidence NUMERIC(3,2),
  metadata JSONB,
  response VARCHAR(20),
  detected_at TIMESTAMP NOT NULL DEFAULT NOW(),
  responded_at TIMESTAMP
);

CREATE INDEX idx_wearable_events_user ON wearable_events(user_id, detected_at DESC);
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: NER Accuracy Threshold
*For any* batch of translated texts containing known location entities and safety intents, the IndicBERT_Engine should extract at least 85% of the entities correctly.
**Validates: Requirements 1.2**

### Property 2: Language Consistency in Responses
*For any* voice command processed in a supported language, the audio feedback provided should be in the same language as the input.
**Validates: Requirements 1.4**

### Property 3: Offline Map Rendering
*For any* geographic location with cached PMTiles data, the system should render vector maps without internet connectivity.
**Validates: Requirements 2.2**

### Property 4: Last-Write-Wins Conflict Resolution
*For any* pair of conflicting offline edits, the system should preserve the edit with the later timestamp after synchronization.
**Validates: Requirements 2.3, 10.1**

### Property 5: Beckn Message Generation
*For any* SOS trigger event, the generated Beckn Protocol search intent should contain valid location coordinates, emergency type, and conform to Beckn Core Specification v1.0 schema.
**Validates: Requirements 3.1, 13.2**

### Property 6: Responder Broadcast Radius
*For any* SOS broadcast at a given location, all registered responders within a 5-kilometer radius should receive the search intent.
**Validates: Requirements 3.2, 8.2**

### Property 7: H3 Resolution Assignment
*For any* geographic coordinate, it should be mapped to Resolution 9 H3 hexagon in urban areas and Resolution 10 in high-density zones.
**Validates: Requirements 4.1, 4.2**

### Property 8: Safety Score Aggregation
*For any* Safety_Hexagon, the overall safety score should be calculated as a weighted combination of crime, streetlight, crowd, and police presence scores using AHP-derived weights.
**Validates: Requirements 4.3**

### Property 9: AHP Consistency Validation
*For any* pairwise comparison matrix, the consistency ratio should be computed, and matrices with CR > 0.1 should be rejected.
**Validates: Requirements 5.1, 5.2**

### Property 10: Reputation Score Calculation
*For any* user with submission history, their EigenTrust reputation score should be calculated based on historical accuracy.
**Validates: Requirements 6.1**

### Property 11: EigenTrust Convergence
*For any* trust matrix, the EigenTrust iteration should either converge (delta < 0.001) or terminate after exactly 50 iterations.
**Validates: Requirements 6.4**

### Property 12: Beckn Round-Trip Consistency
*For any* valid Beckn Protocol message object, serializing to JSON, parsing, then serializing again should produce an equivalent JSON structure.
**Validates: Requirements 13.4**

### Property 13: H3 Round-Trip Consistency
*For any* valid H3 hexagon identifier, serializing to string then deserializing should produce an equivalent H3 index representing the same geographic hexagon.
**Validates: Requirements 14.4**

(Note: Full 49 properties from prework are documented but abbreviated here for brevity. Implementation will reference all properties.)

## Error Handling

### Error Categories

**1. Network Errors**:
- Bhashini WebSocket disconnection → Fallback to Web Speech API
- Beckn Gateway unreachable → Queue SOS locally, retry with exponential backoff
- API timeout → Return cached data if available, show offline indicator

**2. Data Validation Errors**:
- Invalid H3 index → Reject with descriptive error message
- Malformed Beckn message → Return schema violation details
- Out-of-range safety scores → Clamp to valid range (0-100)

**3. Service Degradation**:
- IndicBERT unavailable → Use simple keyword-based entity extraction
- PostGIS connection lost → Fall back to MongoDB geospatial queries
- Redis cache miss → Query database directly

**4. Security Errors**:
- Unauthorized access → Return 401, log attempt, apply rate limiting
- Sybil attack detected → Reduce reputation scores, flag for review
- Location data leak attempt → Deny access, audit log

**5. User Input Errors**:
- Unintelligible voice → Ask user to repeat
- Ambiguous location → Present options for clarification
- Invalid route preferences → Use safe defaults

## Testing Strategy

### Dual Testing Approach

MargMitra employs comprehensive testing combining unit tests for specific scenarios and property-based tests for universal correctness. Both approaches are complementary and necessary.

**Unit Tests**: Validate specific examples, edge cases, integration points, and error conditions. Catch concrete bugs.

**Property Tests**: Verify universal properties across randomized inputs. Catch edge cases missed by examples.

### Property-Based Testing Configuration

**Libraries**:
- **TypeScript/JavaScript**: fast-check
- **Python** (if used for ML services): Hypothesis

**Configuration**:
- Minimum 100 iterations per property test
- Seed-based reproducibility for failed cases
- Shrinking enabled to find minimal failing examples
- Timeout: 30 seconds per property test

**Tagging Convention**:
```typescript
// Feature: marg-mitra-civic-neural-grid, Property 12: Beckn Round-Trip Consistency
test('Beckn message round-trip preserves structure', () => {
  fc.assert(
    fc.property(becknMessageArbitrary, (message) => {
      const json = serializeBeckn(message)
      const parsed = parseBeckn(json)
      const jsonAgain = serializeBeckn(parsed)
      expect(json).toEqual(jsonAgain)
    }),
    { numRuns: 100 }
  )
})
```

### Test Coverage by Component

**Frontend (React)**:
- Unit: Component rendering, Redux actions, service worker caching
- Integration: Map interaction, voice input flow, SOS broadcast
- E2E: Complete user journeys using Cypress

**Backend (Node.js)**:
- Unit: API endpoints, service methods, database queries
- Property: AHP consistency, EigenTrust convergence, H3 serialization
- Integration: Bhashini integration, Beckn workflow, database sync

**Database**:
- Unit: Schema validation, constraint enforcement
- Property: Geospatial queries, safety score calculations
- Performance: Query optimization, index effectiveness

### Integration Testing

**Critical Flows**:
1. Voice SOS → Bhashini → IndicBERT → Beckn → Responder
2. Offline map → Service worker → IndexedDB → Online sync
3. Crowdsource → Reputation → Safety score → Map update

### Performance Testing

**Targets**:
- API response time: <200ms p95
- Map rendering: <1s initial load
- Voice transcription: <300ms p95
- SOS broadcast: <500ms p95
- Database queries: <50ms p95

**Load Testing**:
- 10,000 concurrent users
- 100 SOS broadcasts per minute
- 1,000 safety score queries per second

## Deployment Architecture

### Docker Containers

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build: ./client
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
    depends_on:
      - backend

  backend:
    build: ./server
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongo:27017/margmitra
      - POSTGRES_URI=postgresql://postgres:password@postgres:5432/margmitra
      - REDIS_URI=redis://redis:6379
      - BHASHINI_API_KEY=${BHASHINI_API_KEY}
      - BECKN_GATEWAY_URL=${BECKN_GATEWAY_URL}
    depends_on:
      - mongo
      - postgres
      - redis

  mongo:
    image: mongo:6.0
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  postgres:
    image: postgis/postgis:14-3.3
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=margmitra
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7.0
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

volumes:
  mongo-data:
  postgres-data:
  redis-data:
```

## Conclusion

MargMitra's hybrid MERN + India Stack design combines modern web technologies with government-backed civic infrastructure. The architecture leverages React PWA for offline-capable frontend, Node.js + Express for scalable backend, MongoDB for flexible application data, PostgreSQL/PostGIS for spatial queries, and integrates Bhashini for multilingual voice, Beckn for decentralized coordination, and H3 for privacy-preserving spatial indexing.

Key innovations:
- Hybrid database strategy (MongoDB + PostgreSQL) for optimal data modeling
- Progressive enhancement with India Stack integration
- Voice-first vernacular interaction with fallback mechanisms
- Mathematically rigorous safety scoring (AHP) and reputation (EigenTrust)
- Comprehensive property-based testing for correctness guarantees

The system is designed for the Next Billion Users with offline-first capabilities, multilingual support, and privacy-preserving location tracking.
