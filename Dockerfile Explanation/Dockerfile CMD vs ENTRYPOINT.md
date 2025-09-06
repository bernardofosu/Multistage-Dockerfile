📝 CMD

Think of CMD as the default arguments (like a fallback).

It can be overridden easily at docker run time.

Example:

FROM ubuntu:20.04
CMD ["echo", "Hello from CMD"]


Running:

docker run myimage


→ prints Hello from CMD.

But if you run:

docker run myimage echo "Override CMD"


→ prints Override CMD.
(So CMD gets replaced.)

📝 ENTRYPOINT

Think of ENTRYPOINT as the fixed command that always runs.

Any extra arguments you pass to docker run get appended to ENTRYPOINT.

Example:

FROM ubuntu:20.04
ENTRYPOINT ["echo"]


Running:

docker run myimage Hello


→ prints Hello.

Here, Hello is appended to echo.

🧩 CMD + ENTRYPOINT Together

Best practice is often to use them together:

FROM ubuntu:20.04
ENTRYPOINT ["echo"]
CMD ["Hello from CMD"]


Running docker run myimage → prints Hello from CMD.

Running docker run myimage Hi there → prints Hi there.

So ENTRYPOINT defines the program, and CMD defines the default arguments.

🚀 DevOps Angle

Use ENTRYPOINT when you want your container to act like a specific executable (e.g., always java -jar myapp.jar).

Use CMD when you want to provide defaults but allow easy override (e.g., default config file or options).

⚡ Quick analogy:

ENTRYPOINT = the engine of the car 🚗. It’s always there.

CMD = the GPS destination 🗺️. You can set a default, but the driver can change it at runtime.