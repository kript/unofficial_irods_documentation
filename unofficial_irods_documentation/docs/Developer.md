# Making changes to the core project

(Coped from [6471](https://github.com/irods/irods/issues/6471) )

If you're trying to make changes to the server, it is going to require additional steps.

Here is a quick breakdown of some of the most important directories:

    lib/api/src: Client-side API functions (basically wrappers around procApiRequest)
    lib/core/src: Code that is used on the client-side and server-side
    server/api/src: Server-side API endpoint implementations
    server/core/src: Server-side utilities, main server startup logic, etc.
    plugins/database/src: Database specific logic (db_plugin.cpp)

## General Tips:

    Functions with an rc prefix represent client-side APIs
        These calls invoke policy
    Functions with an rs prefix represent server-side APIs
        These calls do not invoke policy
        Functions with a leading underscore are meant for server-side use only
    The entry point for the server (startup, delay server forking, etc) can be found in rodsServer.cpp
    The agent factory logic can be found in rodsAgent.cpp
    When adding a new file to the server, it will require updating one or more CMakeLists.txt files
        Start with the CMakeLists.txt file located at the root of the repo. That file will help in understanding the structure of the project
    An easy way to see if your change is being compiled into the server is by adding a log message to rsDataObjPut() and then invoking iput. If everything compiles, then you should see your log message in the log file
    Last but not least, we highly recommend compiling and installing all of the packages produced by the https://github.com/irods/irods_development_environment. Doing so avoids weird situations around ABI, etc.
