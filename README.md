# VS Code support for Java using the javac API

Provides Java support using the javac API.
Requires that you have Java 8 installed on your system.

## Installation

[Install from the VS Code marketplace](https://marketplace.visualstudio.com/items?itemName=georgewfraser.vscode-javac)

## [Issues](https://github.com/georgewfraser/vscode-javac/issues)

## Features

### Autocomplete

<img src="http://g.recordit.co/bCbYuegVRV.gif">

### Go-to-definition

<img src="http://g.recordit.co/Fg7cpH1rnz.gif">

### Find symbol

<img src="http://g.recordit.co/6g1bESw3Pp.gif">

### Lint

<img src="http://g.recordit.co/DGTYfpFdSD.gif">

### Type information on hover

<img src="http://g.recordit.co/R3T0nLUpZJ.gif">

### Find references

<img src="http://g.recordit.co/71PXpvFfs8.gif">

## Usage

## Using Maven pom.xml

vscode-javac will look for a file named pom.xml in a parent directory of every open .java file.
vscode-javac will invoke maven in order to get the build classpath, so you need to be sure `mvn` is on your system path.

## Using javaconfig.json

The presence of a `javaconfig.json` file indicates that 
its parent directory is the root of a Java module.
`javaconfig.json` looks like:

    {
        "sourcePath": ["relative/path/to/source/root", ...],
        "classPathFile": "file-with-classpath-as-contents.txt",
        "outputDirectory": "relative/path/to/output/root"
    }
    
The classpath is contained in a separate file, 
in the format `entry.jar:another-entry.jar`.
This file is usually generated by a build tool like maven.

### Examples

### Maven 

pom.xml will be read automatically.

### Maven (using javaconfig.json)

You can configure maven to output the current classpath to a file, 
classpath.txt, where Visual Studio Code will find it.

#### javaconfig.json

Set the source path, and get the class path from a file:

    {
        "sourcePath": ["src/main/java"],
        "classPathFile": "classpath.txt",
        "outputDirectory": "target"
    }

#### pom.xml

Configure maven to output `classpath.txt`

    <project ...>
        ...
        <build>
            ...
            <plugins>
                ...
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-dependency-plugin</artifactId>
                    <version>2.9</version>
                    <executions>
                        <execution>
                            <id>build-classpath</id>
                            <phase>generate-sources</phase>
                            <goals>
                                <goal>build-classpath</goal>
                            </goals>
                        </execution>
                    </executions>
                    <configuration>
                        <outputFile>classpath.txt</outputFile>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </project>

#### .gitignore

Ignore `classpath.txt`, since it will be different on every host

    classpath.txt
    ...

### Gradle

Add this to your `build.gradle`:

```gradle
task vscodeClasspathFile {
    description 'Generates classpath file for the Visual Studio Code java plugin'
    ext.destFile = file("$buildDir/classpath.txt")
    outputs.file destFile
    doLast {
        def classpathString = configurations.compile.collect{ it.absolutePath }.join(File.pathSeparator)
        if (!destFile.parentFile.exists()) {
            destFile.parentFile.mkdirs()
        }
        assert destFile.parentFile.exists()
        destFile.text = classpathString
    }
}

task vscodeJavaconfigFile(dependsOn: vscodeClasspathFile) {
    description 'Generates javaconfig.json file for the Visual Studio Code java plugin'

    def relativePath = { File f ->
        f.absolutePath - "${project.rootDir.absolutePath}/"
    }
    ext.destFile = file("javaconfig.json")
    ext.config = [
        sourcePath: sourceSets.collect{ it.java.srcDirs }.flatten().collect{ relativePath(it) },
        classPathFile: relativePath(tasks.getByPath('vscodeClasspathFile').outputs.files.singleFile),
        outputDirectory: relativePath(new File(buildDir, 'vscode-classes'))
    ]
    doLast {
        def jsonContent = groovy.json.JsonOutput.toJson(ext.config)
        destFile.text = groovy.json.JsonOutput.prettyPrint(jsonContent)
    }
}

task vscode(dependsOn: vscodeJavaconfigFile) {
    description 'Generates config files for the Visual Studio Code java plugin'
    group 'vscode'
}
```

Then run `gradlew vscode`. This will generate
* `javaconfig.json`
* `build/classpath.txt`

### Gradle Android build

For Android gradle project, put the above tasks in the `android` method of your `build.gradle`:
```gradle
android {
    ...
    // add the vscode tasks inside the android method
    task vscodeClasspathFile {
    ...    
}
```

Currently, the generated `classpath.txt` does not contain android platform library, e.g., `/opt/android-sdk-linux/platforms/android-23/android.jar`. You would need to add it manually. See issue #23.

### SBT (Lightbend Activator)

Add this to your `build.sbt` file to generate `classpath.txt` every time you execute the `compile` or `run` tasks. This has been tested with Lightbend Activator/sbt 0.13.11.

```scala
val genClasspath = taskKey[String]("Generate classpath.txt for use with VS Code.")

genClasspath := {
  val log = streams.value.log
  val cp: Seq[File] = (dependencyClasspath in Compile).value.files
 
  val file = baseDirectory.value / "classpath.txt"
  log.info("Writing classpath details to: " + file.getAbsolutePath)
  IO.write(file, cp.map(_.getAbsolutePath).mkString(java.io.File.pathSeparator))
  file.getAbsolutePath
}

// Generate classpath.txt every time you compile or run the project
compile in Compile <<= (compile in Compile).dependsOn(genClasspath)
run in Compile <<= (run in Compile).dependsOn(genClasspath)

``` 

#### javaconfig.json

Add this file and configure `sourcePath` and `outputDirectory` as needed:
```json
{
    "sourcePath": ["app"],
    "classPathFile": "classpath.txt",
    "outputDirectory": "target"
}
```

#### .gitignore

Ignore `classpath.txt`, since it will be different on every host

    classpath.txt
    ...

## Directory structure

### Java service process

A java process that does the hard work of parsing and analyzing .java source files.

    pom.xml (maven project file)
    src/ (java sources)
    repo/ (tools.jar packaged in a local maven repo)
    target/ (compiled java .class files, .jar archives)
    target/fat-jar.jar (single jar that needs to be distributed with extension)

### Typescript Visual Studio Code extension

"Glue code" that launches the external java process
and connects to it using [vscode-languageclient](https://www.npmjs.com/package/vscode-languageclient).

    package.json (node package file)
    tsconfig.json (typescript compilation configuration file)
    tsd.json (project file for tsd, a type definitions manager)
    lib/ (typescript sources)
    out/ (compiled javascript)

## Design

This extension consists of an external java process, 
which communicates with vscode using the [language server protocol](https://github.com/Microsoft/vscode-languageserver-protocol). 

### Java service process

The java service process uses the implementation of the Java compiler in tools.jar, 
which is a part of the JDK.
When VS Code needs to lint a file, perform autocomplete, 
or some other task that requires Java code insight,
the java service process invokes the Java compiler programatically,
then intercepts the data structures the Java compiler uses to represent source trees and types.

### Incremental updates

The Java compiler isn't designed for incremental parsing and analysis.
However, it is *extremely* fast, so recompiling a single file gives good performance,
as long as we don't also recompile all of its dependencies.
We accomplish this by maintaining a single copy of the Java compiler in memory at all times.
When we want to recompile a file, 
we clear that *one* file from the internal caches of the Java compiler,
and then rerun the compiler.

### Multiple javaconfig.json

If you have multiple javaconfig.json files in different subdirectories of your project,
the parent directory of each javaconfig.json will be treated as a separate java root.

## Logs

The java service process will output a log file with a name like 'javac-services.0.log'
in your project directory.

## Contributing

If you have npm and maven installed,
you should be able to install locally using 

    npm install -g vsce
    npm install
    ./scripts/install.sh