📜 The Line
web: java $JAVA_OPTS -jar target/dependency/webapp-runner.jar --port $PORT target/*.war

🔎 Breakdown

web:

Declares the process type.

Heroku (and similar PaaS) expects a web process for handling HTTP requests.

This is what gets exposed to the internet.

java

Runs the Java runtime.

$JAVA_OPTS

Placeholder for extra JVM options, injected by the platform at runtime.

Examples: -Xmx300m (memory limit), garbage collector flags, debugging options.

Keeps the Procfile flexible—no hardcoding JVM settings.

-jar target/dependency/webapp-runner.jar

Tells Java to run the Webapp Runner JAR.

webapp-runner is a lightweight Tomcat wrapper that lets you run .war files directly without installing a full Tomcat server.

--port $PORT

Instructs Webapp Runner to bind on the port given by the platform.

$PORT is an environment variable (Heroku assigns a random free port).

Without this, your app wouldn’t receive traffic from the load balancer.

target/*.war

The actual application artifact (your packaged web app).

*.war expands to something like target/my-app.war built by Maven.

Webapp Runner will deploy this WAR on its embedded Tomcat.

DevOps Angle

This one line acts as your startup manifest.

As a DevOps engineer, you don’t need to know the app’s internal logic—you just need to ensure the Procfile points to the right WAR file, JVM options, and port binding.

It’s equivalent to:

Docker → CMD ["java", "-jar", "webapp-runner.jar", "--port", "8080", "my-app.war"]

Kubernetes → command: ["java", "-jar", "webapp-runner.jar", "--port", "$(PORT)", "my-app.war"]