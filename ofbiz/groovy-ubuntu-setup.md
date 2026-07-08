# Groovy Setup on Ubuntu (Java 21 + Groovy 5)

This guide explains how to install and configure Groovy on Ubuntu for local learning.

After completing this setup, you will be able to:

- Run Groovy scripts from terminal
- Use interactive Groovy shell (`groovysh`)
- Experiment with Groovy closures, classes, and syntax
- Use Groovy similar to how you use Java (`java`, `jshell`)

---

# 1. Install Java (JDK)

Check if Java is already installed:

```bash
java --version
```

Expected output:

```
openjdk 21.x.x
```

If Java is not installed:

```bash
sudo apt update
sudo apt install openjdk-21-jdk
```

Verify:

```bash
java --version
```

---

# 2. Configure JAVA_HOME

Find the Java installation path:

```bash
readlink -f $(which java)
```

Example output:

```
/usr/lib/jvm/java-21-openjdk-amd64/bin/java
```

The Java home path is:

```
/usr/lib/jvm/java-21-openjdk-amd64
```

Open bash configuration:

```bash
nano ~/.bashrc
```

Add these lines at the bottom:

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
```

Save the file.

Reload configuration:

```bash
source ~/.bashrc
```

Verify:

```bash
echo $JAVA_HOME
```

Expected:

```
/usr/lib/jvm/java-21-openjdk-amd64
```

---

# 3. Install SDKMAN

SDKMAN is a tool manager for JVM technologies like:

- Java
- Groovy
- Gradle
- Maven

Install SDKMAN:

```bash
curl -s "https://get.sdkman.io" | bash
```

Reload:

```bash
source ~/.bashrc
```

Verify:

```bash
sdk version
```

Example:

```
SDKMAN 5.x.x
```

---

# 4. Install Groovy

Install Groovy using SDKMAN:

```bash
sdk install groovy
```

Verify:

```bash
groovy --version
```

Expected:

```
Groovy Version: 5.x.x
JVM: 21.x.x
```

---

# 5. Start Groovy Interactive Shell

Run:

```bash
groovysh
```

Expected:

```
Groovy Shell (5.x.x, JVM: 21.x.x)

groovy>
```

Now you can execute Groovy code directly.

Example:

```groovy
println "Hello Groovy"
```

Output:

```
Hello Groovy
```

Exit:

```
/exit
```

---

# 6. Run a Groovy File

Create a Groovy file:

```bash
nano Hello.groovy
```

Add:

```groovy
println "Hello from Groovy"
```

Run:

```bash
groovy Hello.groovy
```

Output:

```
Hello from Groovy
```

---

# 7. Test Groovy Variables

Open:

```bash
groovysh
```

Run:

```groovy
def name = "Groovy"

println name
```

Output:

```
Groovy
```

---

# 8. Test Groovy Closure

Create:

```bash
nano ClosureTest.groovy
```

Add:

```groovy
def greet = {
    println "Hello Closure"
}

greet()
```

Run:

```bash
groovy ClosureTest.groovy
```

Output:

```
Hello Closure
```

---

# 9. Test Groovy Closure Scope

Create:

```bash
nano ScopeChecker.groovy
```

Add:

```groovy
class ScopeChecker {

    String target = "Class Target"

    def runTest() {

        String target = "Method Target"

        def myClosure = {
            println target
        }

        myClosure()
    }
}

new ScopeChecker().runTest()
```

Run:

```bash
groovy ScopeChecker.groovy
```

Output:

```
Method Target
```

Explanation:

The closure uses the variable from the method scope because Groovy closures capture variables from their surrounding scope.

---

# 10. Useful Groovy Commands

Check Groovy version:

```bash
groovy --version
```

Start interactive shell:

```bash
groovysh
```

Run Groovy script:

```bash
groovy FileName.groovy
```

Exit Groovy shell:

```
/exit
```

Show help:

```
/help
```

---

# Troubleshooting

## Problem: JAVA_HOME error

Error:

```
groovy: JAVA_HOME is not defined correctly
```

Check:

```bash
echo $JAVA_HOME
```

It should show:

```
/usr/lib/jvm/java-21-openjdk-amd64
```

Reload:

```bash
source ~/.bashrc
```

---

## Problem: groovysh fails with Security Manager error

Example:

```
Security Manager is deprecated
```

Cause:

Old Groovy versions (example: Groovy 2.4) are not compatible with Java 21.

Check:

```bash
groovy --version
```

Recommended:

```
Groovy 5 + Java 21
```

Install latest Groovy:

```bash
sdk install groovy
```

---

# OFBiz Note

If you work on Apache OFBiz:

OFBiz has its own Gradle and Groovy dependencies.

Keep system Groovy separate.

Use:

## For learning Groovy

```bash
groovysh

groovy MyScript.groovy
```

## For OFBiz

```bash
./gradlew

./gradlew ofbiz
```

Do not change OFBiz dependencies just to install Groovy.

---

# Final Verification

Run these commands:

```bash
java --version
```

```bash
echo $JAVA_HOME
```

```bash
groovy --version
```

```bash
groovysh
```

If all commands work, Groovy is ready for development.