# Go Offline Maven Plugin
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/de.qaware.maven/go-offline-maven-plugin/badge.svg)](https://maven-badges.herokuapp.com/maven-central/de.qaware.maven/go-offline-maven-plugin)

Maven Plugin used to download all Dependencies and Plugins required in a Maven build,
so the build can be run without an internet connection afterwards.

This is especially relevant for modern CI-Systems like Gitlab and Circle-CI which
need a consistent local Maven repository in their cache to build efficiently.

The plugin can also be used to download all source files of all transitive dependencies
of a project.

Maven already has an official way to do all this: the maven-dependency-plugin go-offline goal;  
Unfortunately, the go-offline goal suffers from several drawbacks:

- Multi-Module builds are not supported since the plugin tries to download reactor-dependencies from the remote repository
- Most parameters simply do not work
- There is no option to download dynamic dependencies

The Go Offline Maven Plugin fixes these drawbacks.  

## Requirements
- Java 1.7 or higher
- Maven 3.1.x or higher

**Warning:** Maven 3.5.2 has a Bug that causes the Plugin to hang. If you experience hangups during downloading dependencies, please upgrade to Maven 3.5.3 or higher (See [MNG-6323](https://issues.apache.org/jira/browse/MNG-6323) )

**Warning:** Maven versions 3.2.x and 3.3.x have a bug that can cause the plugin to fail if maven is started in parallel build mode (-T option). Please call the go-offline-maven-plugin
without the -T option or upgrade Maven to a newer version. (See [MNG-6170](https://issues.apache.org/jira/browse/MNG-6170))  

## Goals
The Go Offline Maven Plugin only has one goal: "resolve-dependencies". This goal downloads
all external dependencies and plugins needed for your build to your local repository.
Dependencies that are built inside the reactor build of your project are excluded. For downloading,
the repositories specified in your pom.xml are used.

## Usage
Simply add the plugin to the pom.xml of your project. Use the root reactor pom in case of a multi module project.
Make sure to configure any dynamic dependency your project has (see below).

    <plugin>
        <groupId>de.qaware.maven</groupId>
        <artifactId>go-offline-maven-plugin</artifactId>
        <version>1.2.8</version>
        <configuration>
            <dynamicDependencies>
                <DynamicDependency>
                    <groupId>org.apache.maven.surefire</groupId>
                    <artifactId>surefire-junit4</artifactId>
                    <version>2.20.1</version>
                    <repositoryType>PLUGIN</repositoryType>
                </DynamicDependency>
                <DynamicDependency>
                    <groupId>com.querydsl</groupId>
                    <artifactId>querydsl-apt</artifactId>
                    <version>4.2.1</version>
                    <classifier>jpa</classifier>
                    <repositoryType>MAIN</repositoryType>
                </DynamicDependency>
                <DynamicDependency>
                    <groupId>org.flywaydb</groupId>
                    <artifactId>flyway-commandline</artifactId>
                    <version>4.0.3</version>
                    <type>zip</type>
                    <repositoryType>MAIN</repositoryType>
                </DynamicDependency>
            </dynamicDependencies>
        </configuration>
    </plugin>
    
To download all dependencies to your local repository, use
    
    mvn de.qaware.maven:go-offline-maven-plugin:resolve-dependencies

Make sure to activate any profiles etc. so that all relevant modules of your project are included
in the Maven run.

### Dynamic Dependencies
Unfortunately some Maven Plugins dynamically load additional dependencies when they are run. Since those
dependencies are not necessarily specified anywhere in the plugins pom.xml, the Go Offline Maven Plugin
cannot know that it has to download those dependencies. Most prominently, the surefire-maven-plugin dynamically
loads test-providers based on the tests it finds in the project.

You must tell the Go Offline Maven Plugin of those dynamic depenencies to ensure all dependencies are downloaded.
For each dependency, add a DynamicDependency block to the plugin's configuration as seen in the [Usage](#usage) section.
Each dynamic dependency block consists of the following parameters:

- *groupId* The GroupId of the dynamic dependency to download
- *artifactId* The ArtifactId of the dynamic dependency to download
- *version* The version of the dynamic dependency to download
- *classifier* (optional) The classifier of the dynamic dependency to download
- *type* (optional) The type of the dynamic dependency to download
- *repositoryType* Either 'MAIN' or 'PLUGIN' to control from which repository the dependency is downloaded

Note that Plugins are not consistent about where they pull their dynamic dependencies from. Some use the Plugin-Repository
, some the Main-Repository. If one doesn't work, try the other.

### Downloading Sources and Javadoc
The plugin can also download the source files and/or javadoc of the project's transitive dependencies. This behaviour can either be activated via the pom.xml
or a command line parameter.

    <plugin>
        <groupId>de.qaware.maven</groupId>
        <artifactId>go-offline-maven-plugin</artifactId>
        <version>1.2.8</version>
        <configuration>
           <downloadSources>true</downloadSources>
           <downloadJavadoc>true</downloadJavadoc>
        </configuration>
    </plugin>          
    
or

    mvn de.qaware.maven:go-offline-maven-plugin:resolve-dependencies -DdownloadSources -DdownloadJavadoc
    
### Usage in CI environments
The Go Offline Maven Plugin can be used to build a clean repository for build server environments. The resulting repository includes exactly the dependencies and
plugins needed for building the project.

#### Gitlab

For gitlab, add the following build step to the front of your pipeline:

    download-dependencies:
      image: maven:3-jdk-8
      stage: prepare
      script:
        - 'mvn de.qaware.maven:go-offline-maven-plugin:1.2.8:resolve-dependencies -Dmaven.repo.local=.m2/repository'
      cache:
        key: M2_REPO
        paths:
          - .m2/repository
          
This will fill the cache "M2_REPO" with all needed artifacts, reusing the previous "M2_REPO" cache to avoid downloading all artifacts on each run.

Build steps using the repository may then be configured like this:

    .build
      image: maven:3-jdk-8
      stage: build
      script:
        - 'mvn install -Dmaven.repo.local=.m2/repository'
      cache:
        key: M2_REPO
        paths:
          - .m2/repository
        policy: pull

This will pull the previously filled cache into the build image and use it as the local maven repository.
Policy: pull ensures that artifacts that are generated as part of the build are not written back to the cache
                                                                                                             .
    
## License

Apache 2.0 (https://www.apache.org/licenses/LICENSE-2.0.txt) 
    
