# Requirements Document: MargMitra (The Civic Neural Grid)

## Introduction

MargMitra (The Civic Neural Grid) is a high-impact civic technology platform designed to unbundle "safety" as a public good using open protocols and India Stack principles. The platform focuses on three core pillars: Offline-First Sovereignty (ensuring functionality without constant internet connectivity), Linguistic Inclusion (enabling vernacular voice-first interactions), and Decentralized Safety (distributing emergency response across multiple stakeholders).

This is not merely an application but a node in the India Stack ecosystem that democratizes access to safety infrastructure for the Next Billion Users—particularly illiterate, rural, and low-bandwidth populations who are underserved by current safety solutions.

**Impact Vision**: Success is measured by "Safe Kilometers Traveled" and "Response Time Reduction" rather than traditional engagement metrics, emphasizing real-world safety outcomes over digital vanity metrics.

## Glossary

- **MargMitra_System**: The complete civic safety platform including mobile applications, backend services, and protocol integrations
- **Bhashini_Service**: Government of India's real-time speech translation API providing bi-directional voice translation via WebSocket
- **IndicBERT_Engine**: Named Entity Recognition (NER) model specialized for Indian languages to extract locations and intents from vernacular speech
- **Beckn_Gateway**: Decentralized discovery protocol gateway that broadcasts SOS signals as search intent packets
- **H3_Index**: Uber's hexagonal hierarchical spatial index system (Resolution 9/10) used for safety bucketing
- **PMTiles_Layer**: Serverless vector-based offline map format hosted on Cloudflare R2
- **AHP_Calculator**: Analytic Hierarchy Process algorithm for mathematical weighting of route safety factors
- **EigenTrust_Validator**: Peer-to-peer reputation algorithm preventing Sybil attacks in crowdsourced data
- **YAMNet_Detector**: TensorFlow Lite audio classification model running locally to detect acoustic danger signatures
- **WatermelonDB_Sync**: Local-first database with Last-Write-Wins conflict resolution for offline data synchronization
- **Safety_Hexagon**: H3-indexed geographic cell containing aggregated safety metrics
- **SOS_Responder**: Entity capable of responding to emergency broadcasts (volunteers, private security, NGOs)
- **Safe_Route**: Navigation path optimized for safety factors rather than speed or distance
- **Voice_Intent**: User command or query extracted from vernacular speech input
- **Reputation_Score**: EigenTrust-calculated trust value for crowdsourced data contributors

## User Personas

### Persona 1: Ramesh (Daily Wage Earner)
- **Demographics**: 35-year-old construction worker, illiterate, speaks Bhojpuri
- **Device**: Feature phone or entry-level Android (2GB RAM)
- **Connectivity**: Intermittent 2G/3G, often offline during commute
- **Primary Need**: Voice-first safety alerts without requiring literacy or constant internet
- **Key Scenario**: Walking home late at night through unfamiliar areas after work

### Persona 2: Priya (Night Commuter)
- **Demographics**: 28-year-old IT professional, urban resident, speaks English and Tamil
- **Device**: Mid-range smartphone with reliable 4G
- **Connectivity**: Generally connected but needs offline fallback
- **Primary Need**: Proactive safe routing that prioritizes well-lit, crowded paths over fastest routes
- **Key Scenario**: Commuting from office to home at 10 PM, wants real-time safety intelligence

## Requirements

### Requirement 1: Voice-First Vernacular Interaction

**User Story:** As Ramesh, I want to interact with the safety system using my native language through voice, so that I can access safety features without needing to read or type.

#### Acceptance Criteria

1. WHEN a user speaks in Hindi, Tamil, Bhojpuri, or other regional languages, THE Bhashini_Service SHALL translate the speech to English within 200 milliseconds
2. WHEN the translated text is received, THE IndicBERT_Engine SHALL extract location entities and safety intents with minimum 85% accuracy
3. WHEN internet connectivity is unavailable, THE YAMNet_Detector SHALL process voice locally as a fallback mechanism
4. WHEN a voice command is processed, THE MargMitra_System SHALL provide audio feedback in the user's original language
5. THE Bhashini_Service SHALL maintain bi-directional WebSocket connections for real-time voice streaming

### Requirement 2: Offline-First Map Access

**User Story:** As Ramesh, I want to access safety maps and navigation without internet connectivity, so that I can find safe routes even in areas with poor network coverage.

#### Acceptance Criteria

1. WHEN the application is installed, THE MargMitra_System SHALL download PMTiles_Layer data for the user's city to local storage
2. WHEN offline, THE MargMitra_System SHALL render vector maps using locally cached PMTiles_Layer data via MapLibre
3. WHEN map data is updated on the server, THE WatermelonDB_Sync SHALL synchronize changes using Last-Write-Wins conflict resolution when connectivity is restored
4. THE MargMitra_System SHALL cache Safety_Hexagon data for a 10-kilometer radius around the user's frequent locations
5. WHEN storage space is limited, THE MargMitra_System SHALL prioritize caching high-risk areas and frequently traveled routes

### Requirement 3: Decentralized Emergency Response Discovery

**User Story:** As Priya, I want my SOS signal to reach multiple types of responders simultaneously, so that I get the fastest possible help regardless of which organization can respond.

#### Acceptance Criteria

1. WHEN a user triggers an SOS, THE MargMitra_System SHALL broadcast a Beckn Protocol search intent packet containing location and emergency type
2. WHEN the search intent is broadcast, THE Beckn_Gateway SHALL deliver it to all registered SOS_Responder entities within a 5-kilometer radius
3. WHEN SOS_Responders receive the search intent, THE Beckn_Gateway SHALL collect on_search responses containing responder availability and estimated arrival time
4. WHEN multiple responses are received, THE MargMitra_System SHALL display all available responders ranked by proximity and reputation
5. WHEN a user selects a responder, THE MargMitra_System SHALL complete the Beckn workflow through init and confirm states

### Requirement 4: Hexagonal Safety Intelligence

**User Story:** As Priya, I want to see real-time safety scores for different areas, so that I can make informed decisions about which routes to take.

#### Acceptance Criteria

1. THE MargMitra_System SHALL divide geographic areas into H3_Index hexagons at Resolution 9 (174 meter edge length) for urban areas
2. THE MargMitra_System SHALL divide geographic areas into H3_Index hexagons at Resolution 10 (66 meter edge length) for high-density zones
3. WHEN calculating safety scores, THE MargMitra_System SHALL aggregate crime data, streetlight density, and crowd density within each Safety_Hexagon
4. WHEN displaying the map, THE MargMitra_System SHALL color-code Safety_Hexagons using a gradient from green (safe) to red (unsafe)
5. WHEN a Safety_Hexagon score is updated, THE MargMitra_System SHALL propagate the update to all users within 15 minutes when online

### Requirement 5: Algorithmic Safe Routing

**User Story:** As Priya, I want the system to suggest routes that prioritize my safety over speed, so that I can travel through well-lit and populated areas even if it takes longer.

#### Acceptance Criteria

1. WHEN calculating routes, THE AHP_Calculator SHALL weight safety factors (crime data, streetlight density, crowd density) using mathematically consistent pairwise comparison matrices
2. WHEN the AHP consistency ratio exceeds 0.1, THE AHP_Calculator SHALL reject the weighting matrix and use default safety weights
3. WHEN multiple route options exist, THE MargMitra_System SHALL present the safest route as the primary recommendation
4. WHEN a Safe_Route is calculated, THE MargMitra_System SHALL display the safety score alongside travel time and distance
5. THE MargMitra_System SHALL allow users to adjust the safety-versus-speed tradeoff using a slider interface

### Requirement 6: Crowdsourced Data Integrity

**User Story:** As a platform administrator, I want to prevent malicious actors from poisoning safety data, so that users can trust the safety scores and recommendations.

#### Acceptance Criteria

1. WHEN a user submits crowdsourced safety data, THE EigenTrust_Validator SHALL calculate a Reputation_Score based on historical accuracy
2. WHEN aggregating crowdsourced data, THE MargMitra_System SHALL weight contributions by the contributor's Reputation_Score
3. WHEN detecting potential Sybil attacks (multiple fake accounts), THE EigenTrust_Validator SHALL reduce the reputation of suspicious account clusters
4. THE EigenTrust_Validator SHALL iterate the trust matrix calculation until convergence (delta < 0.001) or maximum 50 iterations
5. WHEN a user's Reputation_Score falls below 0.3, THE MargMitra_System SHALL flag their contributions for manual review

### Requirement 7: Acoustic Danger Detection

**User Story:** As Ramesh, I want the system to automatically detect danger sounds around me, so that I can be alerted to potential threats even when I'm not actively using the app.

#### Acceptance Criteria

1. WHEN the application is running in background mode, THE YAMNet_Detector SHALL continuously analyze ambient audio using the device microphone
2. WHEN acoustic signatures matching screaming, crashes, or gunshots are detected, THE YAMNet_Detector SHALL trigger an automatic alert within 500 milliseconds
3. THE YAMNet_Detector SHALL run as a TensorFlow Lite int8 quantized model to minimize battery consumption
4. WHEN a danger sound is detected, THE MargMitra_System SHALL prompt the user to confirm the emergency before broadcasting an SOS
5. THE YAMNet_Detector SHALL process all audio locally without transmitting raw audio data to servers

### Requirement 8: Multi-Stakeholder Response Ecosystem

**User Story:** As a volunteer responder, I want to receive nearby SOS alerts, so that I can provide immediate assistance to people in my community.

#### Acceptance Criteria

1. WHEN a volunteer registers as an SOS_Responder, THE MargMitra_System SHALL verify their identity and background through government ID validation
2. WHEN an SOS is broadcast, THE Beckn_Gateway SHALL notify all SOS_Responders within the relevant geographic radius
3. WHEN an SOS_Responder accepts an alert, THE MargMitra_System SHALL provide real-time navigation to the incident location
4. WHEN an incident is resolved, THE MargMitra_System SHALL collect feedback from both the requester and responder to update Reputation_Scores
5. THE MargMitra_System SHALL support multiple responder types: individual volunteers, private security firms, NGOs, and government agencies

### Requirement 9: Linguistic Named Entity Recognition

**User Story:** As Ramesh, I want to mention landmarks and locations in my local language, so that the system understands where I am or where I need to go.

#### Acceptance Criteria

1. WHEN processing vernacular speech, THE IndicBERT_Engine SHALL extract location entities including street names, landmarks, and neighborhoods
2. WHEN a location entity is extracted, THE MargMitra_System SHALL resolve it to geographic coordinates using local geocoding databases
3. WHEN multiple location interpretations exist, THE MargMitra_System SHALL prompt the user to clarify using voice
4. THE IndicBERT_Engine SHALL support Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi, and Odia
5. WHEN a location entity cannot be resolved, THE MargMitra_System SHALL ask the user to provide additional context in their native language

### Requirement 10: Data Synchronization and Conflict Resolution

**User Story:** As a developer, I want offline data changes to sync reliably when connectivity is restored, so that users have a consistent experience across devices and network conditions.

#### Acceptance Criteria

1. WHEN multiple offline edits conflict, THE WatermelonDB_Sync SHALL resolve conflicts using Last-Write-Wins strategy based on device timestamps
2. WHEN syncing Safety_Hexagon updates, THE MargMitra_System SHALL merge server-side and client-side changes without data loss
3. WHEN a sync operation fails, THE WatermelonDB_Sync SHALL retry with exponential backoff up to 5 attempts
4. THE WatermelonDB_Sync SHALL maintain a local change log for audit and debugging purposes
5. WHEN sync is complete, THE MargMitra_System SHALL notify the user of any significant safety score changes in their area

### Requirement 11: Low-Bandwidth Optimization

**User Story:** As Ramesh, I want the application to work smoothly on my 2G connection, so that I can access safety features even with slow internet.

#### Acceptance Criteria

1. WHEN transmitting data over the network, THE MargMitra_System SHALL compress payloads using gzip or brotli compression
2. WHEN on 2G/3G networks, THE MargMitra_System SHALL reduce map tile quality and limit real-time updates to critical safety information
3. THE MargMitra_System SHALL prioritize loading Safety_Hexagon data over decorative map elements
4. WHEN bandwidth is severely constrained, THE MargMitra_System SHALL switch to text-only mode for SOS communication
5. THE MargMitra_System SHALL cache API responses aggressively to minimize redundant network requests

### Requirement 12: Privacy-Preserving Location Tracking

**User Story:** As Priya, I want my location data to be protected, so that my movements cannot be tracked or misused by unauthorized parties.

#### Acceptance Criteria

1. WHEN storing location history, THE MargMitra_System SHALL anonymize data by storing only H3_Index hexagon identifiers rather than precise GPS coordinates
2. WHEN transmitting location data, THE MargMitra_System SHALL use end-to-end encryption
3. THE MargMitra_System SHALL allow users to delete their location history at any time
4. WHEN calculating safety scores, THE MargMitra_System SHALL aggregate data across multiple users to prevent individual identification
5. THE MargMitra_System SHALL not share raw location data with third-party SOS_Responders until an SOS is confirmed by the user

## Impact Metrics

### Primary Success Metrics

1. **Safe Kilometers Traveled**: Total distance traveled by users through routes recommended by the Safe_Route algorithm
2. **Response Time Reduction**: Percentage decrease in emergency response time compared to traditional 911/100 systems
3. **Vernacular Adoption Rate**: Percentage of users primarily interacting via voice in non-English languages
4. **Offline Usage Ratio**: Percentage of app sessions that occur partially or fully offline

### Secondary Success Metrics

1. **Responder Network Growth**: Number of verified SOS_Responders across different stakeholder categories
2. **Safety Score Accuracy**: Correlation between predicted safety scores and actual incident reports
3. **Crowdsource Contribution Rate**: Number of users actively contributing safety observations
4. **Battery Efficiency**: Average battery consumption per hour of background acoustic monitoring

## Special Requirements Guidance

### Parser and Serializer Requirements

#### Requirement 13: Beckn Protocol Message Parsing

**User Story:** As a developer, I want to parse and generate Beckn Protocol messages correctly, so that the system can interoperate with the decentralized responder network.

##### Acceptance Criteria

1. WHEN receiving Beckn Protocol messages, THE MargMitra_System SHALL parse JSON payloads conforming to the Beckn Core Specification v1.0
2. WHEN generating Beckn search intents, THE MargMitra_System SHALL serialize SOS data into valid Beckn Protocol JSON format
3. THE MargMitra_System SHALL implement a Beckn_Pretty_Printer to format protocol messages for debugging and logging
4. FOR ALL valid Beckn Protocol messages, parsing then printing then parsing SHALL produce an equivalent message object (round-trip property)
5. WHEN a malformed Beckn message is received, THE MargMitra_System SHALL return a descriptive error indicating the schema violation

#### Requirement 14: H3 Index Serialization

**User Story:** As a developer, I want to serialize and deserialize H3 hexagon identifiers efficiently, so that spatial data can be stored and transmitted compactly.

##### Acceptance Criteria

1. WHEN storing H3_Index values, THE MargMitra_System SHALL serialize them as 64-bit unsigned integers or 15-character hexadecimal strings
2. WHEN deserializing H3_Index values, THE MargMitra_System SHALL validate that the resolution matches expected values (9 or 10)
3. THE MargMitra_System SHALL implement an H3_Pretty_Printer to convert H3 indices to human-readable geographic descriptions
4. FOR ALL valid H3_Index values, serializing then deserializing SHALL produce an equivalent hexagon identifier (round-trip property)
5. WHEN an invalid H3 index is encountered, THE MargMitra_System SHALL reject it with a clear error message

## Conclusion

MargMitra represents a paradigm shift in civic safety infrastructure, moving from centralized, English-only, always-online systems to a decentralized, multilingual, offline-first approach. By leveraging India Stack protocols and edge AI, the platform democratizes access to safety for the Next Billion Users who have been historically underserved by traditional safety solutions.
