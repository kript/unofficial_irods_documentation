# System Administration Tricks and Tips

## Configuration Validation

Manually editing JSON config files can be fraught with issues. Fortunately, RENCI has built a validator into the system;

`python /var/lib/irods/scripts/validate_json.py /etc/irods/server_config
.json file:///var/lib/irods/configuration_schemas/v3/server_config.json`

Unfortunately, all THAT told me was that it was an invalid JSON file. Not much help!

Next, compare my backup of the config file with the live one. You DO take backups, right?. Of course you do!

Enter the `JSON-diff` tool! Also, not as helpful - that also told me the file wasn't valid JSON.

OK, how about a normal linter; `jsonlint`?

```
jsonlint-py3 /etc/irods/server_config.json
/etc/irods/server_config.json:55:8: Error: Expected a ']' but saw '}'
   |  At line 55, column 8, offset 1938
   |  Array started at line 54, column 20, offset 1928
/etc/irods/server_config.json:56:6: Error: Object properties (keys) must be string literals, numbers, or identifiers
   |  At line 56, column 6, offset 1947
   |  Object started at line 1, column 0, offset 0 (AT-START)
/etc/irods/server_config.json:56:6: Error: Missing value for object property, expected ":"
   |  At line 56, column 6, offset 1947
   |  Object started at line 1, column 0, offset 0 (AT-START)
/etc/irods/server_config.json:84:4: Error: Expected a '}' but saw ']'
   |  At line 84, column 4, offset 2867
   |  Object started at line 1, column 0, offset 0 (AT-START)
/etc/irods/server_config.json:84:4: Error: Unexpected text after end of JSON value
   |  At line 84, column 4, offset 2867
/etc/irods/server_config.json: has errors
```

OK, that's helpful, but I'd really like a more succinct 'what did I change'... back to good old `diff`

```
diff -c /etc/irods/server_config.json /etc/irods/server_config.json.20610.2022-05-04@15:55:10~
*** /etc/irods/server_config.json       2022-05-04 15:55:10.618672763 +0100
--- /etc/irods/server_config.json.20610.2022-05-04@15:55:10~    2021-09-17 11:13:50.011210000 +0100
***************
*** 52,58 ****
      "network": {},
      "resource": {},
      "rule_engines": [
-         },
        {
          "instance_name": "irods_rule_engine_plugin-irods_rule_language-instance",
          "plugin_name": "irods_rule_engine_plugin-irods_rule_language",
--- 52,57 ----
```

## Acting as another user

(Illustrated in [#6561](https://github.com/irods/irods/issues/6561))

A lot of the admin commands have the `-M` flag for rodsusers to effectively use _root_ like powers to act on an object et al that they do not have an ACL for.

However that's not always what you want, sometime you want to act as another user (say to upload a file as them). In this case you can set the environment variable `clientUserName`, e.g.

```bash
# as rods account
iput a.log /zone1/home/tom
# a.log is in 'tom' homedir, but they have not ACL to access it.
export clientUserName=tom
iput a.log /zone1/home/tom
# a.log is now owned by 'tom'
```
Note that this will only work for `rodsadmin` accounts - normal aka `rodsuser` users can't do this.
