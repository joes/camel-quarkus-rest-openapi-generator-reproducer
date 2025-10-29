# camel-quarkus-rest-openapi-generator-reproducer

## Camel Quarkus OpenAPI codegen outputs Java files under `src/main/java`, causing IDE package mismatch

When using the Camel Quarkus REST OpenAPI code generator, the generated Java sources end up under:

```
target/generated-sources/camel-quarkus-rest-openapi/src/main/java
```

However, Quarkus's `GenerateCodeMojo` (which invokes the provider) assumes `.java` files are generated directly into:

```
target/generated-sources/camel-quarkus-rest-openapi
```

As a result, the Quarkus plugin automatically registers the top-level directory  
`target/generated-sources/camel-quarkus-rest-openapi`  
as a compile source root - even though the actual Java files live one level deeper.

This leads to **package mismatch errors in IDEs** such as Visual Studio Code and Eclipse:

```
The declared package "org.acme.ping.model" does not match the expected package ""
```

The Maven build itself succeeds (`BUILD SUCCESS`) because the incorrect directory simply contains no `.java` files and is ignored by `javac`.

---

## Steps to Reproduce

1. Build with debug logging:

   ```powershell
   ./mvn clean compile -X
   ```
   You'll see:

   ```
   [DEBUG] Source roots:
   [DEBUG]  C:\camel-quarkus-rest-openapi-generator-reproducer\src\main\java
   [DEBUG]  C:\camel-quarkus-rest-openapi-generator-reproducer\target\generated-sources\camel-quarkus-rest-openapi
   [DEBUG]  C:\camel-quarkus-rest-openapi-generator-reproducer\target\generated-sources\annotations
   ```

2. Open the project in Visual Studio Code and Problems pane shows errors like:

   ```
   The declared package "org.apache.camel.quarkus" does not match the expected package "src.main.java.org.apache.camel.quarkus"
   ```
---

## Expected Behavior

The provider should generate `.java` files directly under:

```
target/generated-sources/camel-quarkus-rest-openapi
```

so that Quarkus's `GenerateCodeMojo` registers a correct source root automatically.

---

## Root Cause

`org.apache.camel.quarkus.component.rest.openapi.deployment.CamelQuarkusSwaggerCodegenProvider.CamelQuarkusSwaggerCodegenProvider` calls:

```java
configurator.setOutputDir(context.outDir().toAbsolutePath().toString());
```

but the Swagger Codegen library always appends `src/main/java` internally, resulting in a nested layout:

```
target/generated-sources/camel-quarkus-rest-openapi/src/main/java
```

[`GenerateCodeMojo`](https://github.com/quarkusio/quarkus/blob/main/devtools/maven/src/main/java/io/quarkus/maven/GenerateCodeMojo.java) then registers only `target/generated-sources/camel-quarkus-rest-openapi` as the compile source root, creating the mismatch.

---

## Suggested Fix

Align the output directory in `org.apache.camel.quarkus.component.rest.openapi.deployment.CamelQuarkusSwaggerCodegenProvider` with what Quarkus expects so that generated Java files are placed directly under `target/generated-sources/camel-quarkus-rest-openapi/`.

---

## Environment

- Operating System: Windows 11 (23H2)
- Camel Quarkus: 3.27.0
- Quarkus: 3.28.5
- JDK: 17  
- IDE: Visual Studio Code 1.105.1 (user setup)

---

## Additional Notes

- Maven builds succeed because the misplaced directory is harmless at compile time.  
-

---


## Running the application in dev mode

You can run your application in dev mode that enables live coding using:

```shell script
./mvnw quarkus:dev
```

> **_NOTE:_**  Quarkus now ships with a Dev UI, which is available in dev mode only at <http://localhost:8080/q/dev/>.

## Packaging and running the application

The application can be packaged using:

```shell script
./mvnw package
```

It produces the `quarkus-run.jar` file in the `target/quarkus-app/` directory.
Be aware that it's not an _über-jar_ as the dependencies are copied into the `target/quarkus-app/lib/` directory.

The application is now runnable using `java -jar target/quarkus-app/quarkus-run.jar`.

If you want to build an _über-jar_, execute the following command:

```shell script
./mvnw package -Dquarkus.package.jar.type=uber-jar
```

The application, packaged as an _über-jar_, is now runnable using `java -jar target/*-runner.jar`.

## Creating a native executable

You can create a native executable using:

```shell script
./mvnw package -Dnative
```

Or, if you don't have GraalVM installed, you can run the native executable build in a container using:

```shell script
./mvnw package -Dnative -Dquarkus.native.container-build=true
```

You can then execute your native executable with: `./target/camel-quarkus-rest-openapi-generator-reproducer-1.0.0-SNAPSHOT-runner`

If you want to learn more about building native executables, please consult <https://quarkus.io/guides/maven-tooling>.

## Related Guides

- Camel REST OpenApi ([guide](https://camel.apache.org/camel-quarkus/latest/reference/extensions/rest-openapi.html)): To call REST services using OpenAPI specification as contract
- Camel Core ([guide](https://camel.apache.org/camel-quarkus/latest/reference/extensions/core.html)): Camel core functionality and basic Camel languages: Constant, ExchangeProperty, Header, Ref, Simple and Tokenize
