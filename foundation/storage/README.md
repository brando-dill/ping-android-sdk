<p align="center">
  <a href="https://github.com/ForgeRock/ping-android-sdk">
    <img src="https://www.pingidentity.com/content/dam/picr/nav/Ping-Logo-2.svg" alt="Logo">
  </a>
  <hr/>
</p>

# Ping Storage SDK

The Ping Storage SDK provides a flexible storage interface and a set of common storage solutions for
the Ping SDKs.

## Integrating the SDK into your project

To add the Ping Storage SDK as a dependency to your project, include the following in
your `build.gradle` file:

```kotlin
dependencies {
    implementation(project(":foundation:storage"))
}
```

## How to Use the SDK

### Creating and Using a Storage Instance

To create a storage instance and use it to persist and retrieve data, follow the example below:

```kotlin
// Define the data class that you want to persist
@Serializable
data class Dog(val name: String, val type: String)

val storage = EncryptedSharedPreferencesStorage<Dog>("myId") // Create the Storage
storage.save(Dog("Lucky", "Golden Retriever")) // Persist the data object

val storedData = storage.get() // Retrieve the object
```

EncryptedSharedPreferencesStorage is a storage solution that
uses [EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences)
to store data securely.

### Enabling Cache for the Storage

You can enable cache for the storage as follows, by default cache is disabled:

```kotlin
val storage =
    EncryptedSharedPreferencesStorage<Dog>("myId", cacheable = true) // Create the Storage with cache enabled
```

### DataStorePreferencesStorage

DataStorePreferenceStorage
uses [Preferences DataStore](https://developer.android.com/topic/libraries/architecture/datastore#preferences-datastore)
to store data.

```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")
val storage = DataStorePreferencesStorage<Dog>(context.dataStore)
```

### DataStoreStorage

DataStoreStorage
uses [DataStore](https://developer.android.com/topic/libraries/architecture/datastore) to store
data.

#### Serialize Object to Json, then persist it in DataStore

```kotlin
val Context.dataStore: DataStore<Dog?> by dataStore("filename", DataToJsonSerializer())
val storage = DataStoreStorage<Dog>(context.dataStore)
```

OR

#### Serialize Object to Json, encrypt the Json with key stored in AndroidKeyStore, then persist it in DataStore

```kotlin
val Context.dataStore: DataStore<Data?> by dataStore("filename", EncryptedDataToJsonSerializer(
    SecretKeyEncryptor {
        logger = Logger.CONSOLE
        keyAlias = "myKeyAlias"
    }
))
val storage = DataStoreStorage<Dog>(context.dataStore)
```

### Encryptor

You can use the `SecretKeyEncryptor` to encrypt and decrypt data. The `SecretKeyEncryptor` uses the
AndroidKeyStore to store the key securely.
or you can create your own encryptor by implementing the `Encryptor` interface.

```kotlin
interface Encryptor {
    suspend fun encrypt(data: ByteArray): ByteArray
    suspend fun decrypt(data: ByteArray): ByteArray
}
```

There are configuration options for `SecretKeyEncryptor`:

```kotlin
 val encryptor = SecretKeyEncryptor {
    keyAlias = "TheKeyAlias"
    enforceAsymmetricKey = true // Flag to enforce the use of an asymmetric key. default is false
    secretKeyStorage = keyStorage // The storage for the secret key.
    throwWhenEncryptError =
        true // Flag to throw an exception when an error occurs during encryption. default is true
    strongBoxPrefered = false // Flag to prefer StrongBox for key storage. default is true
}
```

**Note:** StrongBox offers the strongest security, best for apps truly at risk of physical attacks. But it's slower and consume more resources.


### Creating a Custom Storage

You can create a custom repository by implementing the `Storage` interface. This could be useful
for creating file-based storage, cloud storage, etc...
Here is an example of creating a custom memory storage:

```kotlin
class Memory<T : @Serializable Any> : Storage<T> {
    private var data: T? = null

    override suspend fun save(item: T?) {
        data = item
    }

    override suspend fun get(): T? = data

    override suspend fun delete() {
        data = null
    }
}

// Delegate the MemoryStorage to the Storage
inline fun <reified T : @Serializable Any> MemoryStorage(): Storage<T> = StorageDelegate(Memory())
```

## Available Storage Solutions

The Ping Storage SDK provides the following storage solutions:

| Storage                           | Description                                                                                                                                                                                                  |
|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| EncryptedSharedPreferencesStorage | Storage backed by [EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences), the EncryptedSharedPreferences may soon be deprecated by Google |
| DataStoreStorage                  | Storage that store data in  [DataStore](https://developer.android.com/topic/libraries/architecture/datastore).                                                                                               |
| MemoryStorage                     | Storage that store data in memory.                                                                                                                                                                           |

## Special Storage Solution

### SQLiteStorage

`SQLiteStorage` provides encrypted, table-based storage using SQLCipher. It is suitable for securely storing structured data and supports custom table creation and multiple tables in a single database. It is ideal for use cases requiring relational data or advanced queries.

### Key Components

#### PassphraseProvider Interface

The `PassphraseProvider` interface is the core abstraction for passphrase management. It defines a single method:

```kotlin
fun getPassphrase(): String
```

This method should return a passphrase suitable for encrypting database files or other sensitive data.

#### Available Implementations

##### KeyStorePassphraseProvider (Default)

Uses Android's KeyStore system for secure storage of the passphrase, with encryption using AES/GCM. This is the recommended default provider for most applications.

```kotlin
val provider = KeyStorePassphraseProvider(context)
```

##### DataStorePassphraseProvider 

Uses Android's DataStore for secure, persistent storage of the passphrase.

```kotlin
val provider = DataStorePassphraseProvider(context)
```

##### BlockStorePassphraseProvider

Uses Android's Block Store (API 30+) for secure, cloud-backed storage of the passphrase. Useful for device migration and backup scenarios.

```kotlin
val provider = BlockStorePassphraseProvider(context)
```

#### NonePassphraseProvider

A provider that always returns an empty passphrase, effectively disabling SQLCipher encryption. Use this only when encryption is not required or external encryption is applied.

```kotlin
val provider = NonePassphraseProvider()
```

##### FixedPassphraseProvider

A simple provider that returns a fixed passphrase. Useful for development or when a specific passphrase is required.

```kotlin
// With a specific passphrase
val provider = FixedPassphraseProvider("my-secure-passphrase")

// With a randomly generated passphrase
val provider = FixedPassphraseProvider()
```

#### Creating Providers

You can create providers directly:

```kotlin
// Create a KeyStorePassphraseProvider (recommended default)
val provider = KeyStorePassphraseProvider(context)

// Create a DataStorePassphraseProvider
val provider = DataStorePassphraseProvider(context)

// Create a fixed provider (for testing)
val provider = FixedPassphraseProvider("my-passphrase")
```

#### Creating SQLiteStorage instances

When creating a SQLiteStorage instance, you can provide a custom PassphraseProvider:

```kotlin
// For production code
val storage = SQLiteStorage(
    context = context,
    passphraseProvider = KeyStorePassphraseProvider(context)
)

// For test code
val storage = SQLiteStorage(
    context = context,
    passphraseProvider = TestPassphraseProvider()
)
```

Or use the default provider:

```kotlin
val storage = SQLiteStorage(context)
```

### Using Initial Passphrases

If you want to supply an initial passphrase instead of having one randomly generated:

```kotlin
val provider = DataStorePassphraseProvider(
    context = context,
    initialPassphrase = "my-secure-passphrase"
)
```

This is useful when you need to ensure consistent passphrases across installations or for migration purposes.


#### Basic Usage

```kotlin
val storage = SQLiteStorage(context)
storage.initializeDatabase() // Initializes the encrypted database

// Register a table creator (run before storing data)
storage.registerTableCreator { db ->
    db.execSQL("""
        CREATE TABLE IF NOT EXISTS my_table (
            type TEXT NOT NULL,
            id TEXT NOT NULL,
            data TEXT NOT NULL,
            PRIMARY KEY(type, id)
        )
    """)
}

// Store an item (suspend function)
storage.storeItem("my_table", "Dog", "lucky", "{\"name\":\"Lucky\",\"type\":\"Golden Retriever\"}")

// Retrieve an item (suspend function)
val dogJson = storage.retrieveItem("my_table", "Dog", "lucky")

// Delete an item (suspend function)
val deleted = storage.deleteItem("my_table", "Dog", "lucky")
```

#### Notes
- The database is encrypted using a passphrase from a `PassphraseProvider` (by default, `KeyStorePassphraseProvider`).
- You can register multiple table creators for different tables.
- All operations are suspend functions and should be called from a coroutine.
- For advanced usage, you can subclass `SQLiteStorage` to add custom logic or migrations.
