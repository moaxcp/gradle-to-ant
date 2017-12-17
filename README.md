# gradle-to-ant

This project implements gradle conventions in ant. It can be used to help learn gradle conventions or to help 
prepare ant projects for a switch to gradle.

# Conventions

1. project directory is project name

```
<project>
    <basename property="ant.project.name" file="${basedir}"/>
    <echo>Project: ${ant.project.name}</echo>
</project>
```

2. dependency management made easy with ivy

