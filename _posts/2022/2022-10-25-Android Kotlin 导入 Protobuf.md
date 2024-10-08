---
title: Android Kotlin 导入 Protobuf
date: 2022-10-25 14:34
---

project build.gradle
```groovy
plugins {
    id "com.google.protobuf" version "0.9.1" apply false
}
```

module build.gradle
```groovy
android {
    sourceSets {
        main {
            proto {
                srcDir '../../protos'
            }
        }
    }
}

dependencies {
    implementation 'com.google.protobuf:protobuf-kotlin-lite:3.21.8'
}

protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.21.8'
    }

    generateProtoTasks {
        all().each { task ->
            task.builtins {
                kotlin {
                    option 'lite'
                }
                java {
                    option 'lite'
                }
            }
        }
    }
}
```

参考代码

https://github.com/google/protobuf-gradle-plugin
