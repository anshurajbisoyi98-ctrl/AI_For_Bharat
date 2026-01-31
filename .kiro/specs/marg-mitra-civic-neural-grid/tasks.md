# Implementation Plan: MargMitra (The Civic Neural Grid)

## Overview

This implementation plan breaks down the MargMitra civic safety platform into discrete, manageable tasks. The platform uses MERN stack (MongoDB, Express, React, Node.js) integrated with India Stack protocols (Bhashini, Beckn, H3). Tasks are organized to build incrementally, with testing integrated throughout.

## Tasks

- [ ] 1. Project Setup and Infrastructure
  - Initialize monorepo with client (React) and server (Node.js) directories
  - Configure TypeScript for both frontend and backend
  - Set up Docker Compose with MongoDB, PostgreSQL/PostGIS, and Redis
  - Configure environment variables and secrets management
  - Set up ESLint, Prettier, and Git hooks
  - _Requirements: All (foundational)_

- [ ] 2. Database Schema and Models
  - [ ] 2.1 Create MongoDB schemas using Mongoose
    - Implement User model with authentication fields
    - Implement Observation model with geospatial indexing
    - Implement Session model for JWT token management
    - _Requirements: 12.1, 12.3_
  
  - [ ] 2.2 Create PostgreSQL tables with PostGIS
    - Create safety_hexagons table with H3 index and geometry
    - Create sos_broadcasts table with Beckn tracking fields
    - Create responders table with coverage area geometry
    - Create reputation_scores table for EigenTrust values
    - Add spatial indexes (GIST) for geometry columns
    - _Requirements: 4.1, 4.2, 4.3, 6.1, 8.1_
  
  - [ ]* 2.3 Write property test for H3 serialization
    - **Property 13: H3 Round-Trip Consistency**
    - **Validates: Requirements 14.4**
  
  - [ ] 2.4 Create database connection utilities
    - Implement MongoDB connection with Mongoose
    - Implement PostgreSQL connection pool with pg library
    - Implement Redis client for caching
    - Add connection health checks and retry logic
    - _Requirements: 10.3_

- [ ] 3. Authentication and Authorization
  - [ ] 3.1 Implement JWT-based authentication
    - Create user registration endpoint with bcrypt password hashing
    - Create login endpoint with JWT token generation
    - Create token refresh endpoint
    - Implement auth middleware for protected routes
    - _Requirements: None (infrastructure)_
  
  - [ ]* 3.2 Write unit tests for authentication
    - Test registration with valid/invalid inputs
    - Test login with correct/incorrect credentials
    - Test JWT token validation and expiration
    - Test auth middleware authorization logic
    - _Requirements: None (infrastructure)_

- [ ] 4. H3 Spatial Indexing Service
  - [ ] 4.1 Implement H3Service class
    - Implement coordinatesToH3 method for converting lat/lng to H3 index
    - Implement getHexagonBoundary for map rendering
    - Implement getNeighbors for k-ring queries
    - Implement getAppropriateResolution based on population density
    - _Requirements: 4.1, 4.2_
  
  - [ ]* 4.2 Write property test for H3 resolution assignment
    - **Property 7: H3 Resolution Assignment**
    - **Validates: Requirements 4.1, 4.2**
  
  - [ ]* 4.3 Write property test for H3 serialization format
    - **Property 13: H3 Round-Trip Consistency**
    - **Validates: Requirements 14.1, 14.4**

- [ ] 5. Bhashini Voice Integration
  - [ ] 5.1 Implement BhashiniService class
    - Implement transcribeAudio method for ASR
    - Implement synthesizeSpeech method for TTS
    - Implement createStreamingConnection for WebSocket
    - Add error handling and fallback mechanisms
    - _Requirements: 1.1, 1.4, 1.5_
  
  - [ ] 5.2 Create voice processing API endpoints
    - POST /api/voice/transcribe for audio transcription
    - POST /api/voice/synthesize for text-to-speech
    - Add audio format validation and conversion
    - _Requirements: 1.1, 1.4_
  
  - [ ]* 5.3 Write property test for language consistency
    - **Property 2: Language Consistency in Responses**
    - **Validates: Requirements 1.4**
  
  - [ ]* 5.4 Write unit tests for Bhashini integration
    - Test transcription with sample audio files
    - Test TTS synthesis for multiple languages
    - Test WebSocket connection handling
    - Test fallback to Web Speech API when offline
    - _Requirements: 1.3, 1.4_

- [ ] 6. IndicBERT NER Service
  - [ ] 6.1 Implement IndicBERTService class
    - Implement extractEntities method for NER
    - Implement classifyIntent method for intent detection
    - Implement resolveLocation method for geocoding
    - Add confidence threshold handling
    - _Requirements: 1.2, 9.1, 9.2_
  
  - [ ]* 6.2 Write property test for NER accuracy
    - **Property 1: NER Accuracy Threshold**
    - **Validates: Requirements 1.2**
  
  - [ ]* 6.3 Write property test for entity extraction
    - **Property: Location Entity Extraction**
    - **Validates: Requirements 9.1**
  
  - [ ]* 6.4 Write unit tests for intent classification
    - Test SOS intent detection with keywords
    - Test navigation intent detection
    - Test safety query intent detection
    - Test ambiguous input handling
    - _Requirements: 9.3, 9.5_

- [ ] 7. Checkpoint - Core Services Functional
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Safety Intelligence Module
  - [ ] 8.1 Implement SafetyService class
    - Implement getSafetyScore method querying PostgreSQL
    - Implement updateSafetyScore method with factor aggregation
    - Implement getSafetyHexagonsInRadius for map queries
    - Add caching layer using Redis
    - _Requirements: 4.3, 4.4, 4.5_
  
  - [ ] 8.2 Create safety data API endpoints
    - GET /api/safety/hexagons for fetching safety data
    - GET /api/safety/hexagons/:h3Index for specific hexagon
    - POST /api/safety/observations for crowdsourced reports
    - _Requirements: 4.3, 4.4_
  
  - [ ]* 8.3 Write property test for safety score aggregation
    - **Property 8: Safety Score Aggregation**
    - **Validates: Requirements 4.3**
  
  - [ ]* 8.4 Write unit tests for safety calculations
    - Test safety score calculation with known factor values
    - Test hexagon color mapping for visualization
    - Test caching behavior for repeated queries
    - _Requirements: 4.4, 11.5_

- [ ] 9. AHP Route Safety Weighting
  - [ ] 9.1 Implement AHPService class
    - Implement calculateWeights method with matrix normalization
    - Implement consistency ratio calculation
    - Implement getDefaultWeights fallback
    - Add matrix validation logic
    - _Requirements: 5.1, 5.2_
  
  - [ ]* 9.2 Write property test for AHP consistency
    - **Property 9: AHP Consistency Validation**
    - **Validates: Requirements 5.1, 5.2**
  
  - [ ]* 9.3 Write unit tests for AHP calculations
    - Test weight calculation with valid matrix
    - Test rejection of inconsistent matrix (CR > 0.1)
    - Test default weights fallback
    - _Requirements: 5.2_

- [ ] 10. EigenTrust Reputation System
  - [ ] 10.1 Implement EigenTrustService class
    - Implement calculateTrust method with matrix iteration
    - Implement buildTrustMatrix from user verifications
    - Add convergence detection logic
    - Implement Sybil attack detection heuristics
    - _Requirements: 6.1, 6.2, 6.3, 6.4_
  
  - [ ]* 10.2 Write property test for EigenTrust convergence
    - **Property 11: EigenTrust Convergence**
    - **Validates: Requirements 6.4**
  
  - [ ]* 10.3 Write property test for reputation calculation
    - **Property 10: Reputation Score Calculation**
    - **Validates: Requirements 6.1**
  
  - [ ]* 10.4 Write unit tests for reputation system
    - Test trust matrix construction from verifications
    - Test reputation score updates
    - Test low reputation flagging (< 0.3)
    - Test Sybil attack detection
    - _Requirements: 6.3, 6.5_

- [ ] 11. Safe Route Calculation
  - [ ] 11.1 Implement RoutingService class
    - Implement calculateSafeRoute using pgRouting
    - Implement findNearestNode for road network snapping
    - Add safety-speed tradeoff parameter handling
    - Calculate route metrics (distance, time, safety scores)
    - _Requirements: 5.3, 5.4, 5.5_
  
  - [ ] 11.2 Create routing API endpoint
    - GET /api/safety/route with origin, destination, preferences
    - Return route coordinates, metrics, and safety scores
    - Add route caching for common queries
    - _Requirements: 5.3, 5.4_
  
  - [ ]* 11.3 Write property test for route safety prioritization
    - **Property: Safest Route Prioritization**
    - **Validates: Requirements 5.3**
  
  - [ ]* 11.4 Write unit tests for routing
    - Test route calculation with various safety weights
    - Test route metrics completeness
    - Test nearest node finding
    - _Requirements: 5.4, 5.5_

- [ ] 12. Checkpoint - Intelligence Layer Complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Beckn Protocol Integration
  - [ ] 13.1 Implement BecknService class
    - Implement broadcastSOS method creating search intent
    - Implement getResponderResponses for on_search collection
    - Implement confirmResponder for workflow completion
    - Add Beckn message serialization and parsing
    - _Requirements: 3.1, 3.2, 3.3, 3.5, 13.1, 13.2_
  
  - [ ]* 13.2 Write property test for Beckn message generation
    - **Property 5: Beckn Message Generation**
    - **Validates: Requirements 3.1, 13.2**
  
  - [ ]* 13.3 Write property test for Beckn round-trip
    - **Property 12: Beckn Round-Trip Consistency**
    - **Validates: Requirements 13.4**
  
  - [ ]* 13.4 Write unit tests for Beckn integration
    - Test SOS broadcast message format
    - Test responder response parsing
    - Test workflow state transitions
    - Test error handling for malformed messages
    - _Requirements: 3.5, 13.5_

- [ ] 14. SOS Emergency Broadcast System
  - [ ] 14.1 Implement SOS API endpoints
    - POST /api/sos/broadcast for triggering emergency
    - GET /api/sos/:id for SOS status
    - PUT /api/sos/:id/respond for responder acceptance
    - PUT /api/sos/:id/resolve for incident resolution
    - _Requirements: 3.1, 3.4, 8.3, 8.4_
  
  - [ ] 14.2 Implement Socket.IO real-time updates
    - Emit SOS broadcasts to nearby responders
    - Emit responder responses to SOS requester
    - Emit status updates for ongoing incidents
    - _Requirements: 3.2, 3.3_
  
  - [ ]* 14.3 Write property test for responder broadcast radius
    - **Property 6: Responder Broadcast Radius**
    - **Validates: Requirements 3.2, 8.2**
  
  - [ ]* 14.4 Write unit tests for SOS system
    - Test SOS creation and storage
    - Test responder notification logic
    - Test response collection and ranking
    - Test incident resolution workflow
    - _Requirements: 3.4, 8.4_

- [ ] 15. Responder Management
  - [ ] 15.1 Implement responder API endpoints
    - POST /api/responders/register for responder signup
    - GET /api/responders/nearby for proximity queries
    - PUT /api/responders/:id/availability for status updates
    - Add responder verification workflow
    - _Requirements: 8.1, 8.5_
  
  - [ ]* 15.2 Write unit tests for responder management
    - Test responder registration and verification
    - Test nearby responder queries with PostGIS
    - Test availability status updates
    - Test multiple responder type support
    - _Requirements: 8.1, 8.5_

- [ ] 16. Crowdsourced Observations
  - [ ] 16.1 Implement observation submission
    - POST /api/safety/observations endpoint
    - Store observations in MongoDB with geospatial index
    - Link observations to H3 hexagons
    - Calculate contribution weight based on reputation
    - _Requirements: 6.2, 10.4_
  
  - [ ]* 16.2 Write property test for reputation-weighted aggregation
    - **Property: Reputation-Weighted Aggregation**
    - **Validates: Requirements 6.2**
  
  - [ ]* 16.3 Write unit tests for observations
    - Test observation creation and validation
    - Test H3 hexagon linking
    - Test reputation-based weighting
    - Test observation verification workflow
    - _Requirements: 6.2, 10.4_

- [ ] 17. Checkpoint - Backend API Complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 18. React Frontend Setup
  - [ ] 18.1 Initialize React app with TypeScript
    - Set up Create React App or Vite with TypeScript
    - Configure Redux Toolkit for state management
    - Set up React Router for navigation
    - Configure Axios for API calls
    - _Requirements: None (infrastructure)_
  
  - [ ] 18.2 Implement authentication UI
    - Create Login component
    - Create Register component
    - Implement auth slice in Redux
    - Add protected route wrapper
    - _Requirements: None (infrastructure)_
  
  - [ ]* 18.3 Write unit tests for auth components
    - Test login form validation
    - Test registration form validation
    - Test auth state management
    - Test protected route behavior
    - _Requirements: None (infrastructure)_

- [ ] 19. Map Visualization with Leaflet
  - [ ] 19.1 Implement MapView component
    - Integrate Leaflet.js for map rendering
    - Add user location tracking with Geolocation API
    - Implement map controls (zoom, pan, center)
    - Add loading states and error handling
    - _Requirements: 2.2_
  
  - [ ] 19.2 Implement SafetyOverlay component
    - Fetch safety hexagons from API
    - Render H3 hexagons as polygons on map
    - Apply color gradient based on safety scores
    - Add hexagon click handlers for details
    - _Requirements: 4.4_
  
  - [ ] 19.3 Implement RouteDisplay component
    - Display calculated safe routes on map
    - Show route metrics (distance, time, safety)
    - Add route comparison UI for multiple options
    - _Requirements: 5.3, 5.4_
  
  - [ ]* 19.4 Write unit tests for map components
    - Test map initialization and rendering
    - Test hexagon overlay rendering
    - Test route display and metrics
    - _Requirements: 2.2, 4.4, 5.4_

- [ ] 20. Voice Interaction UI
  - [ ] 20.1 Implement VoiceInput component
    - Add microphone button with recording indicator
    - Integrate Web Speech API for browser-based ASR
    - Send audio to Bhashini API for transcription
    - Display transcribed text and translations
    - _Requirements: 1.1, 1.4_
  
  - [ ] 20.2 Implement LanguageSelector component
    - Create dropdown for language selection
    - Support Hindi, Tamil, English, and 7+ Indian languages
    - Persist language preference in localStorage
    - _Requirements: 1.4, 9.4_
  
  - [ ]* 20.3 Write unit tests for voice components
    - Test microphone permission handling
    - Test audio recording and upload
    - Test transcription display
    - Test language selection persistence
    - _Requirements: 1.1, 1.4_

- [ ] 21. SOS Emergency UI
  - [ ] 21.1 Implement SOSButton component
    - Create prominent emergency button
    - Add confirmation dialog to prevent accidental triggers
    - Capture current location and emergency type
    - Trigger SOS broadcast via API
    - _Requirements: 3.1_
  
  - [ ] 21.2 Implement ResponderList component
    - Display available responders from Beckn responses
    - Show responder details (type, ETA, reputation)
    - Implement responder selection and confirmation
    - Add real-time status updates via Socket.IO
    - _Requirements: 3.3, 3.4, 3.5_
  
  - [ ]* 21.3 Write unit tests for SOS components
    - Test SOS button trigger and confirmation
    - Test responder list rendering and sorting
    - Test responder selection flow
    - Test Socket.IO event handling
    - _Requirements: 3.1, 3.4, 3.5_

- [ ] 22. Safety Observation UI
  - [ ] 22.1 Implement ObservationForm component
    - Create form for safety reports
    - Add location picker (map or current location)
    - Add observation type and rating selectors
    - Add photo upload capability
    - Submit observations to API
    - _Requirements: 10.4_
  
  - [ ]* 22.2 Write unit tests for observation form
    - Test form validation
    - Test location selection
    - Test photo upload
    - Test submission handling
    - _Requirements: 10.4_

- [ ] 23. Progressive Web App (PWA) Setup
  - [ ] 23.1 Configure service worker
    - Implement cache-first strategy for static assets
    - Implement network-first strategy for API calls
    - Add offline fallback page
    - Configure cache expiration policies
    - _Requirements: 2.1, 2.2_
  
  - [ ] 23.2 Implement IndexedDB caching
    - Cache safety hexagon data locally
    - Cache user preferences and session data
    - Implement sync queue for offline operations
    - Add cache management utilities
    - _Requirements: 2.2, 2.4_
  
  - [ ]* 23.3 Write property test for offline map rendering
    - **Property 3: Offline Map Rendering**
    - **Validates: Requirements 2.2**
  
  - [ ]* 23.4 Write unit tests for PWA features
    - Test service worker caching strategies
    - Test IndexedDB operations
    - Test offline queue management
    - Test online/offline state detection
    - _Requirements: 2.2, 2.3_

- [ ] 24. Data Synchronization
  - [ ] 24.1 Implement sync service
    - Detect online/offline state changes
    - Sync queued operations when online
    - Implement Last-Write-Wins conflict resolution
    - Add exponential backoff for failed syncs
    - _Requirements: 2.3, 10.1, 10.2, 10.3_
  
  - [ ]* 24.2 Write property test for conflict resolution
    - **Property 4: Last-Write-Wins Conflict Resolution**
    - **Validates: Requirements 2.3, 10.1**
  
  - [ ]* 24.3 Write unit tests for sync service
    - Test sync queue management
    - Test conflict resolution logic
    - Test retry with exponential backoff
    - Test sync completion notifications
    - _Requirements: 10.2, 10.3, 10.5_

- [ ] 25. Checkpoint - Frontend Complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 26. Integration Testing
  - [ ] 26.1 Write end-to-end tests with Cypress
    - Test complete SOS flow (trigger → broadcast → response → resolution)
    - Test voice interaction flow (record → transcribe → extract entities → action)
    - Test safe route calculation flow (select origin/destination → calculate → display)
    - Test offline-to-online transition with queued operations
    - _Requirements: Multiple (integration)_
  
  - [ ]* 26.2 Write integration tests for API workflows
    - Test Bhashini → IndicBERT → Intent classification pipeline
    - Test SOS → Beckn → Responder coordination flow
    - Test Observation → Reputation → Safety score update pipeline
    - _Requirements: Multiple (integration)_

- [ ] 27. Performance Optimization
  - [ ] 27.1 Optimize database queries
    - Add database indexes for frequently queried fields
    - Optimize PostGIS spatial queries
    - Implement query result caching with Redis
    - Add database connection pooling
    - _Requirements: 11.5_
  
  - [ ] 27.2 Optimize frontend performance
    - Implement code splitting and lazy loading
    - Optimize map rendering with virtualization
    - Add image optimization and lazy loading
    - Implement debouncing for search inputs
    - _Requirements: 11.2, 11.3_
  
  - [ ]* 27.3 Run performance tests
    - Test API response times under load
    - Test map rendering performance
    - Test concurrent user handling
    - Test database query performance
    - _Requirements: None (performance)_

- [ ] 28. Security Hardening
  - [ ] 28.1 Implement security measures
    - Add rate limiting to API endpoints
    - Implement CORS configuration
    - Add input validation and sanitization
    - Implement HTTPS enforcement
    - Add security headers with Helmet.js
    - _Requirements: 12.2_
  
  - [ ] 28.2 Implement privacy features
    - Add location data anonymization with H3
    - Implement end-to-end encryption for sensitive data
    - Add user data deletion capability
    - Implement aggregation privacy for safety scores
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5_
  
  - [ ]* 28.3 Write security tests
    - Test rate limiting enforcement
    - Test input validation and SQL injection prevention
    - Test authentication and authorization
    - Test location data anonymization
    - _Requirements: 12.1, 12.2, 12.4, 12.5_

- [ ] 29. Documentation and Deployment
  - [ ] 29.1 Write API documentation
    - Document all API endpoints with OpenAPI/Swagger
    - Add request/response examples
    - Document authentication requirements
    - Add error code reference
    - _Requirements: None (documentation)_
  
  - [ ] 29.2 Create deployment configuration
    - Configure Docker Compose for production
    - Set up environment variable management
    - Configure logging and monitoring
    - Add health check endpoints
    - _Requirements: None (deployment)_
  
  - [ ] 29.3 Write deployment guide
    - Document deployment steps
    - Add configuration instructions
    - Document backup and recovery procedures
    - Add troubleshooting guide
    - _Requirements: None (documentation)_

- [ ] 30. Final Checkpoint - System Complete
  - Ensure all tests pass, ask the user if questions arise.
  - Verify all requirements are implemented
  - Run full test suite (unit, property, integration, E2E)
  - Perform final security audit
  - Validate performance benchmarks

## Notes

- Tasks marked with `*` are optional test tasks that can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at major milestones
- Property tests validate universal correctness properties
- Unit tests validate specific examples and edge cases
- Integration tests validate end-to-end workflows
- The implementation follows a bottom-up approach: infrastructure → services → API → frontend
- Testing is integrated throughout rather than deferred to the end
