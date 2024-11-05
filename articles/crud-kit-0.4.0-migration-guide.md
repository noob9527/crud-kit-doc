# Crud-kit 0.4.0 migration guide

1. upgrade spring boot to 2.7.x, resolve all problems
  also don't forget to upgrade `io.spring.dependency-management` plugin, otherwise you might get some misleading gradle dependency error.  
  consider add `implementation 'org.springframework.boot:spring-boot-starter-validation'`
2. configure jdk version to at least 17
  e.g.
  ```
  org.gradle.java.home=<Your JDK Path>/azul-17.0.12/Contents/Home
  ```
  replace all
  ```
  java.sourceCompatibility = JavaVersion.VERSION_1_8
  sourceCompatibility = '1.8'
  sourceCompatibility = "1.8"
  jvmTarget = "1.8"
  jvmTarget = '1.8'
  ```
  into
  ```
  java {
      toolchain {
          languageVersion = JavaLanguageVersion.of(17)
      }
  }
  kotlin {
      jvmToolchain(17)
  }
  ```
3. upgrade gradle to latest(for now, it's 8.8)
  with the following command
  ```
  gradle wrapper --gradle-version=8.8 --distribution-type=all
  ```
  otherwise you might encounter the following problem
  ```
    > Could not resolve all artifacts for configuration ':crud-kit-demo:classpath'.
      > Could not resolve org.springframework.boot:spring-boot-gradle-plugin:3.3.2.
      Required by:
      project :crud-kit-demo > org.springframework.boot:org.springframework.boot.gradle.plugin:3.3.2
      > No matching variant of org.springframework.boot:spring-boot-gradle-plugin:3.3.2 was found. The consumer was configured to find a runtime of a library compatible with Java 17, packaged as a jar, and its dependencies declared externally, as well as attribute 'org.gradle.plugin.api-version' with value '7.2' but:
      - Variant 'apiElements' capability org.springframework.boot:spring-boot-gradle-plugin:3.3.2 declares a library compatible with Java 17, packaged as a jar, and its dependencies declared externally:
      - Incompatible because this component declares an API of a component and the consumer needed a runtime of a component
      - Other compatible attribute:
      - Doesn't say anything about org.gradle.plugin.api-version (required '7.2')
      - Variant 'javadocElements' capability org.springframework.boot:spring-boot-gradle-plugin:3.3.2 declares a runtime of a component, and its dependencies declared externally:
      - Incompatible because this component declares documentation and the consumer needed a library
      - Other compatible attributes:
      - Doesn't say anything about its target Java version (required compatibility with Java 17)
      - Doesn't say anything about its elements (required them packaged as a jar)
      - Doesn't say anything about org.gradle.plugin.api-version (required '7.2')
      - Variant 'modernGradleRuntimeElements' capability org.springframework.boot:spring-boot-gradle-plugin:3.3.2 declares a runtime of a library compatible with Java 17, packaged as a jar, and its dependencies declared externally:
      - Incompatible because this component declares a component, as well as attribute 'org.gradle.plugin.api-version' with value '8.7' and the consumer needed a component, as well as attribute 'org.gradle.plugin.api-version' with value '7.2'
      - Variant 'runtimeElements' capability org.springframework.boot:spring-boot-gradle-plugin:3.3.2 declares a runtime of a library compatible with Java 17, packaged as a jar, and its dependencies declared externally:
      - Incompatible because this component declares a component, as well as attribute 'org.gradle.plugin.api-version' with value '7.5' and the consumer needed a component, as well as attribute 'org.gradle.plugin.api-version' with value '7.2'
      - Variant 'sourcesElements' capability org.springframework.boot:spring-boot-gradle-plugin:3.3.2 declares a runtime of a component, and its dependencies declared externally:
      - Incompatible because this component declares documentation and the consumer needed a library
      - Other compatible attributes:
      - Doesn't say anything about its target Java version (required compatibility with Java 17)
      - Doesn't say anything about its elements (required them packaged as a jar)
      - Doesn't say anything about org.gradle.plugin.api-version (required '7.2')
  ```
4. upgrade kotlin version to latest(for now, it's 1.9.22)
  otherwise you might encounter the following problem
  ```
  'void org.gradle.api.artifacts.dsl.DependencyHandler.registerTransform(org.gradle.api.Action)'
  ```
  Also, you might need to remove
  ```
  implementation 'org.jetbrains.kotlin:kotlin-stdlib-jdk8'
  ```
5. upgrade spring boot 2.7.x to 3.x
  You might want to use [idea's migration tool](https://blog.jetbrains.com/idea/2021/06/intellij-idea-eap-6/), in short, Refactor > Migrate > Java EE to Jakarta EE
6. upgrade mybatis 2.x to 3.x
  otherwise, you may encounter
  ```
  Caused by: java.lang.NoClassDefFoundError: org/springframework/core/NestedIOException
  ```
  see https://github.com/spring-projects/spring-framework/issues/28198

References:
- [Spring-Boot-3.0-Migration-Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#configuration-properties-migration)
