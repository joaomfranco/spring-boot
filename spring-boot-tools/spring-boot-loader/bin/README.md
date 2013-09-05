# Spring Boot - Loader
The Spring Boot Loader module allows JAR and WAR files that contain nested dependencies
to be run using `java -jar archive.jar`.

> **Note:** The quickest way to build a compatible archive is to use the
> [spring-boot-maven-plugin](../spring-boot-maven-plugin) or
> [spring-boot-gradle-plugin](../spring-boot-gradle-plugin).

## Nested JARs
Java does not provide any standard way to load nested jar files (i.e. jar files that
are themselves contained within a jar). This can be problematic if you are looking
to distribute a self contained application that you can just run from the command line
without unpacking.

To solve this problem, many developers use 'shaded' jars. A shaded jar simply packages
all classes, from all jars, into a single 'uber jar'. The problem with shaded jars is
that it becomes hard to see which libraries you are actually using in your application.
It can also be problematic if the the same filename is used (but with different content)
in multiple jars. Spring Boot takes a different approach and allows you to actually nest
jars directly.

### JAR file structure
Spring Boot Loader compatible jar files should be structured in the following way:

```
example.jar
 |
 +-META-INF
 |  +-MANIFEST.MF
 +-org
 |  +-springframework
 |     +-boot
 |        +-loader
 |           +-<spring boot loader classes>
 +-com
 |  +-mycompany
 |     + project
 |        +-YouClasses.class
 +-lib
    +-dependency1.jar
    +-dependency2.jar
```

Dependencies should be placed in a nested `lib` directory.

See [executable-jar](src/it/executable-jar) for an example project.

### WAR file structure
Spring Boot Loader compatible war files should be structured in the following way:

```
example.jar
 |
 +-META-INF
 |  +-MANIFEST.MF
 +-org
 |  +-springframework
 |     +-boot
 |        +-loader
 |           +-<spring boot loader classes>
 +-WEB-INF
    +-classes
    |  +-com
    |     +-mycompany
    |        +-project
    |           +-YouClasses.class
    +-lib
    |  +-dependency1.jar
    |  +-dependency2.jar
    +-lib-provided
       +-servlet-api.jar
       +-dependency3.jar
```

Dependencies should be placed in a nested `WEB-INF/lib` directory. Any dependencies
that are required when running embedded but are not required when deploying to
a traditional web container should be placed in `WEB-INF/lib-provided`.

 See [executable-war](src/it/executable-war) for an example project.

## RandomAccessJarFile
The core class used to support loading nested jars is
`org.springframework.boot.loader.jar.RandomAccessJarFile`. It allows  you load jar
content from a standard jar file or from nested child jar data. When first  loaded, the
location of each  `JarEntry` is mapped to a physical file offset of the outer jar:


```
myapp.jar
+---------+---------------------+
|         | /lib/mylib.jar      |
| A.class |+---------+---------+|
|         || B.class | B.class ||
|         |+---------+---------+|
+---------+---------------------+
^          ^          ^
0063       3452       3980
```

The example above shows how `A.class` can be found in `myapp.jar` position `0063`.
`B.class` from the nested jar can actually be found in `myapp.jar` position `3452`
and `B.class` is at position `3980`.

Armed with this information, we can load specific nested entries by simply seeking to
appropriate part if the outer jar. We don't need to unpack the archive and we don't
need to read all entry data into memory.

### Compatibility
Spring Boot Loader strives to remain compatible with existing code and libraries. The
`RandomAccessJarFile` extends from `java.util.jar.JarFile` and should work as a drop-in
replacement. The `RandomAccessJarFile.getURL()` method will return a `URL` that opens
a `java.net.JarURLConnection` compatible connection. `RandomAccessJarFile` URLs can
be used with Java's `URLClassLoader`.

## Launching
The `org.springframework.boot.loader.Launcher` class can be used to run your packaged
application. It takes care of setting up an appropriate `URLClassLoader` and calling
your `main()` method.

### Launcher manifest
You need specify an appropriate `Launcher` as the `Main-Class` attribute of
`META-INF/MANIFEST.MF`. The actual class that you want to launch (i.e. the class that
you wrote that contains a `main` method) should be specified  in the `Start-Class`
attribute.

For example, here is a typical `MANIFEST.MF` for a executable jar file:
```
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.mycompany.project.MyApplication
```

For a war file, it would be:
```
Main-Class: org.springframework.boot.loader.WarLauncher
Start-Class: com.mycompany.project.MyApplication
```
> **Note:** You do not need to specify `Class-Path` entries in your manifest file, the
> classpath will be deduced from the nested jars.

### Exploded archives
Certain PaaS implementations may choose to unpack archives before they run. For example,
Cloud Foundry operates in this way. You can run an unpacked archive by simply starting
the appropriate launcher:

```
$ unzip -q myapp.jar
$ java org.springframework.boot.loader.JarLauncher
```

## Restrictions
There are a number of restrictions that you need to consider when working with a Spring
Boot Loader packaged application.

### URLs
URLs for nested jar entries intentionally look and behave like standard jar URLs,
You cannot, however, directly create a nested jar URL from a string:

```
URL url = classLoader.getResoure("/a/b.txt");
String s = url.toString(); // In the form 'jar:file:/file.jar!/nested.jar!/a/b.txt'
new URL(s); // This will fail
```

If you need to obtain URL using a String, ensure that you always provide a context URL
to the constructor. This will ensure that the custom `URLStreamHandler` used to support
nested jars is used.

```
URL url = classLoader.getResoure("/a");
new URL(url, "b.txt");
```

### Zip entry compression
The `ZipEntry` for a nested jar must be saved using the `ZipEntry.STORED` method. This
is required so that we can seek directly to individual content within the nested jar.
The content of the nested jar file itself can still be compressed, as can any other
entries in the outer jar. You can use the Spring Boot
[Maven](../spring-boot-maven-plugin) or [Gradle](../spring-boot-gradle-plugin) plugins
to ensure that your archives are written correctly.

### System ClassLoader
Launched applications should use `Thread.getContextClassLoader()` when loading classes
(most libraries and frameworks will do this by default). Trying to load nested jar
classes via `ClassLoader.getSystemClassLoader()` will fail. Please be aware that
`java.util.Logging` always uses the system classloader, for this reason you should
consider a different logging implementation.

### Alternatives
If the above restrictions mean that you cannot use Spring Boot Loader the following
alternatives could be considered:

* [Maven Shade Plugin](http://maven.apache.org/plugins/maven-shade-plugin/)
* [JarClassLoader](http://www.jdotsoft.com/JarClassLoader.php)
* [OneJar](http://one-jar.sourceforge.net)

## Further Reading
For more information about any of the classes or interfaces discussed in the document
please refer to the project Javadoc. If you need to build a compatible archives see the
[spring-boot-maven-plugin](../spring-boot-maven-plugin) or
[spring-boot-gradle-plugin](../spring-boot-gradle-plugin). If you are not using Maven
or Gradle [spring-boot-loader-tools](../spring-boot-loader-tools) provides some useful
utilities to rewite existing zip files.
