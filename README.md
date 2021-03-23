## 建立Task

### 宣告Task 

### 執行Task

- 方法一、已安裝Gradle

有在全域變數設定Gradle

```
gradle $TASK_NAME
```

- 方法二、Gradlew

在專案目錄有下載Gradle wrapper

```
./gradlew $TASK_NAME
```


## 撰寫腳本

### 宣告變數

- 區域變數(Local variables)

藉由`val`來當作宣告關鍵字

```
val hello = "hello"

tasks.register("print") {
	println(hello)
}
```

- 額外屬性(Extra properties)

```
val appVersion by extra("1.0.0")

tasks.register("printProperties") {
    doLast {
        println(appVersion)
    }
}
```

更多細節參考[ExtraProperty]

## 處理檔案

### 建立資料夾

`Project.mkdir（java.lang.Object）`可以快速地建立一個資料夾，方便我們管理檔案存放的位置。

```
tasks.register("ensureDirectory") {
    doLast {
        mkdir("images")
    }
}
```

### 移動檔案或目錄

當然建立完資料夾或檔案後，可能會覺得放的位置不合適，會想要移動。  
雖然Gradle沒有提供相關API，但可以使用`Apache Ant integration`來達到一樣的目的。

`ant.withGroovyBuilder`第一個參數：要被移動的資料夾，第二個參數：目的地資料夾。

```
tasks.register("moveFile") {
    doLast {
        ant.withGroovyBuilder {
            "move"("file" to "${buildDir}/reports", "todir" to "${buildDir}/toArchive")
        }
    }
}
```

### 刪除檔案和目錄

透過`Delete`的Task或是`Project.delete(org.gradle.api.Action)`的方法，可以簡單地將資料夾或是檔案刪除。

例如：刪除整個build資料夾

- 透過Delete Task

```
tasks.register<Delete>("deleteWithTask") {
    delete(buildDir)
}
```

- 透過Project.delete()

```
tasks.register("deleteWithProject") {
    delete(buildDir)
}
```

## 處理複製的工作

### 複製檔案

複製檔案的過程很簡單，基本上會有以下幾個流程概念

- 定義一個Copy的Task
- 指定要被複製的檔案或目錄
- 指定被複製檔案的目的地

範例：一個Copy的Task結構

```
tasks.register<Copy>("copyTask") {
    from()
    into() {

    }
    include()
    exclude()
}
```

`Copy`是`CopySpec`的實現，它提供了兩個重要的方法。

- CopySpec.from()：從哪裡複製，接受的參數比較彈性，例如：String、File、FileCollection或從另一個Task結果取得。

- CopySpec.into()： 要複製到哪裡，接受的參數為任何`Project.file()`參數。

範例：各種傳入參數的Copy Task。

```
tasks.register<Copy>("anotherCopyTask") {
    // Copy everything under src/main/webapp
    from("src/main/webapp")
    // Copy a single file
    from("src/staging/index.html")
    // Copy the output of a task
    from(copyTask)
    // Copy the output of a task using Task outputs explicitly.
    from(tasks["copyTaskWithPatterns"].outputs)
    // Copy the contents of a Zip file
    from(zipTree("src/main/assets.zip"))
    // Determine the destination directory later
    into({ getDestDir() })
}
```

### 過濾檔案

Copy的Task還提供以下兩個方法，快速的塞選想的檔案。

- CopySpec.include()：包括檔案。
- CopySpec.exclude()：排除檔案。

```
tasks.register<Copy>("copyTaskWithPatterns") {
    from("src/main/webapp")
    into("$buildDir/explodedWar")
    include("**/*.html")
    include("**/*.jsp")
    exclude { details: FileTreeElement ->
        details.file.name.endsWith(".html") &&
            details.file.readText().contains("DRAFT")
    }
}
```

### 重新命名

有時候檔案被複製後的名字可能不是這麼合適時，會想要將檔案重新命名，可以使用Copy的`rename`來達到目的。

rename()接受兩種參數

- 使用正規表達式
- 使用閉包

例如：重新命名檔案。

```
tasks.register<Copy>("copyFromStaging") {
    from("src/main/kotlin/Main.kt")
    //使用閉包
    rename { filename: String ->
        fileName.toUpperCase()
    }
    //使用正規表達式
    rename("(.+)-staging-(.+)", "$1$2")
    rename("(.+)-staging-(.+)".toRegex().pattern, "$1$2")
    into("$buildDir/targetDir")
}
```

### 過濾檔案內容

在複製檔案的同時，可以過濾內容或轉換內容到另一個檔案。

過濾檔案內容的方法有以下方式

- CopySpec.filter()：印出每一行內容，並返回過濾後結果。

範例：過濾檔案的內容

```
tasks.register<Copy>("filter") {
    from("src/main/webapp")
    into("$buildDir/explodedWar")
    // Use a closure to remove lines
    filter { line: String ->
        if (line.startsWith('-')) null else line
    }
}
```

### 共用複製規格

重複定義一樣的Copy行爲，很容易在需要修改定義後，造成工作之間執行的內容的不一致。透過CopySpec.with()來將行為附加到合適的工作中。

範例：共用複製規格

```
val webAssetsSpec: CopySpec = copySpec {
    from("src/main/webapp")
    include("**/*.html", "**/*.png", "**/*.jpg")
    rename("(.+)-staging(.+)", "$1$2")
}

tasks.register<Copy>("copyAssets") {
    into("$buildDir/inPlaceApp")
    with(webAssetsSpec)
}

tasks.register<Zip>("distApp") {
    archiveFileName.set("my-app-dist.zip")
    destinationDirectory.set(file("$buildDir/dists"))

    from(appClasses)
    with(webAssetsSpec)
}
```

### 在任務中複製檔案

除了使用Copy的Task來複製檔案，還可以使用Project.copy()複製檔案，但這樣不會引入額外的Task工作。

範例：使用Project的Copy()。

```
tasks.register("copyMethod") {
    doLast {
        copy {
            from("src/main/webapp")
            into("$buildDir/explodedWar")
            include("**/*.html")
            include("**/*.jsp")
        }
    }
}
```
