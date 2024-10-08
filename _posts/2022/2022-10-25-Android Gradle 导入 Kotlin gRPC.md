---
title: Android Gradle 导入 Kotlin gRPC
date: 2022-10-25 14:03
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
    implementation 'io.grpc:grpc-stub:1.50.2'
    implementation 'io.grpc:grpc-protobuf-lite:1.50.2'
    implementation 'io.grpc:grpc-kotlin-stub:1.3.0'
    implementation 'com.google.protobuf:protobuf-kotlin-lite:3.21.8'
    runtimeOnly 'io.grpc:grpc-okhttp:1.46.0'
}

protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.21.8'
    }
    plugins {
        java {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.50.0'
        }
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.50.0'
        }
        grpckt {
            artifact = 'io.grpc:protoc-gen-grpc-kotlin:1.3.0:jdk8@jar'
        }
    }
    generateProtoTasks {
        all().each { task ->
            task.builtins {
                kotlin {
                    option 'lite'
                }
            }
            task.plugins {
                java {
                    option 'lite'
                }
                grpc {
                    option 'lite'
                }
                grpckt {
                    option 'lite'
                }
            }
        }
    }
}
```

参考代码

https://github.com/grpc/grpc-kotlin/tree/master/examples/android
