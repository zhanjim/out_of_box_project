# Out of Box - Database Design

## Overview

Out of Box uses a hybrid data architecture combining local SwiftData storage for user data and CloudKit for shared content distribution. This design ensures privacy (local processing) while enabling content updates and cross-device sync.

## Data Architecture Strategy

### Local Storage (SwiftData)
- **Purpose**: User-specific data, challenge history, preferences, photos
- **Benefits**: Privacy, offline access, fast queries, on-device ML
- **Scope**: Personal data, completion history, ML training data

### Cloud Storage (CloudKit)
- **Purpose**: Challenge content, app configuration, cross-device sync
- **Benefits**: Content distribution, sync, scalability
- **Scope**: Shared challenge database, user preferences backup

## SwiftData Models (Local Storage)

### Core Models

#### Challenge Entity
```swift
@Model
class Challenge {
    @Attribute(.unique) var id: String
    var title: String
    var description: String
    var category: ChallengeCategory
    var timeInvestment: TimeInvestment
    var socialRequirement: SocialRequirement
    var costLevel: CostLevel
    var difficultyLevel: Int
    var tags: [String]
    var locationType: LocationType
    var seasonality: Seasonality?
    var isActive: Bool
    var contentHash: String
    var createdAt: Date
    var updatedAt: Date
    
    // Relationships
    @Relationship(deleteRule: .cascade) var userChallenges: [UserChallenge] = []
    
    init(id: String, title: String, description: String, 
         category: ChallengeCategory, timeInvestment: TimeInvestment,
         socialRequirement: SocialRequirement, costLevel: CostLevel,
         difficultyLevel: Int, tags: [String] = []) {
        self.id = id
        self.title = title
        self.description = description
        self.category = category
        self.timeInvestment = timeInvestment
        self.socialRequirement = socialRequirement
        self.costLevel = costLevel
        self.difficultyLevel = difficultyLevel
        self.tags = tags
        self.locationType = .anywhere
        self.isActive = true
        self.contentHash = ""
        self.createdAt = Date()
        self.updatedAt = Date()
    }
}
```

#### UserChallenge Entity
```swift
@Model
class UserChallenge {
    @Attribute(.unique) var id: String
    var challengeID: String
    var status: ChallengeStatus
    var assignedDate: Date
    var completedDate: Date?
    var skippedDate: Date?
    var photoPath: String?
    var userRating: Int? // 1-5 scale
    var completionTime: TimeInterval? // seconds
    var userNotes: String?
    var notificationSentAt: Date?
    var sharedToSocialMedia: Bool
    
    // ML Training Data
    var timeOfDay: Int // hour of completion (0-23)
    var dayOfWeek: Int // 1-7
    var weatherContext: String? // sunny, rainy, etc.
    var moodBefore: Int? // 1-5 scale
    var moodAfter: Int? // 1-5 scale
    
    // Relationships
    var challenge: Challenge?
    
    init(challengeID: String, assignedDate: Date = Date()) {
        self.id = UUID().uuidString
        self.challengeID = challengeID
        self.status = .pending
        self.assignedDate = assignedDate
        self.sharedToSocialMedia = false
    }
}
```

#### UserPreferences Entity
```swift
@Model
class UserPreferences {
    @Attribute(.unique) var id: String
    
    // Notification Settings
    var notificationsEnabled: Bool
    var timeWindowStart: Date // Time of day
    var timeWindowEnd: Date // Time of day
    var timeZone: String
    
    // Challenge Preferences
    var preferredCategories: [ChallengeCategory]
    var maxTimeInvestment: TimeInvestment
    var budgetLevel: CostLevel
    var socialComfortLevel: Int // 1-5 scale
    var difficultyPreference: Int // 1-5 scale
    var locationPreferences: [LocationType]
    
    // Personalization Data
    var personalizationEnabled: Bool
    var lastPersonalizationUpdate: Date
    var modelVersion: String
    var completedChallengesCount: Int
    var totalDaysActive: Int
    
    // App Settings
    var onboardingCompleted: Bool
    var premiumSubscribed: Bool
    var premiumTrialUsed: Bool
    var darkModeEnabled: Bool?
    var analyticsEnabled: Bool
    
    init() {
        self.id = "user_preferences"
        self.notificationsEnabled = true
        self.timeWindowStart = Calendar.current.date(from: DateComponents(hour: 9)) ?? Date()
        self.timeWindowEnd = Calendar.current.date(from: DateComponents(hour: 20)) ?? Date()
        self.timeZone = TimeZone.current.identifier
        self.preferredCategories = []
        self.maxTimeInvestment = .medium
        self.budgetLevel = .small
        self.socialComfortLevel = 3
        self.difficultyPreference = 3
        self.locationPreferences = [.anywhere]
        self.personalizationEnabled = false
        self.lastPersonalizationUpdate = Date()
        self.modelVersion = "1.0"
        self.completedChallengesCount = 0
        self.totalDaysActive = 0
        self.onboardingCompleted = false
        self.premiumSubscribed = false
        self.premiumTrialUsed = false
        self.analyticsEnabled = true
    }
}
```

#### Photo Entity
```swift
@Model
class Photo {
    @Attribute(.unique) var id: String
    var userChallengeID: String
    var filename: String
    var originalFilename: String?
    var fileSize: Int64
    var width: Int
    var height: Int
    var createdAt: Date
    var editedAt: Date?
    var isEdited: Bool
    var compressionRatio: Double
    
    // Metadata
    var location: String? // City, landmark, etc.
    var weather: String?
    var deviceModel: String
    
    // Sharing
    var sharedToSocialPlatforms: [String] // ["instagram", "twitter"]
    var sharedAt: Date?
    
    init(userChallengeID: String, filename: String, width: Int, height: Int, fileSize: Int64) {
        self.id = UUID().uuidString
        self.userChallengeID = userChallengeID
        self.filename = filename
        self.width = width
        self.height = height
        self.fileSize = fileSize
        self.createdAt = Date()
        self.isEdited = false
        self.compressionRatio = 1.0
        self.deviceModel = UIDevice.current.model
        self.sharedToSocialPlatforms = []
    }
}
```

#### MLTrainingData Entity (Phase 2)
```swift
@Model
class MLTrainingData {
    @Attribute(.unique) var id: String
    var userChallengeID: String
    var featureVector: Data // Encoded feature vector
    var outcome: MLOutcome // completed, skipped, rated_high, etc.
    var contextData: Data // Encoded context (time, weather, mood, etc.)
    var modelVersion: String
    var createdAt: Date
    
    init(userChallengeID: String, featureVector: Data, outcome: MLOutcome, contextData: Data) {
        self.id = UUID().uuidString
        self.userChallengeID = userChallengeID
        self.featureVector = featureVector
        self.outcome = outcome
        self.contextData = contextData
        self.modelVersion = "1.0"
        self.createdAt = Date()
    }
}
```

### Enumerations

```swift
enum ChallengeCategory: String, CaseIterable, Codable {
    case physical = "physical"
    case creative = "creative"
    case social = "social"
    case intellectual = "intellectual"
    case emotional = "emotional"
    case adventure = "adventure"
    case learning = "learning"
    case mindfulness = "mindfulness"
}

enum TimeInvestment: Int, CaseIterable, Codable {
    case quick = 15      // 5-15 minutes
    case medium = 45     // 30-60 minutes
    case extended = 120  // 2+ hours
}

enum SocialRequirement: String, CaseIterable, Codable {
    case solo = "solo"
    case strangers = "strangers"
    case friends = "friends"
    case family = "family"
    case group = "group"
}

enum CostLevel: String, CaseIterable, Codable {
    case free = "free"
    case small = "small"    // $1-10
    case medium = "medium"  // $10-50
    case large = "large"    // $50+
}

enum LocationType: String, CaseIterable, Codable {
    case home = "home"
    case outdoor = "outdoor"
    case public = "public"
    case specific = "specific"
    case anywhere = "anywhere"
}

enum ChallengeStatus: String, CaseIterable, Codable {
    case pending = "pending"
    case completed = "completed"
    case skipped = "skipped"
    case expired = "expired"
}

enum Seasonality: String, CaseIterable, Codable {
    case spring = "spring"
    case summer = "summer"
    case fall = "fall"
    case winter = "winter"
    case holiday = "holiday"
}

enum MLOutcome: String, CaseIterable, Codable {
    case completed = "completed"
    case skipped = "skipped"
    case ratedHigh = "rated_high"
    case ratedLow = "rated_low"
    case sharedSocial = "shared_social"
    case repeatedCategory = "repeated_category"
}
```

## CloudKit Schema (Cloud Storage)

### Public Database (Challenge Content)

#### ChallengeRecord
```
ChallengeRecord (CKRecord)
├── challengeID: String (unique)
├── title: String
├── description: String
├── category: String
├── timeInvestment: Int64
├── socialRequirement: String
├── costLevel: String
├── difficultyLevel: Int64
├── tags: [String]
├── locationType: String
├── seasonality: String?
├── isActive: Bool
├── minAppVersion: String
├── contentHash: String
├── createdAt: Date
├── updatedAt: Date
└── validUntil: Date?
```

#### AppConfiguration
```
AppConfigRecord (CKRecord)
├── configKey: String (unique)
├── configValue: String
├── dataType: String
├── isActive: Bool
├── minAppVersion: String
├── updatedAt: Date
└── description: String
```

### Private Database (User Sync)

#### UserSyncRecord
```
UserSyncRecord (CKRecord)
├── userID: String (unique)
├── preferences: Data (encoded UserPreferences)
├── completedChallengeIDs: [String]
├── skippedChallengeIDs: [String]
├── lastSyncDate: Date
├── deviceIdentifier: String
├── appVersion: String
└── syncVersion: Int64
```

## Database Operations

### Local Operations (SwiftData)

#### Challenge Management
```swift
// Fetch today's challenge
func getTodaysChallenge() -> UserChallenge? {
    let today = Calendar.current.startOfDay(for: Date())
    let predicate = #Predicate<UserChallenge> { challenge in
        challenge.assignedDate >= today && 
        challenge.assignedDate < Calendar.current.date(byAdding: .day, value: 1, to: today)!
    }
    return try? modelContext.fetch(FetchDescriptor(predicate: predicate)).first
}

// Complete challenge
func completeChallenge(_ userChallenge: UserChallenge, photo: UIImage? = nil) {
    userChallenge.status = .completed
    userChallenge.completedDate = Date()
    
    if let photo = photo {
        let photoEntity = savePhoto(photo, for: userChallenge)
        userChallenge.photoPath = photoEntity.filename
    }
    
    updateMLTrainingData(for: userChallenge)
    try? modelContext.save()
}

// Get challenge history
func getChallengeHistory(limit: Int = 50) -> [UserChallenge] {
    let predicate = #Predicate<UserChallenge> { $0.status == .completed }
    let descriptor = FetchDescriptor(
        predicate: predicate,
        sortBy: [SortDescriptor(\.completedDate, order: .reverse)]
    )
    descriptor.fetchLimit = limit
    return (try? modelContext.fetch(descriptor)) ?? []
}
```

#### Personalization Queries (Phase 2)
```swift
// Get completion patterns for ML
func getCompletionPatterns() -> [MLTrainingData] {
    let descriptor = FetchDescriptor<MLTrainingData>(
        sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
    )
    return (try? modelContext.fetch(descriptor)) ?? []
}

// Get category preferences
func getCategoryPreferences() -> [(ChallengeCategory, Double)] {
    let completed = getChallengeHistory()
    let categoryStats = Dictionary(grouping: completed) { $0.challenge?.category ?? .physical }
    return categoryStats.map { (category, challenges) in
        let avgRating = challenges.compactMap { $0.userRating }.reduce(0, +) / challenges.count
        return (category, Double(avgRating))
    }
}
```

### Cloud Operations (CloudKit)

#### Challenge Sync
```swift
// Fetch latest challenges
func syncChallenges() async throws {
    let query = CKQuery(recordType: "ChallengeRecord", predicate: NSPredicate(value: true))
    query.sortDescriptors = [NSSortDescriptor(key: "updatedAt", ascending: false)]
    
    let (results, _) = try await database.records(matching: query)
    
    for (_, result) in results {
        switch result {
        case .success(let record):
            await updateLocalChallenge(from: record)
        case .failure(let error):
            print("Error fetching challenge: \(error)")
        }
    }
}

// Sync user preferences
func syncUserPreferences() async throws {
    guard let preferences = getUserPreferences() else { return }
    
    let record = CKRecord(recordType: "UserSyncRecord", recordID: CKRecord.ID(recordName: preferences.id))
    record["preferences"] = try JSONEncoder().encode(preferences)
    record["lastSyncDate"] = Date()
    record["deviceIdentifier"] = UIDevice.current.identifierForVendor?.uuidString
    
    try await database.save(record)
}
```

## Data Migration Strategy

### Version Management
```swift
enum DatabaseVersion: Int, CaseIterable {
    case v1_0 = 1
    case v1_1 = 2  // Added ML training data
    case v1_2 = 3  // Added photo metadata
    case v2_0 = 4  // Major schema changes for community features
}

func migrateDatabase(from oldVersion: DatabaseVersion, to newVersion: DatabaseVersion) {
    switch (oldVersion, newVersion) {
    case (.v1_0, .v1_1):
        migrateToMLSupport()
    case (.v1_1, .v1_2):
        migratePhotoMetadata()
    default:
        break
    }
}
```

### Backup and Recovery
- iCloud backup for user data (via CloudKit)
- Local SQLite backup before major migrations
- Graceful degradation for sync failures
- Data export capability for user data portability

## Performance Optimization

### Indexing Strategy
```swift
// SwiftData automatically indexes @Attribute(.unique) fields
// Additional compound indexes for common queries:
// - UserChallenge: (status, assignedDate)
// - Challenge: (category, isActive, difficultyLevel)
// - Photo: (userChallengeID, createdAt)
```

### Query Optimization
- Use predicates to limit fetch results
- Implement pagination for large datasets
- Cache frequently accessed challenges
- Lazy loading for photo data

### Storage Management
- Automatic photo compression
- Periodic cleanup of old data
- Configurable retention policies
- Efficient binary data storage

## Privacy and Security

### Data Protection
- Keychain storage for sensitive settings
- Core Data encryption for local database
- CloudKit private database for user sync
- No third-party analytics on personal data

### GDPR Compliance
- User data export functionality
- Complete data deletion capability
- Granular privacy controls
- Transparent data usage policies

## Monitoring and Analytics

### Database Metrics
- Query performance monitoring
- Storage usage tracking
- Sync success rates
- Migration completion rates

### Privacy-Safe Analytics
- Anonymous usage patterns
- Feature adoption rates
- Performance benchmarks
- Error tracking (without personal data)