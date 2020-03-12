OutbackCDX (nee tinycdxserver)
==============================

A [RocksDB](https://rocksdb.org/)-based capture index (CDX) server for web archives.

* [API Documentation](https://nla.github.io/outbackcdx/api.html)

Features:

* Speaks both [OpenWayback](https://github.com/iipc/openwayback/) (XML) and [PyWb](https://github.com/webrecorder/pywb) (JSON) CDX protocols
* Realtime, incremental updates
* Compressed indexes (varint packing + snappy), typically 1/4 - 1/5 the size of CDX files.
* Primary-secondary replication
* Access control (experimental, see below)

Things it doesn't do (yet):

* Sharding, replication
* CDXJ

Used in production at the National Library of Australia and British Library with
8-9 billion record indexes.

Usage
-----

Build:

    mvn package

Run:

    java -jar target/outbackcdx*.jar

Command line options:

```
Usage: java -jar outbackcdx.jar [options...]

  -b bindaddr           Bind to a particular IP address
  -c, --context-path url-prefix
                        Set a URL prefix for the application to be mounted under
  -d datadir            Directory to store index data under
  -i                    Inherit the server socket via STDIN (for use with systemd, inetd etc)
  -j jwks-url perm-path Use JSON Web Tokens for authorization
  -k url realm clientid Use a Keycloak server for authorization
  -m max-open-files     Limit the number of open .sst files to control memory usage
                        (default 396 based on system RAM and ulimit -n)
  -p port               Local port to listen on
  -t count              Number of web server threads
  -r count              Cap on number of rocksdb records to scan to serve a single request
  -x                    Output CDX14 by default (instead of CDX11)
  -v                    Verbose logging
  -y file               Custom fuzzy match canonicalization YAML configuration file

Primary mode (runs as a replication target for downstream Secondaries)
  --replication-window interval      interval, in seconds, to delete replication history from disk.
                                     0 disables automatic deletion. History files can be deleted manually by
                                     POSTing a replication sequenceNumber to /<collection>/truncate_replication

Secondary mode (runs read-only; polls upstream server on 'collection-url' for changes)
  --primary collection-url           URL of collection on upstream primary to poll for changes
  --update-interval poll-interval    Polling frequency for upstream changes, in seconds. Default: 10
  --accept-writes                    Allow writes to this node, even though running as a secondary
  --batch-size                       Approximate max size (in bytes) per replication batch
```

The server supports multiple named indexes as subdirectories.  Currently indexes
are created automatically when you first write records to them.

### Loading records

OutbackCDX does not include a CDX indexing tool for reading WARC or ARC files. Use
the `cdx-indexer` scripts included with OpenWayback or PyWb.

You can load records into the index by POSTing them in the (11-field) CDX format
Wayback uses:

    $ cdx-indexer mycrawlw.warc.gz > records.cdx
    $ curl -X POST --data-binary @records.cdx http://localhost:8080/myindex
    Added 542 records

The canonicalized URL (first field) is ignored, OutbackCDX performs its own
canonicalization.

**Limitation:** Loading an extremely large number of CDX records in one POST request
can cause an [out of memory error](https://github.com/nla/outbackcdx/issues/13). 
Until this is fixed you may need to break your request up into several smaller ones. 
Most users send one POST per WARC file.

### Deleting records

Deleting records works the same way as loading them. POST the records you wish to
delete to /{collection}/delete:

    $ curl -X POST --data-binary @records.cdx http://localhost:8080/myindex/delete
    Deleted 542 records

When deleting OutbackCDX does not check whether the records actually existed in the
index. Deleting non-existent records has no effect and will not cause an error.

### Querying

Records can be queried in CDX format:

    $ curl 'http://localhost:8080/myindex?url=example.org'
    org,example)/ 20030402160014 http://example.org/ text/html 200 MOH7IEN2JAEJOHYXIEPEEGHOHG5VI=== - - 2248 396 mycrawl.warc.gz

CDX formatted as JSON arrays:
    
    $ curl 'http://localhost:8080/myindex?url=example.org&output=json'
    [
      [
        "org,example)/",
        20030402160014,
        "http://example.org/",
        "text/html",
        200,
        "MOH7IEN2JAEJOHYXIEPEEGHOHG5VI===",
        2248,
        396,
        "mycrawl.warc.gz"
      ]
    ]

OpenWayback "OpenSearch" XML:

    $ curl 'http://localhost:8080/myindex?q=type:urlquery+url:http%3A%2F%2Fexample.org%2F'
    <?xml version="1.0" encoding="UTF-8"?>
    <wayback>
       <request>
           <startdate>19960101000000</startdate>
           <enddate>20180526162512</enddate>
           <type>urlquery</type>
           <firstreturned>0</firstreturned>
           <url>org,example)/</url>
           <resultsrequested>10000</resultsrequested>
           <resultstype>resultstypecapture</resultstype>
       </request>
       <results>
           <result>
               <compressedoffset>396</compressedoffset>
               <compressedendoffset>2248</compressedendoffset>
               <mimetype>text/html</mimetype>
               <file>mycrawl.warc.gz</file>
               <redirecturl>-</redirecturl>
               <urlkey>org,example)/</urlkey>
               <digest>MOH7IEN2JAEJOHYXIEPEEGHOHG5VI===</digest>
               <httpresponsecode>200</httpresponsecode>
               <robotflags>-</robotflags>
               <url>http://example.org/</url>
               <capturedate>20030402160014</capturedate>
           </result>
       </results>
    </wayback>

Query URLs that match a given URL prefix:

    $ curl 'http://localhost:8080/myindex?url=http://example.org/abc&matchType=prefix'
    
Find the first 5 URLs with a given domain:

    $ curl 'http://localhost:8080/myindex?url=example.org&matchType=domain&limit=5'

Find the next 10 URLs in the index starting from the given URL prefix:

    $ curl 'http://localhost:8080/myindex?url=http://example.org/abc&matchType=range&limit=10'

Return results in reverse order:
 
    $ curl 'http://localhost:8080/myindex?url=example.org&sort=reverse'
 
Return results ordered closest to furthest from a given timestamp:

    $ curl 'http://localhost:8080/myindex?url=example.org&sort=closest&closest=20030402172120'

See the [API Documentation](https://nla.github.io/outbackcdx/api.html) for more details
about the available options.
        
Configuring replay tools
------------------------

### OpenWayback

Point Wayback at a OutbackCDX index by configuring a RemoteResourceIndex. See the example RemoteCollection.xml shipped with OpenWayback.

```xml
    <property name="resourceIndex">
      <bean class="org.archive.wayback.resourceindex.RemoteResourceIndex">
        <property name="searchUrlBase" value="http://localhost:8080/myindex" />
      </bean>
    </property>
```

### PyWb

Create a pywb config.yaml file containing:

```yaml
collections:
  testcol:
    archive_paths: /tmp/warcs/
    #archive_paths: http://remote.example.org/warcs/
    index:
      type: cdx
      api_url: http://localhost:8080/myindex?url={url}&closest={closest}&sort=closest
    
      # outbackcdx doesn't serve warc records 
      # so we blank replay_url to force pywb to read the warc file itself
      replay_url: ""
```

### Heritrix

The ukwa-heritrix project includes [some classes](https://github.com/ukwa/ukwa-heritrix/blob/21d31329065f2a6a68186309757a5644af00daec/src/main/java/uk/bl/wap/modules/uriuniqfilters/OutbackCDXRecentlySeenUriUniqFilter.java)
that allow OutbackCDX to be used as a source of deduplication data for Heritrix crawls.

Access Control
--------------

Access control can be enabled by setting the following environment variable:

    EXPERIMENTAL_ACCESS_CONTROL=1

Rules can be configured through the GUI. Have Wayback or other clients query a particular named access
point. For example to query the 'public' access point.

    http://localhost:8080/myindex/ap/public

See [docs/access-control.md](docs/access-control.md) for details of the access control model.

Canonicalisation Aliases
------------------------

Alias records allow the grouping of URLs so they will deliver as if they are different snapshots of the same page.

    @alias <source-url> <target-url>
    
For example:

    @alias http://legacy.example.org/page-one http://www.example.org/page1
    @alias http://legacy.example.org/page-two http://www.example.org/page2

Aliases do not currently work with url prefix queries. Aliases are resolved after normal canonicalisation rules
are applied.

Aliases can be mixed with regular CDX lines either in the same file or separate files and in any order. Any existing records that the alias rule affects the canonicalised URL for will be updated when the alias is added to the index.

Deletion of aliases is not yet implemented.

Tuning Memory Usage
-------------------

RocksDB some data in memory (binary search index, bloom filter) for each open SST file. This improves performance at
the cost of using more memory. OutbackCDX uses the following heuristic by default to limit the maximum number of open
SST files in an attempt not to exhaust the system's memory.

    RocksDB max_open_files = (totalSystemRam / 2 - maxJvmHeap) / 10 MB

This default may not be suitable when multiple large indexes are in use or when OutbackCDX is sharing a server with
many other processes. You can override the limit OutbackCDX's `-m` option.

If you find OutbackCDX using too much memory or you need more performance try adjusting the limit. The optimal setting
will depend on your index size and hardware. If you have a lot of memory `-m -1` (no limit) will allow RocksDB to open
all SST files on startup and should give the best query performance. However with slow disks it can also make startup
very slow. You may also need to increase the kernel's max open file description limit (`ulimit -n`).
 

Authorization
-------------

By default OutbackCDX is unsecured and assumes some external method of authorization such as firewall
rules or a reverse proxy are used to secure it. Take care not to expose it to the public internet.

Alternatively one of the following authorization methods can be enabled.

### Generic JWT authorization

Authorization to modify the index and access control rules can be controlled using [JSON Web Tokens](https://jwt.io/).
To enable this you will typically use some sort of separate authentication server to sign the JWTs.

OutbackCDX's `-j` option takes two arguments, a JWKS URL for the public key of the auth server and a slash-delimited
path for where to find the list of permissions in the JWT received as a HTTP bearer token. Refer to your auth server's
documentation for what to use.

Currently the OutbackCDX web dashboard does not support generic JWT/OIDC authorization. (Patches welcome.)

### Keycloak authorization

OutbackCDX can use [Keycloak](https://www.keycloak.org/) as an auth server to secure both the API and dashboard.

1. In your Keycloak realm's settings create a new client for OutbackCDX with the protocol `openid-connect` and
   the URL of your OutbackCDX instance.
3. Under the client's roles tab create the following roles:
    * index_edit - can create or delete index records
    * rules_edit - can create, modify or delete access rules
    * policies_edit - can create, modify or delete access policies
4. Map your users or service accounts to these client roles as appropriate.
5. Run OutbackCDX with this option:

```
-k https://{keycloak-server}/auth {realm} {client-id}
```

Note: JWT authentication will be enabled automatically when using Keycloak. You don't need to set the `-j` option.
