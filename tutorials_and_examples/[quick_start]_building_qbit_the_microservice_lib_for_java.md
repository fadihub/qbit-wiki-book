# [Quick Start] Building QBit the microservice lib for Java


## Building QBit.

This Wiki will walk through the process of building the QBit framework on your machine.


##What you will build

You will build this very powerful and fast framework called QBit using gradle.
##How to complete this guide

In order to complete this guide successfully you will need the following installed on your machine:

- Gradle; if you need help installing it, visit [Installing Gradle. ](http://www.gradle.org/docs/current/userguide/installation.html)
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.

Now that your machine is all ready let's get started:
- [Download QBit ](https://github.com/fadihub/qbit/archive/master.zip) and unzip the source repository for this guide, or clone it using Git with your terminal:

```bash
git clone https://github.com/fadihub/qbit.git
```

cd into `~/qbit` then run the following:
```bash
gradle clean build install
```
This will take a few minutes especially if you are using gradle for the first time.

####build.gradle Listing

`~/qbit/build.gradle/`
```gradle
allprojects {

    group = 'io.advantageous.qbit'
    apply plugin: 'idea'
    apply plugin: 'java'
    apply plugin: 'maven'
    apply plugin: 'application'

    version = '0.5.2-SNAPSHOT'
}


subprojects {

    repositories {
        mavenLocal()
        mavenCentral()
    }

    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8


    task buildDockerfile (type: Dockerfile) {
        dependsOn distTar
        from "java:openjdk-8"
        add "$distTar.archivePath", "/"
        workdir "/$distTar.archivePath.name" - ".$distTar.extension" + "/bin"
        entrypoint "./$project.name"
    }

    task buildDockerImage (type: Exec) {
        dependsOn buildDockerfile
        commandLine "docker", "build", "-t", "$project.group/$project.name:$version", buildDockerfile.dockerDir
    }


    task runDockerImage (type: Exec) {
        dependsOn buildDockerImage
        commandLine "docker", "run", "-t", "$project.group/$project.name:$version", buildDockerfile.dockerDir
    }


    task dockerRun (type: Exec) {
        commandLine "docker", "run", '-m="4g"', "-t", "$project.group/$project.name:$version", buildDockerfile.dockerDir
    }
}

project(':qbit-core') {

    apply plugin: 'signing'

    dependencies {
        testCompile group: 'junit', name: 'junit', version: '4.10'
        compile group: 'io.fastjson', name: 'boon', version: '0.31-SNAPSHOT'
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        testCompile 'ch.qos.logback:logback-classic:1.1.2'
    }

    task javadocJar(type: Jar, dependsOn: javadoc) {
        classifier = 'javadoc'
        from 'build/docs/javadoc'
    }

    task sourcesJar(type: Jar) {
        from sourceSets.main.allSource
        classifier = 'sources'
    }

    artifacts {
        archives jar
        archives javadocJar
        archives sourcesJar
    }

    signing {
        required false
        sign configurations.archives
    }

    uploadArchives {
        repositories {
            mavenDeployer {
                beforeDeployment { MavenDeployment deployment -> signing.signPom(deployment) }

                repository(url: "https://oss.sonatype.org/service/local/staging/deploy/maven2/") {
                    try {
                        authentication(userName: sonatypeUsername, password: sonatypePassword)
                    } catch (MissingPropertyException ignore) {
                    }
                }

                pom.project {
                    name 'qbit-core'
                    packaging 'jar'
                    description 'Go Channels inspired Java lib'
                    url 'https://github.com/advantageous/qbit'

                    scm {
                        url 'scm:git@github.com:advantageous/qbit.git'
                        connection 'scm:git@github.com:advantageous/qbit.git'
                        developerConnection 'scm:git@github.com:advantageous/qbit.git'
                    }

                    licenses {
                        license {
                            name 'The Apache Software License, Version 2.0'
                            url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                            distribution 'repo'
                        }
                    }

                    developers {
                        developer {
                            id 'richardHightower'
                            name 'Richard Hightower'
                        }
                        developer {
                            id 'sailorgeoffrey'
                            name 'Geoffrey Chandler'
                        }
                    }
                }
            }
        }
    }
}

project(':qbit-boon') {

    apply plugin: 'signing'

    sourceSets.main.resources.srcDir 'src/main/java'
    dependencies {
        compile project(':qbit-core')
        compile group: 'io.fastjson', name: 'boon', version: '0.31-SNAPSHOT'
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        testCompile 'ch.qos.logback:logback-classic:1.1.2'
        testCompile group: 'junit', name: 'junit', version: '4.10'
    }

    task javadocJar(type: Jar, dependsOn: javadoc) {
        classifier = 'javadoc'
        from 'build/docs/javadoc'
    }

    task sourcesJar(type: Jar) {
        from sourceSets.main.allSource
        classifier = 'sources'
    }

    artifacts {
        archives jar
        archives javadocJar
        archives sourcesJar
    }

    signing {
        required false
        sign configurations.archives
    }

    uploadArchives {
        repositories {
            mavenDeployer {
                beforeDeployment { MavenDeployment deployment -> signing.signPom(deployment) }

                repository(url: "https://oss.sonatype.org/service/local/staging/deploy/maven2/") {
                    try {
                        authentication(userName: sonatypeUsername, password: sonatypePassword)
                    } catch (MissingPropertyException ignore) {
                    }
                }

                pom.project {
                    name 'qbit-boon'
                    packaging 'jar'
                    description 'Go Channels inspired Java lib'
                    url 'https://github.com/advantageous/qbit'

                    scm {
                        url 'scm:git@github.com:advantageous/qbit.git'
                        connection 'scm:git@github.com:advantageous/qbit.git'
                        developerConnection 'scm:git@github.com:advantageous/qbit.git'
                    }

                    licenses {
                        license {
                            name 'The Apache Software License, Version 2.0'
                            url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                            distribution 'repo'
                        }
                    }

                    developers {
                        developer {
                            id 'richardHightower'
                            name 'Richard Hightower'
                        }
                        developer {
                            id 'sailorgeoffrey'
                            name 'Geoffrey Chandler'
                        }
                    }
                }
            }
        }
    }
}

project(':qbit-examples-standalone') {
    sourceSets.main.resources.srcDir 'src/main/java'
    dependencies {
        compile project(':qbit-vertx')
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        compile 'ch.qos.logback:logback-classic:1.1.2'

        testCompile group: 'junit', name: 'junit', version: '4.10'
    }
}


project(':qbit-perf-client') {
    sourceSets.main.resources.srcDir 'src/main/java'
    apply plugin:'application'

    /*
    You can run this by doing this: gradle run -PappArgs="['localhost', 8080]"
     */

    mainClassName = "io.advantageous.qbit.vertx.http.PerfClientTest"

    run {
        if ( project.hasProperty("appArgs") ) {
            args Eval.me(appArgs)
        }
    }


    dependencies {
        compile project(':qbit-vertx')
        compile project(':qbit-boon')
        compile group: 'io.fastjson', name: 'boon', version: '0.31-SNAPSHOT'
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        compile 'ch.qos.logback:logback-classic:1.1.2'

        testCompile group: 'junit', name: 'junit', version: '4.10'
    }

}


project(':qbit-perf-server') {
    sourceSets.main.resources.srcDir 'src/main/java'
    apply plugin:'application'

    /*
    You can run this by doing this: gradle run -PappArgs="['localhost', 8080]"
     */

    mainClassName = "io.advantageous.qbit.vertx.http.PerfClientTest"

    run {
        if ( project.hasProperty("appArgs") ) {
            args Eval.me(appArgs)
        }
    }

    mainClassName = "io.advantageous.qbit.vertx.http.PerfServerTest"

    dependencies {
        compile project(':qbit-vertx')

        compile project(':qbit-boon')
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        compile 'ch.qos.logback:logback-classic:1.1.2'

        testCompile group: 'junit', name: 'junit', version: '4.10'
    }

}




project(':qbit-sample-server') {

    apply plugin:'application'


    sourceSets.main.resources.srcDir 'src/main/java'

    mainClassName = "io.advantageous.qbit.sample.server.TodoServerMain"

    dependencies {
        compile project(':qbit-boon')
        compile project(':qbit-sample-model')
        compile project(':qbit-vertx')
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        compile 'ch.qos.logback:logback-classic:1.1.2'

        testCompile group: 'junit', name: 'junit', version: '4.10'
    }


    buildDockerfile {
        expose 8080
    }
}

project(':qbit-sample-client') {
    sourceSets.main.resources.srcDir 'src/main/java'
    apply plugin:'application'

    mainClassName = "io.advantageous.qbit.sample.server.TodoClientMain"

    dependencies {
        compile project(':qbit-boon')
        compile project(':qbit-sample-model')
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        compile 'ch.qos.logback:logback-classic:1.1.2'

        testCompile group: 'junit', name: 'junit', version: '4.10'
    }


    buildDockerfile {
        expose 7070
    }
}


project(':qbit-sample-model') {
    sourceSets.main.resources.srcDir 'src/main/java'
    dependencies {
        compile project(':qbit-boon')
        compile project(':qbit-vertx')
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        compile 'ch.qos.logback:logback-classic:1.1.2'
        testCompile group: 'junit', name: 'junit', version: '4.10'
    }
}

project(':qbit-vertx') {

    apply plugin: 'signing'

    task javadocJar(type: Jar, dependsOn: javadoc) {
        classifier = 'javadoc'
        from 'build/docs/javadoc'
    }

    task sourcesJar(type: Jar) {
        from sourceSets.main.allSource
        classifier = 'sources'
    }

    artifacts {
        archives jar
        archives javadocJar
        archives sourcesJar
    }

    signing {
        required false
        sign configurations.archives
    }

    jar {
        manifest {
            attributes 'Main-Class': 'io.rd.cognizance.example.SampleService'
        }
        from { configurations.compile.collect { it.isDirectory() ? it : zipTree(it) } }
    }
    sourceSets.main.resources.srcDir 'src/main/java'
    dependencies {
        compile project(':qbit-core')
        compile project(':qbit-boon')
        compile group: 'io.vertx', name: 'vertx-core', version: vertxVersion
        compile group: 'io.vertx', name: 'vertx-platform', version: vertxVersion
        compile group: 'io.vertx', name: 'lang-groovy', version: vertxVersion
        compile "org.slf4j:slf4j-api:[1.7,1.8)"
        testCompile 'ch.qos.logback:logback-classic:1.1.2'

        testCompile group: 'junit', name: 'junit', version: '4.10'

    }

    uploadArchives {
        repositories {
            mavenDeployer {
                beforeDeployment { MavenDeployment deployment -> signing.signPom(deployment) }

                repository(url: "https://oss.sonatype.org/service/local/staging/deploy/maven2/") {
                    try {
                        authentication(userName: sonatypeUsername, password: sonatypePassword)
                    } catch (MissingPropertyException ignore) {
                    }
                }

                pom.project {
                    name 'qbit-vertx'
                    packaging 'jar'
                    description 'Go Channels inspired Java lib'
                    url 'https://github.com/advantageous/qbit'

                    scm {
                        url 'scm:git@github.com:advantageous/qbit.git'
                        connection 'scm:git@github.com:advantageous/qbit.git'
                        developerConnection 'scm:git@github.com:advantageous/qbit.git'
                    }

                    licenses {
                        license {
                            name 'The Apache Software License, Version 2.0'
                            url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                            distribution 'repo'
                        }
                    }

                    developers {
                        developer {
                            id 'richardHightower'
                            name 'Richard Hightower'
                        }
                        developer {
                            id 'sailorgeoffrey'
                            name 'Geoffrey Chandler'
                        }
                    }
                }
            }
        }
    }
}




class Dockerfile extends DefaultTask {
    def dockerfileInfo = ""
    def dockerDir = "$project.buildDir/docker"
    def dockerfileDestination = "$project.buildDir/docker/Dockerfile"
    def filesToCopy = []

    File getDockerfileDestination() {
        project.file(dockerfileDestination)
    }

    def from(image="java") {
        dockerfileInfo += "FROM $image\r\n"
    }

    def maintainer(contact) {
        maintainer += "MAINTAINER $contact\r\n"
    }

    def add(sourceLocation, targetLocation) {
        filesToCopy << sourceLocation
        def file = project.file(sourceLocation)
        dockerfileInfo += "ADD $file.name ${targetLocation}\r\n"
    }

    def run(command) {
        dockerfileInfo += "RUN $command\r\n"
    }

    def env(var, value) {
        dockerfileInfo += "ENV $var $value\r\n"
    }

    def expose(port) {
        dockerfileInfo += "EXPOSE $port\r\n"
    }

    def workdir(dir) {
        dockerfileInfo += "WORKDIR $dir\r\n"
    }

    def cmd(command) {
        dockerfileInfo += "CMD $command\r\n"
    }

    def entrypoint(command) {
        dockerfileInfo += "ENTRYPOINT $command\r\n"
    }

    @TaskAction
    def writeDockerfile() {
        for (fileName in filesToCopy) {
            def source = project.file(fileName)
            def target = project.file("$dockerDir/$source.name")
            target.parentFile.mkdirs()
            target << source.bytes
        }
        def file = getDockerfileDestination()
        file.parentFile.mkdirs()
        file.write dockerfileInfo
    }
}

```
This is where all the plugins and dependencies are located.

Congrats now you have QBit installed on your machine!
