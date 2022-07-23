# Making changes to the core project

(Copied from [6471](https://github.com/irods/irods/issues/6471) )

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

## Custom plugins

(Taken from [#6458](https://github.com/irods/irods/issues/6458))

Calling a rule defined in a custom plugin (a .so file) which is defined in the server conf with a particular `instance_name`.

To make this work, your plugin must implement exec_rule_text(). You can use the Logical Quotas REP as a reference for achieving this. See the following:

    https://github.com/irods/irods_rule_engine_plugin_logical_quotas/blob/df75c52f6d8aa89a2b452aeb56626c2f944b5986/src/main.cpp#L360
    https://github.com/irods/irods_rule_engine_plugin_logical_quotas/blob/df75c52f6d8aa89a2b452aeb56626c2f944b5986/src/main.cpp#L331
    https://github.com/irods/irods_rule_engine_plugin_logical_quotas/blob/df75c52f6d8aa89a2b452aeb56626c2f944b5986/src/main.cpp#L220

If you want to invoke rules within your plugin from a delay rule, you'll also need to implement exec_rule_expression(). The Logical Quotas REP supports both of these functions and demonstrates how someone can support these through a single function.

Keep in mind that the implementation for your plugin could be very different from the Logical Quotas implementation depending on how your plugin accepts input. Logical Quotas accepts JSON as input. If your plugin also accepts JSON, you should be able to use the same pattern as the Logical Quotas REP.
