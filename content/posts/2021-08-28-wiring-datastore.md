---
title: "Wiring your DataStore"
date: 2021-08-28T14:13:46+03:00
categories: ['Android']
draft: true
---

DataStore is a fairly recent addition on Android developers toolkit - a powerful persistence library written in Kotlin
and using `kotlinx.coroutines` under the hood to provide neat API and great performance.

<!--more-->

There are two "flavors" of it currently available - `DataStore` and `PreferencesDataStore`, the latter being an almost
drop-in replacement for `SharedPreferences` and the most likely choice for developers getting acquainted with the tool.

While key-based persistence proved itself to be a potent instrument over the years, it tends to have some problems:

1. Key discovery may prove to be challenging, especially for project newcomers;
2. It's common to re-use the same default instance of the file which leads to it becoming bloated over the lifetime of
   the application;
3. In turn, subtle bugs may be introduced if the same key is mistakenly used to store a different piece of data;
4. Storing non-primitive types is challenging and often requires some form of serialization (JSON being arguably the
   most popular one);
5. Enums usually require an adapter of sorts, either `String`-based or, much worse, ordinal-based.

`DataStore` allows to use any custom data serialization format as long as it can be serialized to disk using standard
Java APIs (`InputStream` & `OutputStream`).

There are a wide range of formats that can be used for this purpose, but in this post we'll explore [Protocol buffers]
as they offer some facilities that can help with the issues mentioned above.

## Wire

[Wire] is a Kotlin library built by Square that generates Kotlin classes from `.proto` files. It requires minimal
configuration, is extremely fast and well-documented.

To set up wire, make sure you do the following in your root `build.gradle.kts`:

```kotlin
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("com.squareup.wire:wire-gradle-plugin:3.7.0")
    }
}
```

Now, add the Wire plugin to your module's `build.gradle.kts`:

```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
    id("com.squareup.wire")
}

wire {
    kotlin { }
}
```

At this point, Wire may complain about missing directories. Please follow the instructions that the plugin provides and
create a `src/main/proto` directory that follows your package name (e.g. `scr/main/proto/com/example/library`).

## Defining a .proto file

Wire works with schema defined by you using Protocol buffers. Let's create a `chat.proto` file that defines user
preferences for a chat screen:

```protobuf
syntax = "proto3";

package com.example.library;

message Chat {
    bool large_emoji = 1;
    enum ListView {
        TWO_LINE = 0;
        THREE_LINE = 1;
    }
    ListView list_view = 2;
}
```

Building the module will generate a Kotlin representation of `com.example.library.Chat` which you can inspect and
instantiate.

## DataStore

Setup is simple and straightforward - just add a dependency to your module's `build.gradle.kts` file:

```kotlin
dependencies {
    implementation("androidx.datastore:datastore:1.0.0")
}
```

## Wiring it all together

To read and write our chat preferences we need to provide a `Serializer` which will be used by `DataStore` to perform
said operations:

```kotlin
object ChatSerializer : Serializer<Chat> {

    override val defaultValue: Chat
        get() = Chat(large_emoji = true)

    override suspend fun readFrom(input: InputStream): Chat =
        withContext(Dispatchers.IO) { Chat.ADAPTER.decode(input) }

    override suspend fun writeTo(t: Chat, output: OutputStream) =
        withContext(Dispatches.IO) { t.encode(output) }
}
```

`Serializer` requires us to provide a `defaultValue` for when no file was found on the disk; it also allows us to
provide non-standard default values for certain fields (i.e. `large_emoji` is `false` by default as per Protocol
buffers, but we really want it to be `true`).

Wire generates an `ADAPTER` that we can use in both `readFrom`/`writeTo` (it is also used in `Chat.encode()` under the
hood). Wrapping the calls in `withContext(Dispatchers.IO)` makes sure we do IO only on a background `Dispatcher`.

It's time to create a `DataStore` instance and use it across the app:

```kotlin
val Context.chatDataStore by dataStore(
    filename = "chat.pb",
    serializer = ChatSerializer
)
```

You may also want to use Dagger/Hilt (or another dependency injection tool) to provide this instance:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ChatDataStoreModule {

    @Provides
    @Singleton
    fun Application.provideChatDataStore(): DataStore<Chat> =
        DataStoreFactory.create(ChatSerializer) { dataStoreFile("chat.pb") }
}
```

Do note that only one `DataStore` instance should read the file; otherwise you may run into all kinds of issues.

[Protocol buffers]: https://developers.google.com/protocol-buffers

[Wire]: https://github.com/square/wire

[Store typed objects with Proto DataStore]: https://developer.android.com/topic/libraries/architecture/datastore#proto-datastore
