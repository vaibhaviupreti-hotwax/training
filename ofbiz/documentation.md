# New OFBiz project - 
## Step 1 - Check Setup 
### Foundational Folders
- framework/: The core technical engine (entity engine, service engine, webapp framework).
- applications/: The standard out-of-the-box ERP business apps (e.g., party, order, accounting, product).
- themes/: The visual styling and UI templates for the frontend.
- plugins/: Custom or third-party add-ons. (Note: It's completely normal for this to be empty on a fresh clone).

### build system
- build.gradle: The main configuration file for building the project.
- gradlew (Mac/Linux) or gradlew.bat (Windows): The script you use to run build commands.

### Compilation status
- Look for: build/libs/ofbiz.jar
- If it's missing: The project has not been compiled yet (or was recently cleaned). You must run ./gradlew build or ./gradlew loadAll.
- If it's there, the system is compiled and ready to be started with ./gradlew ofbiz.

### Database and data initialization
- runtime/data/derby/ofbiz/ (default) (if empty - "./gradlew loadAll" to populate it)

### for new run
- ./gradlew cleanAll loadAll (This builds the .jar files and seeds the database. Once it finishes, you just run ./gradlew ofbiz to start the server.)

### for the second time - skip cleanAll loadAll, what you run next entirely depends on what kind of files you are editing.
- If you changed .java code, you need to recompile: ./gradlew build
- You changed UI or configs(XML, Groovy, FTL, HTML), no need to run gradle commands, OFBiz dynamically reloads these files.
- If you add complete new db Tables, just restart OFBiz.
- If you accidently deleted important data, just run ./gradlew loadAll
