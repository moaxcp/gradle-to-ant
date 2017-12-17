# gradle-to-ant

This project implements gradle conventions in ant. It can be used to help learn gradle conventions or to help 
prepare ant projects for a switch to gradle.

# project directory is project name

In gradle the project name defaults to the directory of the project. This can be changed in settings.gradle
but it is a good convention to follow in ant. The project name in ant can be set dynamically by removing it 
from the project element and defining the `ant.project.name` property.

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

# build directory contains everything built

In gradle everything built goes into the build directory. A build directory can be implemented in ant.

First we add the build directory location. It is important to use a property location rather than a property 
value. One code smell in ant is to use a property value rather than location to set a property. Using a location 
sets the property to the absolute path on the filesystem.

```
<property name="build.dir" location="build"/>
```

Next we need to add tasks to initialize the build directory.

```
<target name="init">
    <mkdir dir="${build.dir}"/>
</target>
```

# clean deletes everything built

Simply delete build.dir.

```
<target name="clean">
    <delete dir="${build.dir}"/>
</target>
```

# dependency management made easy with ivy

Gradle uses ivy concepts for dependency management. It is also a really good system for ant. In order to use
ivy in ant it needs to be installed. There are two static locations ivy can be installed for use in ant.
The first is ANT_HOME/lib where ANT_HOME is the ant install directory. The second is $HOME/.ant/lib. This
is useful when files cannot be added to ANT_HOME.

The problem with installing ivy with these methods is all projects need to use the same version of ivy or
risk classpath issues. Dynamically loading ivy to a project location allows every project to use a different
version of ivy.

Now that there is a build directory we can dynamically install ivy to the build directory at runtime.

## installing ivy at runtime

First we want to set a property for the version of ivy to use.

```
<property name="ivy.version" value="2.4.0"/>
```

This is a convienece property that will make it easy to update the version of ivy used.

Next the `unless` xml namespace needs to be added to the project to build the task for installing ivy. This will 
make more sense once the task is introduced.

```
<project xmlns:unless="ant:unless">
```

Now the install-ivy task is ready to be made. First we need to setup a few local properties.

```
<local name="ivy.dir"/>
<local name="ivy.file"/>
<local name="ivy.available"/>

<property name="ivy.dir" location="${build.dir}/ivy"/>
<property name="ivy.file" location="${ivy.dir}/ivy-${ivy.version}.jar"/>
<available file="${ivy.file}" property="ivy.present"/>
```

Ivy will be installed into build/ivy so it can be cleaned. The jar also has the version which makes updating 
the ivy version easier. `ivy.present` will be set if the ivy jar is present in build/ivy. This property allows the 
target to skip downloading the file when possible. Here is the complete target:

```
<target name="install-ivy" depends="init">
    <local name="ivy.dir"/>
    <local name="ivy.file"/>
    <local name="ivy.available"/>

    <property name="ivy.dir" location="${build.dir}/ivy"/>
    <property name="ivy.file" location="${ivy.dir}/ivy-${ivy.version}.jar"/>
    <available file="${ivy.file}" property="ivy.present"/>

    <mkdir unless:set="ivy.present" dir="${ivy.dir}"/>
    <get unless:set="ivy.present" dest="${ivy.file}"
            src="http://search.maven.org/remotecontent?filepath=org/apache/ivy/ivy/${ivy.version}/ivy-${ivy.version}.j$

    <taskdef resource="org/apache/ivy/ant/antlib.xml" uri="antlib:org.apache.ivy.ant" classpath="${ivy.file}"/>
</target>
```

ivy.dir is created and the file is downloaded unless ivy.file is available. taskdef is used to load the ant 
tasks from the jar.

## using ivy for dependency management

Gradle projects have methods to configure repositories, configurations, and dependencies. While ivy has these 
concepts the configuration has less conventions than gradle.

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
settings 
create a default-chain which will 
resolve the local first and then the main chain. The main chain resolves to shared and public resolvers. This 
is similar to resolving a local maven repo before resolving shared and public repositories. Our goal 
is to use default setting and only override the public resolver so we can use jcenter. We can reuse these 
settings replacing the public resolver.

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
the project itself. ivy.xml can reuse properties from build.xml. It is useful to declare project info in 
build.xml and then reference them in ivy.xml. This way project info is only declared once. This is especially 
true for the project name.

```
<ivy-module version="2.0">
    <info organisation="${project.organisation}" module="${ant.project.name}"/>
</ivy-module>
```

Here we reuse the project name and add a new property called project.organisation.

From here configuring ivy depends on what is needed for the project. A good start which may be more advanced is 
to add build testing to the project. This will allow the build to be developed with automated tests. antunit 
can be used to test the build but first it must be added.

# Adding the first build dependencies

Gradle has a concept of plugins. This allows developers to add new features to gradle such as support for 
languages, publishing artifacts, and generating reports. An antlib has the same function. They add new types 
and tasks to ant. Ivy is an example of an antlib. It add dependency management support to ant. Now that ivy is 
loaded into the project it can be used to manage dependencies for bringing in other ant libraries.

## build configuration

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

# testing the build

Gradle has support out of the box to test tasks and plugins for a project. The same can be done with 
antunit.
