## Use multi-stage build

A multi-stage build divides the Dockerfile into multiple stages, each with its own purpose. You can use one stage to install dependencies, another to build the application, and a final stage to run it.

I have used : base, deps, build, and final 
Check here: [https://github.com/Ayachi-Sharma/Tech.Care/blob/main/Dockerfile]

Benifits:
    Smaller final images by excluding unnecessary build dependencies and source files.
    Better separation of build and runtime responsibilities.
    faster rebuilds through Docker's layer caching.
    Improved security by reducing unnecessary software in the final image.
    Easy debugging cause of clean steps.

## Best-Practice for Dockerfile:

Docker allows a stage to use a previous stage as its starting point. This helps reuse configuration and dependencies instead of repeating instructions.

Use a previous stage as a new stage  
    FROM base as deps
    FROM deps as build
    FROM base as final

This approach keeps the Dockerfile organized and helps prevent unnecessary build files from being included in the final image.