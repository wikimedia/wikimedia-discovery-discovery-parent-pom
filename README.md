# Discovery parent POM

This project provides a common configuration for Maven based projects of the
Wikimedia Discovery team. If factorize common configuration and build process.
This allow to have more coherent projects and reuse the efforts done in
configuring various build plugins.

## What does this pom contains

* Fixed versions of all plugins.
* A small set of dependencies that should make sense in all projects. This is
  provided as a <dependencyManagement/> section, so you don't have to use them.
* Configuration of some static analysis and quality tools.
* Standardization on UTF-8 and Java 8.

## How to use

Have your project inherit from this pom. Override the settings which do not
make sense for your project. At least the <scm/> section should be overridden.

## More details

This is a non exhaustive list of things that are configured in this pom. Read
the pom itself if you want all the details.

### Distribution Management

By default, deployments are done to WMF archiva repository. To deploy to Maven
Central instead, use the `deploy-central` profile:

    ./mvnw deploy -P deploy-central

or via a release:

    ./mvnw -B -P deploy-central release:prepare && ./mvnw -B -P deploy-central release:perform

### Standard Properties

* `maven.compiler.source=1.8`
* `maven.compiler.target=1.8`
* `project.build.sourceEncoding=UTF-8`
* `project.reporting.outputEncoding=UTF-8`

These properties can be overridden in child projects, but there should be no
reason to do so.

### Dependency Management

Some standard dependencies are provided. Note that `<dependencyManagement/>`
does not add those dependencies to the classpath of child projects, it only
fixes the versions. The provided dependencies are either test dependencies or
annotations that are used by static analysis tools. They are provided as an
indication that those are interesting libraries, that you should probably
discover if you don't already know them.

#### Lombok

If you decide to use [Lombok](http://jnb.ociweb.com/jnb/jnbJan2010.html) you
should probably add a `lombok.config` file in your main package. See the
[docs](https://projectlombok.org/features/configuration)
if you want more infos.

        lombok.equalsAndHashCode.callSuper = call
        lombok.extern.findbugs.addSuppressFBWarnings = true
        lombok.addLombokGeneratedAnnotation = true

### Jar with dependencies / ueberjar

In some cases, it makes sense to publish an artifact that includes project code
and dependencies. For example to publish a runnable jar, or a single artifact
executed in specific context. In most cases, the [maven-assembly-plugin with
the "jar-with-dependencies"](https://maven.apache.org/plugins/maven-assembly-plugin/descriptor-refs.html#jar-with-dependencies)
is the simplest solution. The [maven-shade-plugin](http://maven.apache.org/plugins/maven-shade-plugin/)
can be used for more complex use cases, for example when you require merging
properties files or other metadata.

### plugins

#### sortpom-maven-plugin

Ensure the pom is sorted in the recommended order. Having a stable order
ensures that diffs in code reviews are not subject to reordering noise. Having
a standard order makes it slightly easier to find your way in a pom.

To sort an existing pom: `mvn sortpom:sort`.

#### spotbugs-maven-plugin

[Spotbugs](https://spotbugs.github.io/) is the successor to Findbugs. It is a
static analyzer which finds some interesting bugs and code smells. If you use
JSR305 annotations, it can do additional checks. Rules can be ignored with the
`@SupressFBWarnings(value="RULE_ID", justification="Why you don't care")`
annotation.

#### forbiddenapis

Checks for methods you should never use. For example deprecated methods,
methods that have known performance issues, ...

This check can be ignored by the `@SuppressForbidden` annotation.

#### maven-checkstyle-plugin

The usual Java linter. Mostly about style, but some rules help discover actual
bugs. Rules chan be ignored with `@SupressWarnings("checkstyle:NameOfRule")`.

Checkstyle is configured in the [discovery-maven-tool-configs
project](https://github.com/wikimedia/wikimedia-discovery-discovery-maven-tool-configs/tree/master/src/main/resources/org/wikimedia/discovery/build/tools/checkstyle).

A configuration for IntelliJ which follows those rules is available
on [Maven Central](http://central.maven.org/maven2/org/wikimedia/discovery/discovery-maven-tool-configs/).
It can be imported in IntelliJ via `File -> Import Settings...`.

#### spotless-maven-plugin

The spotless-maven-plugin formats code. It is configured to format java code to comply with the
[Google Java Style](https://google.github.io/styleguide/javaguide.html).
Furthermore, it is configured to…

* …use 4 spaces for indentation (instead of 2)
* …organize imports to comply with check-style rules (no unused, sorted, grouped)

#### maven-enforcer-plugin

The maven-enforcer-plugin can enforce a number of rules. At the moment, we
enforce:

* a minimal Maven version
* no package cycles are allowed

While it is a really good idea to not have cyclic dependencies between packages
for the usual modularity reasons, those tend to creep into existing code
whenever you turn your back on it. To disable the plugin while you fix those,
the following can be added to the child pom, in the `<plugins>` section:

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <executions>
          <execution>
                <id>enforce-no-package-cycles</id>
                <!-- disable enforcer as this project has package cyclic dependencies -->
                <phase>none</phase>
          </execution>
    </executions>
    </plugin>
<plugin>
```

#### duplicate-finder-maven-plugin

Fail if multiple versions of the same class exist on the classpath (jar hell).
You can reconfigure the plugin in a child pom to exclude some dependencies from
analysis:

```
<plugin>
    <groupId>org.basepom.maven</groupId>
    <artifactId>duplicate-finder-maven-plugin</artifactId>
    <configuration>
        <ignoredDependencies>
            <!--
                the following dependencies are problematic but non trivial to
                clean, it will come in a second time...
            -->
            <dependency>
                <groupId>org.my.group.id</groupId>
                <artifactId>artifact-id</artifactId>
            </dependency>
        </ignoredDependencies>
    </configuration>
</plugin>
```

##### How to fix duplicates:
There can be various reasons why classes or files are duplicated. Some scenarios and their solutions are provided below:
- Two dependencies can both rely on a third dependency but of different versions. The third dependency could be named differently although they are the same, for example maybe if the project's ownership changed or the project's naming convention changed. In this case, `groupId` would be different but the `artifactId` would be the same (asm:asm, org.ow2.asm:asm). Try to keep the latest dependency and exclude the older one. Search for both dependencies in [search.maven.org](https://search.maven.org/) and check the last updated date. Whichever one is older, exclude it.
  - **How to exclude a dependency?** First find who is using the old dependency (spark_core in the example below). Run `./mvnw dependency:tree` or `./mvnw --projects <sub-module-name> dependency:tree`. Search by the name of the dependency you want to exclude (objenesis in the example below). Once you find who uses it, go to the pom.xml file, under the dependency you just found (spark_core here), do this:
    ```
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <scope>provided</scope>
        <exclusions>
            <exclusion>
                <groupId>org.objenesis</groupId>
                <artifactId>objenesis</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    ```
- Sometimes a project can copy-paste its dependencies into its root. This does not show up as dependency in the dependency-tree but look at the folder strcuture of the the projects that are causing duplication errors, you may find folders that seem to not belong there. Example: projectX causes duplication error with projectY. The files that are listed as duplicated seem to belong to projectX (e.g com.projectX.fileName). Looking at the folder strcuture of projectY, you see a folder called projectX. So now you can exclude projectX from your pom.xml using the method described above.
- Sometimes only one file is duplicated in multiple projects. Those could be standard files and their patterns can be ignored. See [Ignoring Dependencies and Resources.md](https://github.com/basepom/duplicate-finder-maven-plugin/blob/master/docs/Ignoring%20Dependencies%20and%20Resources.md) for more. Example:
  ```
  <plugin>
      <groupId>org.basepom.maven</groupId>
      <artifactId>duplicate-finder-maven-plugin</artifactId>
      <configuration>
          <ignoredResourcePatterns>
              <ignoredResourcePattern>nameOfYourFile_1</ignoredResourcePattern>
              <ignoredResourcePattern>nameOfYourFile_2</ignoredResourcePattern>
          </ignoredResourcePatterns>
      </configuration>
  </plugin>
  ```
  
#### jacoco-maven-plugin

Calculates the coverage of unit tests. Coverage of integration tests is not
tracked as usually this isn't a metric which makes much sense. It is possible
to make the build fail if a minimal coverage is not met, but this isn't active
at the moment. Feel free to enable such check in your child project.

#### git-commit-id-plugin

Create a git.properties file containing information on the current git commit.
This can be used either to track where a jar comes from, or actively in your
application, parsing it at runtime.

#### pitest

[Pitest](http://pitest.org/) is a mutation testing framework for Java. It is
not enabled by default as mutation testing can be both expensive and
problematic depending on the code under test.

To activate it on a specific module you can add it to the pom.xml of a module
in the build/plugins section. This will bind pitest to the `test` phase of the
build.

    <plugin>
        <groupId>org.pitest</groupId>
        <artifactId>pitest-maven</artifactId>
    </plugin>

An alternative is to run pitest only on demand. In this case, no change to the
pom.xml is required, and pitest can be invoked with:

    ./mvnw org.pitest:pitest-maven:mutationCoverage

Also note that pitest can run only on code that has been changed in the last
commit:

    ./mvnw org.pitest:pitest-maven:scmMutationCoverage

Some test frameworks (randomizedtesting for example) require assertions to be
enabled. Additional configuration for pitest to enable assertions:

    <configuration>
        <jvmArgs>
            <value>-ea</value>
        </jvmArgs>
    </configuration>

By default, pitest only enables stable mutators that don't generate too many
equivalent mutations. To enable all mutators, add the following config:

    <configuration>
        <mutators>
            <mutator>ALL</mutator>
        </mutators>
    </configuration>

Or list [individual mutators](http://pitest.org/quickstart/mutators/) that you
want to use.

To add mutation testing reports to the maven site, add the following section to
the reporting/plugins section of the modules tested:

    <plugin>
        <groupId>org.pitest</groupId>
        <artifactId>pitest-maven</artifactId>
        <version>LATEST</version>
        <reportSets>
            <reportSet>
                <reports>
                    <report>report</report>
                </reports>
            </reportSet>
        </reportSets>
    </plugin>

#### SonarQube

A default configuration of the sonar-maven-plugin is provided. It sets
properties that should be shared by all children projects:

* sonar.host.url: https://sonarcloud.io
* sonar.login: ${env.SONAR_API_KEY}
* sonar.organization: wmftest

Note that the SONAR_API_KEY is taken from an environment variable that needs to
be defined outside of Maven.

To publish to sonarcloud, run:

    SONAR_API_KEY=your_api_key ./mvnw sonar:sonar

#### Deadcode4j

[Deadcode4j](https://github.com/Scout24/deadcode4j) helps you find code that
is no longer used by your application. It isn't bound to any lifecycle phase,
but can be run manually with:

    ./mvnw deadcode4j:find -Dmaven.test.skip=true

To skip the packaging phase (if the project is already packaged, in case of
repeated analysis), you can use:

    ./mvnw deadcode4j:find.only

#### Standard Plugins

The version of all standard Maven plugins are defined, so that they don't
change unexpectedly.

#### Dependency plugin analysis
Dependency plugin allows analysis of specified dependencies - which dependencies are in
use and not explicitly defined, and which aren't in use but defined. While useful, by
default all dependencies that are used outside of the code (like the servlet container for
web.xml) will be reported as unused. To mitigate this, you can use the following
configuration to ignore non-compile dependencies from analysis:
```
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <configuration>
            <ignoreNonCompile>true</ignoreNonCompile>
        </configuration>
    </plugin>
```
Before doing that - please make sure dependencies reported are actually used.

### Reports

A number of reports are configured. Those will be generated as part of the
maven site. Use `mvn site:site` to generate the site. In a multi module
project, use `mvn site:site site:stage` to aggregate the sites of each modules.

#### Vulnerability check

The [dependency-check-maven plugin](https://jeremylong.github.io/DependencyCheck/dependency-check-maven/)
allows to check your project against known vulnerabilities. Since this process
is somewhat expansive, it is not enabled by default. It can be run manually
with:

    ./mvnw dependency-check:check

The report will be available for each module in `target/dependency-check-report.html`.

## Building

### Maven Wrapper

This project bundles [Maven Wrapper](https://github.com/takari/maven-wrapper).
This means that you don't need to install Maven, just use `./mvnw` instead of
the `mvn` command. Maven Wrapper help ensure all developer use the same version
of Maven.

### Dependencies

Some of the configuration of the static analysis tools are bundled in a
separate project (discovery-maven-tool-configs). When a change is made to those
configs, the following must be done:

* release a new version of discovery-maven-tool-config
* update the `discovery-maven-tool-configs.version` property in the
  discovery-parent-pom project
* release a new version of discovery-parent-pom

### Deploy a SNAPSHOT

To deploy a new SNAPSHOT to Sonatype OSS SNAPSHOT repository, just run:
`./mvnw deploy`. This requires your Sonatype credentials to be configured in
your `settings.xml`.

### Release

As part of the release, we upload artifacts to WFM Archiva repository or to
Sonatype OSS (Maven Central). This requires signing the artifacts. So in
addition to the SNAPSHOT deployment prerequisites, you will need a GPG key. You
will be prompted for the passphrase during the `release:perform` phase.

To release to WMF Archiva:

* `./mvnw -B release:clean`
* `./mvnw -B release:prepare`
* `./mvnw -B release:perform`

To release to Maven Central:

* `./mvnw -B -P deploy-central release:clean`
* `./mvnw -B -P deploy-central release:prepare`
* `./mvnw -B -P deploy-central release:perform`
