# gradle-to-ant

This project implements gradle conventions in ant. This is done by adding 
[extension-points](https://ant.apache.org/manual/targets.html#extension-points) which represent 
the typical gradle build phases for the `base` and `java` plugins in gradle. There is also support for ant sources 
and ant tests which is how this project is developed. The ant files can be 
[imported](https://ant.apache.org/manual/Tasks/import.html) in your own ant project to implement these conventions.

# Ant files

In gradle plugins are appied which provide support for typical tasks within a project. To do this in ant we use 
[import](https://ant.apache.org/manual/Tasks/import.html).

## base.xml

The first import which adds gradle conventions to the build. Base can be extended by adding target to the 
extension points: assemble, build, check, clean, cleanCaches, configure, init, and test.



```
Buildfile: gradle-to-ant/src/main/ant/base.xml

        Provides base properties and targets which are useful to any build.

Main targets:

 assemble            Extension point for assembling packages.
 build               Builds everything.
 check               Extension point for all checks including tests. Extensions should add checks to the build.
 clean               Entry point for cleaning the build. Extensions should delete directories.
 cleanAll            Cleans everything by calling clean and cleanCaches.
 cleanBuild          Deletes build directory.
 cleanBuildCache     Deletes build cache.
 cleanCaches         Entry point for cleaning all caches. Extensions should delete caches
 cleanIvyCache       Cleans ivy cache.
 configure           Extension point for configuring the build and future targets. Extntensions should create properties, condition properties, and add third party tasks.
 configureBuild      Resolves build configuration and creates build classpath.
 init                Entry point for initializing the build. Extensions should create directories, set properties for targets, and create condition properties for running configure targets.
 initBuild           Creates build directory.
 initConfigureBuild  Sets run.configureBuild if ivy.xml is present.
 installIvy          Adds ivy jar to build cache and adds ivy tasks to project.
 test                Extension point for running tests. Should execute all tests. Extensions should run specific types of tests.
```

## ant.xml

# Conventions


## project directory is project name

In gradle the project name defaults to the directory of the project. This can be changed in settings.gradle
but it is a good convention to follow in ant. The project name in ant can be set dynamically by removing it 
from the project element and defining the `ant.project.name` property. In base.xml:

```
<project name="gradle-to-ant">

</project>
```

Is changed to:

```
<project>
    <basename property="ant.project.name" file="${basedir}"/>
</project>
```

## build directory contains everything built

In gradle everything built goes into the build directory. base.xml provides a build directory for ant projects. 
This makes cleaning the build very easy.

## build cache

base.xml uses a build cache to cache downloads and possibly other files needed to last beyond a clean. 
Currently the build cache is used to store the ivy download. It can also be used by extensions during the init 
and configure phases of the build.

## clean deletes everything built

Simply delete the build directory. This does not delete the build cache.

## dependency management with ivy

Gradle uses ivy concepts for dependency management. Ivy is a dependency management system for ant. 
base.xml adds ivy support out of the box.

## build dependencies

Gradle is able to add new tasks and plugins at runtime. base.xml will resolve and cache the build configuration 
using ivy. Implementing projects can extend the configure phase to add tasks downloaded by the build 
configuration. This is done in ant.xml.

## testing the build

Gradle provides classes which can be used in buildSrc tests that test an actual build. This is the reason for 
ant.xml. It provides antunit tasks which can be used to test an ant build.

# ivy

Gradle projects have methods to configure repositories, configurations, and dependencies. While ivy has these
concepts the configuration has less conventions than gradle.

## Why ivy is installed in base.xml

In order to use ivy in ant it needs to be installed. base.xml will handle installing ivy from maven central so 
it doesn't need to be installed in ant manually.

There are two static locations ivy can be installed for use in ant. The first is ANT_HOME/lib where ANT_HOME is 
the ant install directory. The second is $HOME/.ant/lib. This is useful when files cannot be added to ANT_HOME 
which is the case for many linux installs.

The problem with installing ivy using these methods is all projects need to use the same version of ivy or
risk classpath issues. Dynamically loading ivy to a project location allows every project to use a different
version of ivy.

With a build cache base.xml downloads a version of ivy for the project and installs it at runtime. Ivy will 
only be downloaded once unless the build cache is cleaned.

The ivy version installed is 2.4.0 but any version may be installed by setting the ivy.version property before 
importing base.xml.

## ivysettings.xml

Ivy comes with [default settings](http://ant.apache.org/ivy/history/2.1.0/tutorial/defaultconf.htm) in the jar. 

```
<ivysettings>
  <settings defaultResolver="default"/>
  <include url="${ivy.default.settings.dir}/ivysettings-public.xml"/>
  <include url="${ivy.default.settings.dir}/ivysettings-shared.xml"/>
  <include url="${ivy.default.settings.dir}/ivysettings-local.xml"/>
  <include url="${ivy.default.settings.dir}/ivysettings-main-chain.xml"/>
  <include url="${ivy.default.settings.dir}/ivysettings-default-chain.xml"/>
</ivysettings>
```

The includes elements above pointing to ${ivy.default.settings.dir} provide more default settings. These 
settings create a default-chain which will resolve the local first and then the main chain. The main chain 
resolves to shared and public resolvers. This is similar to resolving a local maven repo before resolving 
shared and public repositories. Our goal is to use default setting and only override the public resolver 
so we can use jcenter. We can reuse these settings while only replacing the public resolver to use
jcenter.

```
<ivysettings>
    <settings defaultResolver="default"/>
    <resolvers>
      <ibiblio name="public" root="https://jcenter.bintray.com/" m2compatible="true"/>
    </resolvers>
    <include url="${ivy.default.settings.dir}/ivysettings-shared.xml"/>
    <include url="${ivy.default.settings.dir}/ivysettings-local.xml"/>
    <include url="${ivy.default.settings.dir}/ivysettings-main-chain.xml"/>
    <include url="${ivy.default.settings.dir}/ivysettings-default-chain.xml"/>
</ivysettings>
```

Now we can add public dependencies to our project through jcenter. To do this we need to add another file: 
ivy.xml.

## ivy.xml

ivy.xml declares configurations and dependencies for the project. It also contains information used to resolve 
the project itself. It is similar to setting up configurations and dependencies in gradle.

ivy.xml can reuse properties from build.xml. It is useful to declare project info in 
build.xml and then reference them in ivy.xml. This way project info is only declared once. This is especially 
true for the project name.

```
<ivy-module version="2.0">
    <info organisation="${project.organisation}" module="${ant.project.name}"/>
</ivy-module>
```

Here we reuse the project name and add a new property called project.organisation.

From here configuring ivy depends on what is needed for the project. A good start which may be more advanced 
for ant is to add build testing to the project. This will allow the build to be developed with automated tests. 
antunit can be used to test the build but first it must be added.

### Adding the first build dependencies

Gradle has a concept of plugins. This allows developers to add new features to gradle such as support for 
languages, publishing artifacts, and generating reports. An antlib has the same function. They add new types 
and tasks to ant. Ivy is an example of an antlib. It add dependency management support to ant. Now that ivy is 
loaded into the project it can be used to manage dependencies for bringing in other ant libraries.

#### build configuration

Like gradle configurations, ivy configurations are sets of dependencies used for a specific purpose. The build 
configuration is the set of build dependencies. It is be added to ivy.xml.

```
<ivy-module version="2.0">
    <info organisation="${project.organisation}" module="${ant.project.name}"/>
    <configurations defaultconfmapping="build->master">
        <conf name="build" visibility="private"
            description="libraries added to the ant build classpath"/>
    </configurations>
</ivy-module>
```
#### build dependencies

Now that we have a configuration a dependency can be added. Here is the full file.

```
<ivy-module version="2.0">
    <info organisation="${project.organisation}" module="${ant.project.name}"/>
    <configurations defaultconfmapping="build->master">
        <conf name="build" visibility="private"
            description="libraries added to the ant build classpath"/>
    </configurations>
    <dependencies>
        <dependency org="org.apache.ant" name="ant-antunit" rev="1.3" conf="build"/>
    </dependencies>
</ivy-module>
```

The `build` configuration will contain the antunit jar only.

## build initialisation

Initialising the build involves adding tasks and types to ant for all dependencies in the build configuration.
Every ant lib added to the build configuration needs to loaded with a `taskdef` task. To do this a classpath
is generated using ivy and `taskdef`s are added for each antlib. Here is the code:

```
<target name="configureAntUnit" extensionOf="configure">
    <taskdef resource="org/apache/ant/antunit/antlib.xml" uri="antlib:org.apache.ant.antunit"
        classpathref="build.classpath"/>
</target>
```

# testing the build with ant.xml

## running tests

Gradle has support out of the box to test tasks and plugins for a project. The same can be done with 
antunit using ant.xml.

ant.xml adds an extension to the test phase which runs all antunit tests. The task will execute all tests in 
src.antunit.ant.dir. The tests are located in `src/antunit/ant`. This is to follow the standard maven project 
layout gradle follows. Output of the tests is generated in the console as well as an junit xml and junit html.

## making tests

Testing a build.xml can be tricky. ant.xml will move tests to the build directory and run them. This allows 
local directories to be used and cleaned up without affecting the project sources.

### test base

A base xml file can be used to set project properties correctly before the project is included.

```
<project xmlns:au="antlib:org.apache.ant.antunit"> 
    <property name="build.dir" location="build/test-build"/>
    <property name="build.cache.dir" location="build/test-build-cache"/>
    <property name="reports.ant.unit.dir" location="build/reports/antunit"/>
    <import file="${antunit.build.base}"/>
</project>
```

Using an include allows all targets to be prefixed with `project.`. This ensures antunit only executes antunit 
tests rather than project test targets. This base build file can be imported by other tests. Every property 
in the main project is available in tests. For example:

```
<project xmlns:au="antlib:org.apache.ant.antunit">
    <import file="project-base.xml"/>
    <target name="test-project-name">
        <au:assertPropertyEquals name="ant.project.name" value="base-project"/>
    </target>
</project>
```

This tests the project name that is dynamically set to the project directory.

All project tasks are available in tests prefixed with `project`. For example:

```
<project xmlns:au="antlib:org.apache.ant.antunit">

    <include file="project-base.xml" as="project"/>

    <target name="tearDown" depends="project.clean"/>

    <target name="test-clean-no-build">
        <au:assertFileDoesntExist file="${build.dir}"/>
        <antcall target="project.clean"/>
        <au:assertFileDoesntExist file="${build.dir}"/>
    </target>

    <target name="test-clean">
        <au:assertFileDoesntExist file="${build.dir}"/>
        <antcall target="project.build"/>
        <au:assertFileExists file="${build.dir}"/>
        <antcall target="project.clean"/>
        <au:assertFileDoesntExist file="${build.dir}"/>
    </target>
</project>
```

These tests execute the project targets and check the resulting files.
