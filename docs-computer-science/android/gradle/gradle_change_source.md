# Gradle换源


## gradle-wrapper.properties 换源

```txt
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
#distributionUrl=https\://services.gradle.org/distributions/gradle-8.9-bin.zip
distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-8.9-all.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionSha256Sum=258e722ec21e955201e31447b0aed14201765a3bfbae296a46cf60b70e66db70

# 可以将 gradle.org 换成 tencent 源
# 下载的 gradle 可以下载 all，避免gradle的其他包走 gradle.org 下载
```

## 配置单个项目gradle包下载

> 修改`build.gradle`文件
> 添加或修改下面内容（内容仅供参考）

```gradle
pluginManagement {
    repositories {
         // 使用阿里云镜像替代默认仓库
        maven { url=uri ("https://maven.aliyun.com/repository/google")}
        maven { url=uri ("https://maven.aliyun.com/repository/central")}
        maven { url=uri ("https://maven.aliyun.com/repository/gradle-plugin")}
        maven { url=uri ("https://maven.aliyun.com/repository/public")}

        google()
        mavenCentral()
        gradlePluginPortal()
    }
}


dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        // 使用阿里云镜像替代默认仓库
        maven { url=uri ("https://maven.aliyun.com/repository/google")}
        maven { url=uri ("https://maven.aliyun.com/repository/central")}
        maven { url=uri ("https://maven.aliyun.com/repository/gradle-plugin")}
        maven { url=uri ("https://maven.aliyun.com/repository/public")}

        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```



## 配置全局gradle包下载

> 添加或修改 `init.gradle.kts` 文件
>
>（Mac） `/Users/youname/.gradle\` 目录
>（Win） `C:\Users\youname\.gradle\` 目录

```kts
fun RepositoryHandler.enableMirror() {
    all {
        if (this is MavenArtifactRepository) {
            val originalUrl = this.url.toString().removeSuffix("/")
            urlMappings[originalUrl]?.let {
                logger.lifecycle("Repository[$url] is mirrored to $it")
                this.setUrl(it)
            }
        }
    }
}

val urlMappings = mapOf(
    "https://repo.maven.apache.org/maven2" to "https://mirrors.tencent.com/nexus/repository/maven-public/",
    "https://dl.google.com/dl/android/maven2" to "https://mirrors.tencent.com/nexus/repository/maven-public/",
    "https://plugins.gradle.org/m2" to "https://mirrors.tencent.com/nexus/repository/gradle-plugins/"
)

gradle.allprojects {
    buildscript {
        repositories.enableMirror()
    }
    repositories.enableMirror()
}

gradle.beforeSettings { 
    pluginManagement.repositories.enableMirror()
    dependencyResolutionManagement.repositories.enableMirror()
}
```


