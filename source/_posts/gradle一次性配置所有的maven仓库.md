
title: gradle一次性配置所有的maven仓库
date: 2017-04-17 17:32:32
tags: [youdaonote]
---


在~/.gradle/init.gradle文件中添加
```
allprojects{
    repositories {
        def REPOSITORY_URL = 'http://maven.oschina.net/content/groups/public'
        all { ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                    remove repo
                }
            }
        }
        maven {
            url REPOSITORY_URL
        }
    }
}
```

参考：
- https://docs.gradle.org/current/userguide/init_scripts.html
- https://yrom.net/blog/2015/02/07/change-gradle-maven-repo-url/
