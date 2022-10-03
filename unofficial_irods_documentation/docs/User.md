# User / Client Notes

## icommands

There is no garuntee of compatability between versions, so it's best to install the version that's running on the serevrs.
If you have some installed already, you can use the `imiscsrvinfo` icommand to check the remote system is running the same version.

If you didn't want to take a chance on the wrong version, you can ask the server. For example, from the command line (replace localhost with the server name);

```
root@irodsServer:/# echo "hello" | nc localhost 1247 | egrep -a rods
<relVersion>rods4.2.10</relVersion>
```
