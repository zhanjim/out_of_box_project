# Out of Box - API Specification

## Overview

Out of Box primarily uses Apple's CloudKit as its backend service, eliminating the need for custom API development. However, this document outlines the internal service layer APIs and potential future REST API endpoints for advanced features.

## Architecture Pattern

### Primary Approach: CloudKit Integration
- **No custom backend required**: CloudKit handles data sync, storage, and distribution
- **Native iOS integration**: Seamless authentication and privacy
- **Automatic scaling**: Apple handles infrastructure
- **Privacy-first**: User data stays in Apple's ecosystem

### Internal Service APIs
The app uses internal Swift protocols and services that abstract CloudKit operations and provide a clean interface for the UI layer.

## Internal Service Layer APIs

### 1. Challenge Service Protocol

```swift
protocol ChallengeServiceProtocol {
    // Challenge Retrieval
    func getTodaysChallenge() async throws -> Challenge?
    func getAvailableChallenges(count: Int) async throws -> [Challenge]
    func getChallengeHistory(limit: Int) async throws -> [UserChallenge]
    func searchChallenges(query: String, filters: ChallengeFilters) async throws -> [Challenge]
    
    // Challenge Management
    func assignTodaysChallenge() async throws -> UserChallenge
    func completeChallenge(_ challengeID: String, photo: UIImage?) async throws
    func skipChallenge(_ challengeID: String, reason: SkipReason?) async throws
    func rateChallenge(_ challengeID: String, rating: Int) async throws
    
    // Personalization (Phase 2)
    func getPersonalizedChallenges(count: Int) async throws -> [Challenge]
    func updateUserPreferences(from interaction: ChallengeInteraction) async throws
}

// Implementation
class ChallengeService: ChallengeServiceProtocol {
    private let cloudKitService: CloudKitServiceProtocol
    private let localDataService: LocalDataServiceProtocol
    private let personalizationEngine: PersonalizationEngineProtocol?
    
    init(cloudKitService: CloudKitServiceProtocol, 
         localDataService: LocalDataServiceProtocol,
         personalizationEngine: PersonalizationEngineProtocol? = nil) {
        self.cloudKitService = cloudKitService
        self.localDataService = localDataService
        self.personalizationEngine = personalizationEngine
    }
    
    func getTodaysChallenge() async throws -> Challenge? {
        // Check if user already has today's challenge
        if let existingChallenge = try await localDataService.getTodaysUserChallenge() {
            return try await localDataService.getChallenge(id: existingChallenge.challengeID)
        }
        
        // Assign new challenge for today
        let userChallenge = try await assignTodaysChallenge()
        return try await localDataService.getChallenge(id: userChallenge.challengeID)
    }
    
    func completeChallenge(_ challengeID: String, photo: UIImage?) async throws {
        guard let userChallenge = try await localDataService.getUserChallenge(for: challengeID) else {
            throw ChallengeError.challengeNotFound
        }
        
        userChallenge.status = .completed
        userChallenge.completedDate = Date()
        
        if let photo = photo {
            let photoPath = try await savePhoto(photo, for: userChallenge)
            userChallenge.photoPath = photoPath
        }
        
        try await localDataService.save(userChallenge)
        
        // Update personalization engine if available
        await personalizationEngine?.recordCompletion(for: challengeID)
        
        // Sync to cloud
        try await cloudKitService.syncUserProgress()
    }
}
```

### 2. CloudKit Service Protocol

```swift
protocol CloudKitServiceProtocol {
    // Challenge Content
    func fetchLatestChallenges() async throws -> [ChallengeRecord]
    func fetchChallengeUpdates(since date: Date) async throws -> [ChallengeRecord]
    
    // User Data Sync
    func syncUserProgress() async throws
    func backupUserPreferences(_ preferences: UserPreferences) async throws
    func restoreUserPreferences() async throws -> UserPreferences?
    
    // Configuration
    func fetchAppConfiguration() async throws -> AppConfiguration
    func checkForContentUpdates() async throws -> Bool
}

class CloudKitService: CloudKitServiceProtocol {
    private let container: CKContainer
    private let publicDatabase: CKDatabase
    private let privateDatabase: CKDatabase
    
    init() {
        self.container = CKContainer(identifier: "iCloud.com.outofbox.app")
        self.publicDatabase = container.publicCloudDatabase
        self.privateDatabase = container.privateCloudDatabase
    }
    
    func fetchLatestChallenges() async throws -> [ChallengeRecord] {
        let query = CKQuery(recordType: "ChallengeRecord", 
                           predicate: NSPredicate(format: "isActive == YES"))
        query.sortDescriptors = [NSSortDescriptor(key: "updatedAt", ascending: false)]
        
        let (results, _) = try await publicDatabase.records(matching: query)
        
        return results.compactMap { (_, result) in
            switch result {
            case .success(let record):
                return ChallengeRecord(from: record)
            case .failure:
                return nil
            }
        }
    }
    
    func syncUserProgress() async throws {
        let userID = try await getCurrentUserID()
        let completedChallenges = try await localDataService.getCompletedChallengeIDs()
        
        let record = CKRecord(recordType: "UserSyncRecord", 
                             recordID: CKRecord.ID(recordName: userID))
        record["completedChallengeIDs"] = completedChallenges
        record["lastSyncDate"] = Date()
        record["deviceIdentifier"] = UIDevice.current.identifierForVendor?.uuidString
        
        try await privateDatabase.save(record)
    }
}
```

### 3. Photo Service Protocol

```swift
protocol PhotoServiceProtocol {
    // Photo Management
    func savePhoto(_ image: UIImage, for challengeID: String) async throws -> String
    func loadPhoto(path: String) async throws -> UIImage
    func deletePhoto(path: String) async throws
    func compressPhoto(_ image: UIImage, quality: PhotoQuality) async -> UIImage
    
    // Photo Metadata
    func getPhotoMetadata(path: String) async throws -> PhotoMetadata
    func updatePhotoMetadata(path: String, metadata: PhotoMetadata) async throws
    
    // Sharing
    func preparePhotoForSharing(_ image: UIImage) async throws -> UIImage
    func getShareablePhotoURL(path: String) async throws -> URL
}

enum PhotoQuality {
    case high      // Original size
    case medium    // 50% compression
    case low       // 25% compression
    case thumbnail // 200x200 px
}

struct PhotoMetadata {
    let filename: String
    let fileSize: Int64
    let dimensions: CGSize
    let createdAt: Date
    let location: String?
    let deviceModel: String
    var isEdited: Bool
    var compressionRatio: Double
}
```

### 4. Notification Service Protocol

```swift
protocol NotificationServiceProtocol {
    // Permission Management
    func requestNotificationPermission() async throws -> Bool
    func getNotificationSettings() async -> UNNotificationSettings
    
    // Scheduling
    func scheduleRandomNotification(in timeWindow: TimeWindow) async throws
    func scheduleDailyNotifications(for days: Int) async throws
    func cancelAllNotifications() async
    func cancelNotification(with identifier: String) async
    
    // Delivery Optimization
    func getOptimalNotificationTime() async -> Date?
    func updateNotificationTiming(based on: [NotificationInteraction]) async
}

struct TimeWindow {
    let start: Date  // Time of day
    let end: Date    // Time of day
    let timeZone: TimeZone
}

struct NotificationInteraction {
    let sentAt: Date
    let openedAt: Date?
    let challengeCompleted: Bool
    let timeFromSentToOpened: TimeInterval?
}
```

### 5. Personalization Engine Protocol (Phase 2)

```swift
protocol PersonalizationEngineProtocol {
    // Learning
    func recordCompletion(for challengeID: String) async
    func recordSkip(for challengeID: String, reason: SkipReason) async
    func recordRating(for challengeID: String, rating: Int) async
    func recordTiming(completion: Date, notification: Date) async
    
    // Prediction
    func getPersonalizedChallenges(count: Int) async throws -> [String]
    func predictCompletionProbability(for challengeID: String) async -> Double
    func getOptimalNotificationTime() async -> Date?
    func getPreferredChallengeCategories() async -> [ChallengeCategory: Double]
    
    // Model Management
    func trainModel() async throws
    func getModelMetrics() async -> PersonalizationMetrics
    func exportModelData() async throws -> Data
}

struct PersonalizationMetrics {
    let accuracy: Double
    let totalTrainingData: Int
    let lastTrainingDate: Date
    let modelVersion: String
}
```

## Data Transfer Objects

### Challenge DTOs

```swift
struct ChallengeDTO: Codable {
    let id: String
    let title: String
    let description: String
    let category: ChallengeCategory
    let timeInvestment: TimeInvestment
    let socialRequirement: SocialRequirement
    let costLevel: CostLevel
    let difficultyLevel: Int
    let tags: [String]
    let locationType: LocationType
    let seasonality: Seasonality?
    let isActive: Bool
    let createdAt: Date
    let updatedAt: Date
}

struct UserChallengeDTO: Codable {
    let id: String
    let challengeID: String
    let status: ChallengeStatus
    let assignedDate: Date
    let completedDate: Date?
    let skippedDate: Date?
    let photoPath: String?
    let userRating: Int?
    let completionTime: TimeInterval?
    let userNotes: String?
}

struct ChallengeFilters {
    let categories: [ChallengeCategory]?
    let maxTimeInvestment: TimeInvestment?
    let maxCostLevel: CostLevel?
    let socialRequirements: [SocialRequirement]?
    let difficultyRange: ClosedRange<Int>?
    let locationTypes: [LocationType]?
    let excludeCompleted: Bool
}
```

### User Preference DTOs

```swift
struct UserPreferencesDTO: Codable {
    let notificationSettings: NotificationSettingsDTO
    let challengePreferences: ChallengePreferencesDTO
    let privacySettings: PrivacySettingsDTO
    let appSettings: AppSettingsDTO
}

struct NotificationSettingsDTO: Codable {
    let enabled: Bool
    let timeWindow: TimeWindowDTO
    let timeZone: String
}

struct ChallengePreferencesDTO: Codable {
    let preferredCategories: [ChallengeCategory]
    let maxTimeInvestment: TimeInvestment
    let budgetLevel: CostLevel
    let socialComfortLevel: Int
    let difficultyPreference: Int
    let locationPreferences: [LocationType]
}
```

## Error Handling

### Service Errors

```swift
enum ChallengeError: LocalizedError {
    case challengeNotFound
    case alreadyCompleted
    case invalidChallenge
    case photoSaveFailed
    case syncFailed
    case networkUnavailable
    case unauthorized
    case quotaExceeded
    
    var errorDescription: String? {
        switch self {
        case .challengeNotFound:
            return "Challenge not found"
        case .alreadyCompleted:
            return "Challenge already completed"
        case .invalidChallenge:
            return "Invalid challenge data"
        case .photoSaveFailed:
            return "Failed to save photo"
        case .syncFailed:
            return "Failed to sync with cloud"
        case .networkUnavailable:
            return "Network connection unavailable"
        case .unauthorized:
            return "User not authenticated"
        case .quotaExceeded:
            return "Storage quota exceeded"
        }
    }
}

enum CloudKitError: LocalizedError {
    case accountNotAvailable
    case networkFailure
    case quotaExceeded
    case recordNotFound
    case conflictResolutionFailed
    
    var errorDescription: String? {
        // Error descriptions
    }
}
```

## Future REST API (Phase 3+)

### Potential Endpoints for Advanced Features

```swift
// If we ever need custom backend services
struct RestAPIEndpoints {
    // User Management
    static let userProfile = "/api/v1/user/profile"
    static let userPreferences = "/api/v1/user/preferences"
    static let userStats = "/api/v1/user/stats"
    
    // Social Features
    static let friends = "/api/v1/social/friends"
    static let friendRequests = "/api/v1/social/requests"
    static let sharedPhotos = "/api/v1/social/photos"
    static let comments = "/api/v1/social/comments"
    
    // Advanced Analytics
    static let personalInsights = "/api/v1/analytics/insights"
    static let challengeRecommendations = "/api/v1/analytics/recommendations"
    
    // Content Management
    static let reportContent = "/api/v1/content/report"
    static let contentFeedback = "/api/v1/content/feedback"
}
```

## Service Layer Architecture

### Dependency Injection

```swift
protocol ServiceContainer {
    var challengeService: ChallengeServiceProtocol { get }
    var cloudKitService: CloudKitServiceProtocol { get }
    var photoService: PhotoServiceProtocol { get }
    var notificationService: NotificationServiceProtocol { get }
    var personalizationEngine: PersonalizationEngineProtocol? { get }
}

class DefaultServiceContainer: ServiceContainer {
    lazy var challengeService: ChallengeServiceProtocol = {
        ChallengeService(
            cloudKitService: cloudKitService,
            localDataService: localDataService,
            personalizationEngine: personalizationEngine
        )
    }()
    
    lazy var cloudKitService: CloudKitServiceProtocol = {
        CloudKitService()
    }()
    
    // Other service initializations...
}
```

### Service Coordination

```swift
@MainActor
class AppCoordinator: ObservableObject {
    private let services: ServiceContainer
    
    @Published var currentChallenge: Challenge?
    @Published var isLoading: Bool = false
    @Published var error: Error?
    
    init(services: ServiceContainer) {
        self.services = services
    }
    
    func loadTodaysChallenge() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            currentChallenge = try await services.challengeService.getTodaysChallenge()
        } catch {
            self.error = error
        }
    }
    
    func completeChallenge(with photo: UIImage?) async {
        guard let challenge = currentChallenge else { return }
        
        do {
            try await services.challengeService.completeChallenge(challenge.id, photo: photo)
            // Refresh state
            await loadTodaysChallenge()
        } catch {
            self.error = error
        }
    }
}
```

## Testing Strategy

### Service Testing

```swift
// Mock implementations for testing
class MockChallengeService: ChallengeServiceProtocol {
    var mockChallenge: Challenge?
    var shouldThrowError: Bool = false
    
    func getTodaysChallenge() async throws -> Challenge? {
        if shouldThrowError {
            throw ChallengeError.networkUnavailable
        }
        return mockChallenge
    }
    
    // Other mock implementations...
}

// Unit tests
class ChallengeServiceTests: XCTestCase {
    var sut: ChallengeServiceProtocol!
    var mockCloudKit: MockCloudKitService!
    var mockLocalData: MockLocalDataService!
    
    override func setUp() {
        mockCloudKit = MockCloudKitService()
        mockLocalData = MockLocalDataService()
        sut = ChallengeService(
            cloudKitService: mockCloudKit,
            localDataService: mockLocalData
        )
    }
    
    func testGetTodaysChallenge_ReturnsExistingChallenge() async throws {
        // Given
        let existingChallenge = Challenge(id: "test", title: "Test", ...)
        mockLocalData.mockTodaysChallenge = existingChallenge
        
        // When
        let result = try await sut.getTodaysChallenge()
        
        // Then
        XCTAssertEqual(result?.id, existingChallenge.id)
    }
}
```

This API specification provides a comprehensive foundation for the Out of Box app's service layer, focusing on CloudKit integration while maintaining flexibility for future enhancements.