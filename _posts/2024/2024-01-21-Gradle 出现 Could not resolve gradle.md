---
title: Gradle 出现 Could not resolve gradle
date: 2024-01-21 15:02
---

Gradle 在进行 sync 的时候会出现

````
Caused by: org.gradle.internal.resolve.ModuleVersionResolveException: Could not resolve gradle:gradle:8.2.
````

查看异常信息发现 Gradle 无法下载 `https://services.gradle.org/distributions/gradle-8.2-src.zip`，这个链接重定向到 `https://github.com/gradle/gradle-distributions/releases/download/v8.2.0/gradle-8.2-src.zip`，Github 很难连上。

在 `gradle-wrapper.properties` 中 `distributionUrl` 设置为 `https\://mirror.nju.edu.cn/gradle/gradle-8.2-bin.zip` ，Gradle 仍然会下载 `https://services.gradle.org/distributions/gradle-8.2-src.zip`。为什么 Gradle 不使用镜像源呢？翻了一下 Gradle 的源码，发现这个链接是写死的。

```kotlin
private
fun createSourceRepository() = ivy {
    val repoName = repositoryNameFor(gradleVersion)
    name = "Gradle $repoName"
    setUrl("https://services.gradle.org/$repoName")
    metadataSources {
        artifact()
    }
    patternLayout {
        if (isSnapshot(gradleVersion)) {
            ivy("/dummy") // avoids a lookup that interferes with version listing
        }
        artifact("[module]-[revision](-[classifier])(.[ext])")
    }
}
```

没有任何方法可以直接修改这个链接。

要解决这个问题，可以直接为 Gradle 设置代理进行网络加速。但是这样会导致之前设置的 Maven 镜像链接也会经过代理。

继续翻看 Gradle 源码发现有这样一段代码

```kotlin
    private
    fun sourceRootsOf(gradleInstallation: File, sourceDistributionResolver: SourceDistributionProvider): Collection<File> =
        gradleInstallationSources(gradleInstallation) ?: downloadedSources(sourceDistributionResolver)


    private
    fun gradleInstallationSources(gradleInstallation: File) =
        File(gradleInstallation, "src").takeIf { it.exists() }?.let { subDirsOf(it) }
```

 `gradleInstallation` 存在 `src` 目录的时候就不会继续下载 `gradle-8.2-src.zip`。继续往上翻，发现这个值就是 `project.gradle.gradleHomeDir`

直接把这个变量在 `build.gradle.kts` 中打印出来就是 `%USERPROFILE%\.gradle\wrapper\dists\gradle-8.2-bin\4zwrvmkltlrdjhbk3gu6ax49g\gradle-8.2`。这个文件夹就是 `gradle-8.2-bin.zip` 解压后的。

于是直接把`gradle-wrapper.properties`里 `distributionUrl` 的 `bin` 改为 `all`，再把 `distributionSha256Sum` 修改为对应的值。也就是

```properties
distributionSha256Sum=5022b0b25fe182b0e50867e77f484501dba44feeea88f5c1f13b6b4660463640
distributionUrl=https\://mirror.nju.edu.cn/gradle/gradle-8.2-all.zip
```

直接 Build 通过，没有任何问题。

`gradle-8.2-all.zip` 里面已经包含了 `src` 目录，Gradle 不会继续下载 `src`。

可以查询 https://gradle.org/release-checksums 找到对应版本的 `distributionSha256Sum`。如果本来就没用它，可以不改这个值。