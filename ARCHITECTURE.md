# Out of Box - Technical Architecture

## System Overview

Out of Box is built as a native iOS application leveraging Apple's ecosystem for maximum privacy, performance, and user experience. The architecture follows a client-centric design with minimal server dependencies.

## Architecture Principles

### Privacy by Design
- All user data processing happens on-device
- Machine learning models run locally using Core ML
- Minimal data transmission to cloud services
- User controls all data sharing decisions

### Apple Ecosystem Integration
- SwiftUI for modern, declarative UI
- SwiftData for local persistence and Core Data evolution
- CloudKit for cloud sync and shared challenge database
- Core ML for on-device intelligence
- UserNotifications for challenge delivery

### Scalable Simplicity
- Start simple, add intelligence progressively
- Modular architecture supports feature phases
- Clean separation between data, business logic, and presentation

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    iOS Application                      │
├─────────────────────────────────────────────────────────┤
│  Presentation Layer (SwiftUI)                          │
│  ├─ Challenge Views                                     │
│  ├─ Photo Capture & Gallery                            │
│  ├─ Settings & Onboarding                              │
│  └─ Social Sharing                                     │
├─────────────────────────────────────────────────────────┤
│  Business Logic Layer                                  │
│  ├─ Challenge Engine                                   │
│  ├─ Notification Manager                               │
│  ├─ Photo Manager                                      │
│  ├─ ML Personalization Engine (Phase 2)               │
│  └─ Sharing Coordinator                                │
├─────────────────────────────────────────────────────────┤
│  Data Layer                                            │
│  ├─ SwiftData Models                                   │
│  ├─ CloudKit Sync                                      │
│  ├─ Local Storage                                      │
│  └─ Core ML Models                                     │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                  Cloud Services                        │
├─────────────────────────────────────────────────────────┤
│  CloudKit Container                                     │
│  ├─ Challenge Database                                 │
│  ├─ User Sync Data                                     │
│  └─ App Configuration                                  │
└─────────────────────────────────────────────────────────┘
```

## Component Architecture

### 1. Presentation Layer (SwiftUI)

#### Views Structure
```
Views/
├── Root/
│   ├── ContentView.swift
│   ├── TabNavigationView.swift
│   └── AppDelegate.swift
├── Challenge/
│   ├── ChallengeCardView.swift
│   ├── ChallengeDetailView.swift
│   ├── ChallengeCompletionView.swift
│   └── ChallengeHistoryView.swift
├── Photo/
│   ├── PhotoCaptureView.swift
│   ├── PhotoEditView.swift
│   └── PhotoGalleryView.swift
├── Settings/
│   ├── SettingsView.swift
│   ├── OnboardingView.swift
│   ├── NotificationSettingsView.swift
│   └── PrivacySettingsView.swift
└── Shared/
    ├── Components/
    └── Extensions/
```

#### View Models (ObservableObject)
- `ChallengeViewModel`: Manages challenge state and user interactions
- `PhotoViewModel`: Handles photo capture, editing, and storage
- `SettingsViewModel`: User preferences and app configuration
- `NotificationViewModel`: Notification scheduling and permissions

### 2. Business Logic Layer

#### Core Services
```swift
// Challenge Engine
protocol ChallengeEngineProtocol {
    func getTodaysChallenge() async -> Challenge?
    func completeChallenge(_ challenge: Challenge, photo: UIImage?)
    func skipChallenge(_ challenge: Challenge)
    func getChallengeHistory() -> [CompletedChallenge]
}

// Notification Manager
protocol NotificationManagerProtocol {
    func scheduleRandomNotification(in timeWindow: TimeWindow)
    func requestPermissions() async -> Bool
    func cancelAllNotifications()
}

// ML Personalization Engine (Phase 2)
protocol PersonalizationEngineProtocol {
    func updateUserPreferences(from interaction: ChallengeInteraction)
    func getPersonalizedChallenges() async -> [Challenge]
    func getOptimalNotificationTime() -> Date?
}
```

#### Data Flow
1. **Challenge Delivery**: NotificationManager → ChallengeEngine → ChallengeViewModel → UI
2. **Challenge Completion**: UI → ChallengeViewModel → ChallengeEngine → DataStore
3. **Personalization**: ChallengeEngine → MLEngine → UserPreferences → ChallengeSelection

### 3. Data Layer

#### SwiftData Models
```swift
@Model
class Challenge {
    var id: String
    var title: String
    var description: String
    var category: ChallengeCategory
    var timeInvestment: TimeInvestment
    var socialRequirement: SocialRequirement
    var costLevel: CostLevel
    var difficultyLevel: Int
    var tags: [String]
    var isActive: Bool
    var createdAt: Date
    var updatedAt: Date
}

@Model
class UserChallenge {
    var id: String
    var challenge: Challenge
    var status: ChallengeStatus // pending, completed, skipped
    var assignedDate: Date
    var completedDate: Date?
    var photoPath: String?
    var userRating: Int?
    var completionTime: TimeInterval?
}

@Model
class UserPreferences {
    var id: String
    var timeWindow: TimeWindow
    var preferredCategories: [ChallengeCategory]
    var maxTimeInvestment: TimeInvestment
    var budgetLevel: CostLevel
    var socialComfortLevel: Int
    var notificationsEnabled: Bool
    var lastPersonalizationUpdate: Date
}
```

#### CloudKit Schema
```
ChallengeRecord (Public Database)
├── challengeID: String
├── title: String
├── description: String
├── category: String
├── timeInvestment: Int
├── socialRequirement: Int
├── costLevel: Int
├── difficultyLevel: Int
├── tags: [String]
├── isActive: Bool
├── minAppVersion: String
└── contentHash: String

UserSyncRecord (Private Database)
├── userID: String
├── preferences: Data (encoded UserPreferences)
├── completedChallenges: [String] (challenge IDs)
├── lastSyncDate: Date
└── deviceIdentifier: String
```

## Phase-Specific Architecture

### Phase 1: Foundation (MVP)
- Basic challenge delivery using random selection
- Local photo storage with simple gallery
- External social media sharing
- CloudKit for challenge content distribution

### Phase 2: Intelligence
- Core ML integration for personalization
- Enhanced data collection and analysis
- Sophisticated challenge recommendation engine
- Advanced notification timing optimization

### Phase 3: Community
- Extended CloudKit schema for social features
- Photo sharing within app community
- Friend connections and discovery
- Enhanced privacy controls

## Security Architecture

### Data Protection
```
User Data Security Layers:
├── Device Level
│   ├── Keychain for sensitive data
│   ├── Face/Touch ID for app access (optional)
│   └── iOS Secure Enclave integration
├── Application Level
│   ├── Data encryption at rest
│   ├── Certificate pinning for API calls
│   └── Local data validation
└── Cloud Level
    ├── CloudKit encryption in transit
    ├── Private database access control
    └── Minimal data transmission
```

### Privacy Implementation
- All ML processing happens on-device using Core ML
- User photos stored locally with optional iCloud backup
- Minimal telemetry (only anonymous usage patterns)
- Granular privacy controls in settings

## Performance Architecture

### Optimization Strategies
- **Lazy Loading**: Challenges loaded on-demand
- **Image Compression**: Smart photo compression for storage
- **Background Processing**: ML inference during idle time
- **Caching Strategy**: 3-challenge lookahead for offline use
- **Memory Management**: Efficient SwiftData queries

### Monitoring
- Performance metrics collection (anonymous)
- Crash reporting through Xcode Organizer
- Energy usage optimization
- Network usage minimization

## Development Architecture

### Project Structure
```
OutOfBox/
├── App/
│   ├── OutOfBoxApp.swift
│   ├── AppDelegate.swift
│   └── SceneDelegate.swift
├── Core/
│   ├── Models/
│   ├── Services/
│   ├── Utilities/
│   └── Extensions/
├── Features/
│   ├── Challenge/
│   ├── Photo/
│   ├── Settings/
│   └── Social/
├── UI/
│   ├── Views/
│   ├── ViewModels/
│   ├── Components/
│   └── Styles/
├── Resources/
│   ├── Assets.xcassets
│   ├── Localizable.strings
│   └── Config/
└── Tests/
    ├── UnitTests/
    ├── IntegrationTests/
    └── UITests/
```

### Dependency Management
- Swift Package Manager for third-party dependencies
- Minimal external dependencies (privacy focused)
- Apple frameworks preference over third-party solutions

### Build Configuration
- Debug, Beta, Release configurations
- Feature flags for phase rollouts
- Environment-specific CloudKit containers
- Automated testing integration

## Deployment Architecture

### Distribution Strategy
- App Store as primary distribution channel
- TestFlight for beta testing
- Staged rollout for major updates
- Emergency update capability

### CloudKit Setup
- Development, Staging, Production containers
- Automatic scaling for user growth
- Geographic distribution for performance
- Backup and disaster recovery

## Future Architecture Considerations

### Scalability
- Microservices consideration for complex features
- CDN for challenge content distribution
- Advanced analytics platform integration
- Multi-platform expansion (iPad, Mac, Watch)

### Technology Evolution
- SwiftUI evolution adoption
- New iOS features integration
- Machine learning model improvements
- Performance optimization opportunities