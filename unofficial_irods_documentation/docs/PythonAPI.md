# Python API

Some notes on best use, along with some cookbook snippets that people might find helpful.

## Get the size of all objects under a particular collection

Code snippet from [#373](https://github.com/irods/python-irodsclient/issues/373)

```
total_size_in_bytes = 0
n_data_objs = 0
collection = session.collections.get('/myZone/home/myUser/home/testCollection')
for info in collection.walk( ):
    n_data_objs += len( info[2] )
    total_size_in_bytes += sum( d.size for d in info[2] )
```
