# SwiftSecurity

[![Platforms](https://img.shields.io/badge/platforms-_iOS_|_macOS_|_watchOS_|_tvOS_|_visionOS-lightgrey.svg?style=flat)](https://developer.apple.com/resources/)
[![SPM supported](https://img.shields.io/badge/SPM-supported-DE5C43.svg?style=flat)](https://swift.org/package-manager)
[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat)](http://mit-license.org)

SwiftSecurity is a modern Swift API for Apple [Security](https://developer.apple.com/documentation/security) framework (Keychain API, SharedWebCredentials API, etc). Secure the data your app manages in a much easier way with compile-time checks. 

## Features

How does SwiftSecurity differ from other wrappers?

* Supports every [Keychain item class](https://developer.apple.com/documentation/security/keychain_services/keychain_items/item_class_keys_and_values) (Generic & Internet Password, Key, Certificate and Identity).
* Prevents creation of an incorrect set of [attributes](https://developer.apple.com/documentation/security/keychain_services/keychain_items/item_attribute_keys_and_values) for items.
* Compatible with [CryptoKit](https://developer.apple.com/documentation/cryptokit/) and [SwiftUI](https://developer.apple.com/documentation/swiftui/).
* Clear of deprecated and legacy calls.

## Installation

#### Requirements

* iOS 14.0+ / macOS 11.0+ / Mac Catalyst 14.0+ / watchOS 7.0+ / tvOS 14.0+ / visionOS 1.0+
* Swift 5.9

#### Swift Package Manager

To use the `SwiftSecurity`, add the following dependency in your `Package.swift`:
```swift
.package(url: "https://github.com/dm-zharov/swift-security.git", from: "1.0.0")
```

Finally, add `import SwiftSecurity` to your source code.

##  Quick Start

####  Basic

```swift
// Choose Keychain
let keychain = Keychain.default

// Store secret
try keychain.store("8e9c0a7f", query: .credential(for: "OpenAI"))

// Retrieve secret
let token: String? = try keychain.retrieve(.credential(for: "OpenAI"))

// Remove secret
try keychain.remove(.credential(for: "OpenAI"))
```

#### Basic (SwiftUI)

```swift
struct AuthView: View {
    @Credential("OpenAI") private var token: String?

    var body: some View {
        VStack {
            Button("Save") {
                // Store secret
                try? _token.store("8e9c0a7f")
            }
            Button("Delete") {
                // Remove secret
                try? _token.remove()
            }
        }
        .onChange(of: token) {
            if let token {
                // Use secret
            }
        }
    }
} 
```

#### Web Credential

```swift
// Store password for a website
try keychain.store(
    password, query: .credential(for: "username", space: .website("https://example.com"))
)

// Retrieve password for a website
let password: String? = try keychain.retrieve(
    .credential(for: "username", space: .website("https://example.com"))
)
```

For example, if you need to store distinct ports credentials for the same user working on the same server, you might further characterize the query by specifying protection space.

```swift
let space1 = WebProtectionSpace(host: "example.com", port: 443)
try keychain.store(password1, query: .credential(for: user, space: space1))

let space2 = WebProtectionSpace(host: "example.com", port: 8443)
try keychain.store(password2, query: .credential(for: user, space: space2))
```

#### Get Attributes

```swift
if let info = try keychain.info(for: .credential(for: "OpenAI")) {
    // Creation date
    print(info.creationDate)
    // Comment
    print(info.comment)
    ...
}
```

#### Remove All

```swift
try keychain.removeAll()
```

## Advanced Usage

#### Query

```swift
// Create query
var query = SecItemQuery<GenericPassword>()

// Customize query
query.synchronizable = true
query.service = "OpenAI"
query.label = "OpenAI Access Token"

// Perform query
try keychain.store(secret, query: query, accessPolicy: AccessPolicy(.whenUnlocked, options: .biometryAny))
_ = try keychain.retrieve(query, authenticationContext: LAContext())
try keychain.remove(query)
```

Query prevents the creation of an incorrect set of attributes for item:

```swift
var query = SecItemQuery<InternetPassword>()
query.synchronizable = true  // ✅ Common
query.server = "example.com" // ✅ Only for `InternetPassword`
query.service = "OpenAI"     // ❌ Only for `GenericPassword`, so not accessible
query.keySizeInBits = 2048   // ❌ Only for `SecKey`, so not accessible
```

Possible queries:

```swift
SecItemQuery<GenericPassword>   // kSecClassGenericPassword
SecItemQuery<InternetPassword>  // kSecClassInternetPassword
SecItemQuery<SecKey>            // kSecClassSecKey
SecItemQuery<SecCertificate>    // kSecClassSecCertificate
SecItemQuery<SecIdentity>       // kSecClassSecIdentity
```

#### Get Data & Persistent Reference

```swift
// Retrieve multiple values at once
if case let .dictionary(info) = try keychain.retrieve([.data, .persistentReference], query: .credential(for: "OpenAI")) {
    // Data
    info.data
    // Persistent Reference
    info.persistentReference
}
```

#### CryptoKit

```swift
// Store private key
let privateKey = P256.KeyAgreement.PrivateKey()
try keychain.store(privateKey, query: .privateKey(for: "Alice"))

// Retrieve private key (+ public key)
let privateKey: P256.KeyAgreement.PrivateKey? = try keychain.retrieve(.privateKey(for: "Alice"))
let publicKey = privateKey.publicKey
```

#### X.509 Certificate

```swift
import X509 // package from `https://github.com/apple/swift-certificates`

// Prepare certificate
let certificateData: Data = ... // content of file with `cer`/`der` extension 
try! certificate = Certificate(derRepresentation: certificateData) // entity from `X509` package

// Store certificate
try keychain.store(certificate, query: .certificate(for: "Apple"))
```

#### Identity (Certificate + PrivateKey)

```
// Import digital identity from `PKCS #12` data
let pkcs12Data = ... // content of file, often with `p12` extension
for importItem in try keychain.import(pkcs12Data, passphrase: "8e9c0a7f") {
    if let identity = importItem.identity {
        // Store digital identity
        try keychain.store(identity, query: .identity(for: "Apple Development"))
    }
}

// Retrieve digital identity
if let identity = try keychain.retrieve(.identity(for: "Apple Development")) {
    identity.rawRepresentation // SecIdentity
}
```

#### Debug

```swift
// Print Query (or use LLDB po command)
print(query.debugDescription) // ["Class: GenericPassword", ..., "Service: OpenAI"]

// Print Keychain 
print(keychain.debugDescription)
```

#### Error Handling

```swift
do {
    try keychain.store("8e9c0a7f", query: .credential(for: "OpenAI"))
} catch {
    switch error as? SwiftSecurityError {
    case .duplicateItem:
        // handle duplicate
    default:
        // unhandled
    }
}
```

## How to Choose Keychain

#### Default

```swift
let keychain = Keychain.default
```

The system considers the first item in the list of [keychain access groups](https://developer.apple.com/documentation/security/keychain_services/keychain_items/sharing_access_to_keychain_items_among_a_collection_of_apps/) to be the app’s default access group, evaluated in this order:
- The optional [Keychain Access Groups Entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/keychain-access-groups) holds an array of strings, each of which names an access group.
- Application identifier, formed as the team identifier (team ID) plus the bundle identifier (bundle ID). For example, `J42EP42PB2.com.example.app`.

If the [Keychain Sharing](https://developer.apple.com/documentation/xcode/configuring-keychain-sharing#Specify-the-default-keychain-group) capability is not enabled, the default access group is `app ID`.

> [!NOTE]
> To enable macOS support, make sure to include the [Keychain Sharing (macOS)](https://developer.apple.com/documentation/xcode/configuring-keychain-sharing#Specify-the-default-keychain-group) capability and create a group `${TeamIdentifierPrefix}com.example.app`, to prevent errors in operations. This sharing group is [automatically generated](https://developer.apple.com/documentation/security/keychain_services/keychain_items/sharing_access_to_keychain_items_among_a_collection_of_apps/#2974917) for other platforms and accessible without capability. You could refer to [TestHost](https://github.com/dm-zharov/swift-security/tree/main/Tests/TestHost.xcodeproj) for information regarding project configuration.

#### Sharing within Keychain Group

If you prefer not to rely on the automatic behavior of default storage selection, you have the option to explicitly specify a keychain sharing group.

```swift
let keychain = Keychain(accessGroup: .keychainGroup(teamID: "J42EP42PB2", nameID: "com.example.app"))
```

#### Sharing within App Group

Sharing could also be achieved by using [App Groups](https://developer.apple.com/documentation/xcode/configuring-app-groups) capability. Unlike a keychain sharing group, the app group can’t automatically became the default storage for keychain items. You might already be using an app group, so it's probably would be the most convenient choice.

```swift
let keychain = Keychain(accessGroup: .appGroupID("group.com.example.app"))
```

> [!NOTE]
> Use `Sharing within Keychain Group` for sharing on macOS, as the described behavior is not present on this platform. There's no issue with using one sharing solution on one platform and a different one on another.

## 🔓 Protection with Face ID (Touch ID) and Passcode

#### Store protected item

```swift
// Store with specified `AccessPolicy`
try keychain.store(
    secret,
    query: .credential(for: "FBI"),
    accessPolicy: AccessPolicy(.whenUnlocked, options: .userPresence) // Requires biometry/passcode authentication
)
```

#### Retrieve protected item

If you request the protected item, an authentication screen will automatically appear.

```swift
// Retrieve value
try keychain.retrieve(.credential(for: "FBI"))
```

If you want to manually authenticate before making a request or customize authentication screen, provide [LAContext](https://developer.apple.com/documentation/localauthentication/lacontext) to the retrieval method.

```swift
// Create an LAContext
var context = LAContext()

// Authenticate
do {
    let success = try await context.evaluatePolicy(
        .deviceOwnerAuthentication,
        localizedReason: "Authenticate to proceed." // Authentication prompt
    )
} else {
    // Handle LAError error
}

// Check authentication result 
if success {
    // Retrieve value
    try keychain.retrieve(.credential(for: "FBI"), authenticationContext: context)
}

```

> [!WARNING]
> Include the [NSFaceIDUsageDescription](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW75) key in your app’s Info.plist file. Otherwise, authentication request may fail.

## Data Types

You can store, retrieve, and remove various types of values.

```swift
Foundation:
    - Data // GenericPassword, InternetPassword
    - String // GenericPassword, InternetPassword
CryptoKit:
    - SymmetricKey // GenericPassword
    - Curve25519 -> PrivateKey // GenericPassword
    - SecureEnclave.P256 -> PrivateKey // GenericPassword (SE's Key Data is Persistent Reference)
    - P256, P384, P521 -> PrivateKey // SecKey (ANSI x9.63 Elliptic Curves)
X509 (from package `apple/swift-certificates`):
    - Certificate // SecCertificate
SwiftSecurity:
    - PKCS12.Blob // Import as SecIdentity (SecCertificate + SecKey)
```

To add support for custom types, you can extend them by conforming to the following protocols.

```swift
// Store as Data (GenericPassword, InternetPassword)
extension CustomType: SecDataConvertible {}

// Store as Key (ANSI x9.63, Elliptic Curves)
extension CustomType: SecKeyConvertible {}

// Store as Certificate (X.509)
extension CustomType: SecCertificateConvertible {}
```

These protocols are inspired by Apple's sample code from the [Storing CryptoKit Keys in the Keychain](https://developer.apple.com/documentation/cryptokit/storing_cryptokit_keys_in_the_keychain) article.

## 🔑 Shared Web Credential

> [!TIP]
> [SharedWebCredentials API](https://developer.apple.com/documentation/security/shared_web_credentials) makes it possible to share credentials with the website counterpart. For example, a user may log in to a website in Safari and save credentials to the iCloud Keychain. Later, the user may run an app from the same developer, and instead of asking the user to reenter a username and password, it could access the existing credentials. The user can create new accounts, update passwords, or delete account from within the app. These changes should be saved from the app to be used by Safari.

```swift
// Store
SharedWebCredential.store("https://example.com", account: "username", password: "secret") { result in
    switch result {
    case .failure(let error):
        // Handle error
    case .success:
        // Handle success
    }
}

// Remove
SharedWebCredential.remove("https://example.com", account: "username") { result in
    switch result {
    case .failure(let error):
        // Handle error
    case .success:
        // Handle success
    }
}

// Retrieve
// - Use `ASAuthorizationController` to make an `ASAuthorizationPasswordRequest`.
```

## 🔒 Secure Data Generator

```swift
// Data with 20 uniformly distributed random bytes
let randomData = try SecureRandomDataGenerator(count: 20).next()
```

## Security

The framework’s default behavior provides a reasonable trade-off between security and accessibility.

- `kSecUseDataProtectionKeychain: true` helps to improve the portability of code across platforms. Can't be changed.
- `kSecAttrAccessibleWhenUnlocked` makes keychain items accessible from `background` processes. Changeable by `AccessPolicy`.

## Communication

- If you **found a bug**, open an issue.
- If you **have a feature request**, open an issue.
- If you **want to contribute**, submit a pull request.

## Knowledge

* [Sharing access to keychain items among a collection of apps](https://developer.apple.com/documentation/security/keychain_services/keychain_items/sharing_access_to_keychain_items_among_a_collection_of_apps/)
* [Storing CryptoKit Keys in the Keychain](https://developer.apple.com/documentation/cryptokit/storing_cryptokit_keys_in_the_keychain)
* [TN3137: On Mac keychain APIs and implementations](https://developer.apple.com/documentation/technotes/tn3137-on-mac-keychains)
* [SecItem: Fundamentals](https://developer.apple.com/forums/thread/724023)
* [SecItem: Pitfalls and Best Practices](https://developer.apple.com/forums/thread/724013)

## Author

Dmitriy Zharov, contact@zharov.dev

## License

SwiftSecurity is available under the MIT license. See the [LICENSE file](https://github.com/dm-zharov/swift-security/blob/master/LICENSE) for more info.
