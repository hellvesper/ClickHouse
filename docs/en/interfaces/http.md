---
toc_priority: 19
toc_title: HTTP Interface
---

# HTTP Interface {#http-interface}

The HTTP interface lets you use ClickHouse on any platform from any programming language. We use it for working from Java and Perl, as well as shell scripts. In other departments, the HTTP interface is used from Perl, Python, and Go. The HTTP interface is more limited than the native interface, but it has better compatibility.

By default, clickhouse-server listens for HTTP on port 8123 (this can be changed in the config).

If you make a GET / request without parameters, it returns 200 response code and the string which defined in [http\_server\_default\_response](../operations/server_configuration_parameters/settings.md#server_configuration_parameters-http_server_default_response) default value “Ok.” (with a line feed at the end)

``` bash
$ curl 'http://localhost:8123/'
Ok.
```

Use GET /ping request in health-check scripts. This handler always returns “Ok.” (with a line feed at the end). Available from version 18.12.13.

``` bash
$ curl 'http://localhost:8123/ping'
Ok.
```

Send the request as a URL ‘query’ parameter, or as a POST. Or send the beginning of the query in the ‘query’ parameter, and the rest in the POST (we’ll explain later why this is necessary). The size of the URL is limited to 16 KB, so keep this in mind when sending large queries.

If successful, you receive the 200 response code and the result in the response body.
If an error occurs, you receive the 500 response code and an error description text in the response body.

When using the GET method, ‘readonly’ is set. In other words, for queries that modify data, you can only use the POST method. You can send the query itself either in the POST body or in the URL parameter.

Examples:

``` bash
$ curl 'http://localhost:8123/?query=SELECT%201'
1

$ wget -O- -q 'http://localhost:8123/?query=SELECT 1'
1

$ echo -ne 'GET /?query=SELECT%201 HTTP/1.0\r\n\r\n' | nc localhost 8123
HTTP/1.0 200 OK
Date: Wed, 27 Nov 2019 10:30:18 GMT
Connection: Close
Content-Type: text/tab-separated-values; charset=UTF-8
X-ClickHouse-Server-Display-Name: clickhouse.ru-central1.internal
X-ClickHouse-Query-Id: 5abe861c-239c-467f-b955-8a201abb8b7f
X-ClickHouse-Summary: {"read_rows":"0","read_bytes":"0","written_rows":"0","written_bytes":"0","total_rows_to_read":"0"}

1
```

As you can see, curl is somewhat inconvenient in that spaces must be URL escaped.
Although wget escapes everything itself, we don’t recommend using it because it doesn’t work well over HTTP 1.1 when using keep-alive and Transfer-Encoding: chunked.

``` bash
$ echo 'SELECT 1' | curl 'http://localhost:8123/' --data-binary @-
1

$ echo 'SELECT 1' | curl 'http://localhost:8123/?query=' --data-binary @-
1

$ echo '1' | curl 'http://localhost:8123/?query=SELECT' --data-binary @-
1
```

If part of the query is sent in the parameter, and part in the POST, a line feed is inserted between these two data parts.
Example (this won’t work):

``` bash
$ echo 'ECT 1' | curl 'http://localhost:8123/?query=SEL' --data-binary @-
Code: 59, e.displayText() = DB::Exception: Syntax error: failed at position 0: SEL
ECT 1
, expected One of: SHOW TABLES, SHOW DATABASES, SELECT, INSERT, CREATE, ATTACH, RENAME, DROP, DETACH, USE, SET, OPTIMIZE., e.what() = DB::Exception
```

By default, data is returned in TabSeparated format (for more information, see the “Formats” section).
You use the FORMAT clause of the query to request any other format.

``` bash
$ echo 'SELECT 1 FORMAT Pretty' | curl 'http://localhost:8123/?' --data-binary @-
┏━━━┓
┃ 1 ┃
┡━━━┩
│ 1 │
└───┘
```

The POST method of transmitting data is necessary for INSERT queries. In this case, you can write the beginning of the query in the URL parameter, and use POST to pass the data to insert. The data to insert could be, for example, a tab-separated dump from MySQL. In this way, the INSERT query replaces LOAD DATA LOCAL INFILE from MySQL.

Examples: Creating a table:

``` bash
$ echo 'CREATE TABLE t (a UInt8) ENGINE = Memory' | curl 'http://localhost:8123/' --data-binary @-
```

Using the familiar INSERT query for data insertion:

``` bash
$ echo 'INSERT INTO t VALUES (1),(2),(3)' | curl 'http://localhost:8123/' --data-binary @-
```

Data can be sent separately from the query:

``` bash
$ echo '(4),(5),(6)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20VALUES' --data-binary @-
```

You can specify any data format. The ‘Values’ format is the same as what is used when writing INSERT INTO t VALUES:

``` bash
$ echo '(7),(8),(9)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20FORMAT%20Values' --data-binary @-
```

To insert data from a tab-separated dump, specify the corresponding format:

``` bash
$ echo -ne '10\n11\n12\n' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20FORMAT%20TabSeparated' --data-binary @-
```

Reading the table contents. Data is output in random order due to parallel query processing:

``` bash
$ curl 'http://localhost:8123/?query=SELECT%20a%20FROM%20t'
7
8
9
10
11
12
1
2
3
4
5
6
```

Deleting the table.

``` bash
$ echo 'DROP TABLE t' | curl 'http://localhost:8123/' --data-binary @-
```

For successful requests that don’t return a data table, an empty response body is returned.

You can use the internal ClickHouse compression format when transmitting data. The compressed data has a non-standard format, and you will need to use the special `clickhouse-compressor` program to work with it (it is installed with the `clickhouse-client` package). To increase the efficiency of data insertion, you can disable server-side checksum verification by using the [http\_native\_compression\_disable\_checksumming\_on\_decompress](../operations/settings/settings.md#settings-http_native_compression_disable_checksumming_on_decompress) setting.

If you specified `compress=1` in the URL, the server compresses the data it sends you.
If you specified `decompress=1` in the URL, the server decompresses the same data that you pass in the `POST` method.

You can also choose to use [HTTP compression](https://en.wikipedia.org/wiki/HTTP_compression). To send a compressed `POST` request, append the request header `Content-Encoding: compression_method`. In order for ClickHouse to compress the response, you must append `Accept-Encoding: compression_method`. ClickHouse supports `gzip`, `br`, and `deflate` [compression methods](https://en.wikipedia.org/wiki/HTTP_compression#Content-Encoding_tokens). To enable HTTP compression, you must use the ClickHouse [enable\_http\_compression](../operations/settings/settings.md#settings-enable_http_compression) setting. You can configure the data compression level in the [http\_zlib\_compression\_level](#settings-http_zlib_compression_level) setting for all the compression methods.

You can use this to reduce network traffic when transmitting a large amount of data, or for creating dumps that are immediately compressed.

Examples of sending data with compression:

``` bash
#Sending data to the server:
$ curl -vsS "http://localhost:8123/?enable_http_compression=1" -d 'SELECT number FROM system.numbers LIMIT 10' -H 'Accept-Encoding: gzip'

#Sending data to the client:
$ echo "SELECT 1" | gzip -c | curl -sS --data-binary @- -H 'Content-Encoding: gzip' 'http://localhost:8123/'
```

!!! note "Note"
    Some HTTP clients might decompress data from the server by default (with `gzip` and `deflate`) and you might get decompressed data even if you use the compression settings correctly.

You can use the ‘database’ URL parameter to specify the default database.

``` bash
$ echo 'SELECT number FROM numbers LIMIT 10' | curl 'http://localhost:8123/?database=system' --data-binary @-
0
1
2
3
4
5
6
7
8
9
```

By default, the database that is registered in the server settings is used as the default database. By default, this is the database called ‘default’. Alternatively, you can always specify the database using a dot before the table name.

The username and password can be indicated in one of three ways:

1.  Using HTTP Basic Authentication. Example:

<!-- -->

``` bash
$ echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-
```

1.  In the ‘user’ and ‘password’ URL parameters. Example:

<!-- -->

``` bash
$ echo 'SELECT 1' | curl 'http://localhost:8123/?user=user&password=password' -d @-
```

1.  Using ‘X-ClickHouse-User’ and ‘X-ClickHouse-Key’ headers. Example:

<!-- -->

``` bash
$ echo 'SELECT 1' | curl -H 'X-ClickHouse-User: user' -H 'X-ClickHouse-Key: password' 'http://localhost:8123/' -d @-
```

If the user name is not specified, the `default` name is used. If the password is not specified, the empty password is used.
You can also use the URL parameters to specify any settings for processing a single query or entire profiles of settings. Example:http://localhost:8123/?profile=web&max\_rows\_to\_read=1000000000&query=SELECT+1

For more information, see the [Settings](../operations/settings/index.md) section.

``` bash
$ echo 'SELECT number FROM system.numbers LIMIT 10' | curl 'http://localhost:8123/?' --data-binary @-
0
1
2
3
4
5
6
7
8
9
```

For information about other parameters, see the section “SET”.

Similarly, you can use ClickHouse sessions in the HTTP protocol. To do this, you need to add the `session_id` GET parameter to the request. You can use any string as the session ID. By default, the session is terminated after 60 seconds of inactivity. To change this timeout, modify the `default_session_timeout` setting in the server configuration, or add the `session_timeout` GET parameter to the request. To check the session status, use the `session_check=1` parameter. Only one query at a time can be executed within a single session.

You can receive information about the progress of a query in `X-ClickHouse-Progress` response headers. To do this, enable [send\_progress\_in\_http\_headers](../operations/settings/settings.md#settings-send_progress_in_http_headers). Example of the header sequence:

``` text
X-ClickHouse-Progress: {"read_rows":"2752512","read_bytes":"240570816","total_rows_to_read":"8880128"}
X-ClickHouse-Progress: {"read_rows":"5439488","read_bytes":"482285394","total_rows_to_read":"8880128"}
X-ClickHouse-Progress: {"read_rows":"8783786","read_bytes":"819092887","total_rows_to_read":"8880128"}
```

Possible header fields:

-   `read_rows` — Number of rows read.
-   `read_bytes` — Volume of data read in bytes.
-   `total_rows_to_read` — Total number of rows to be read.
-   `written_rows` — Number of rows written.
-   `written_bytes` — Volume of data written in bytes.

Running requests don’t stop automatically if the HTTP connection is lost. Parsing and data formatting are performed on the server-side, and using the network might be ineffective.
The optional ‘query\_id’ parameter can be passed as the query ID (any string). For more information, see the section “Settings, replace\_running\_query”.

The optional ‘quota\_key’ parameter can be passed as the quota key (any string). For more information, see the section “Quotas”.

The HTTP interface allows passing external data (external temporary tables) for querying. For more information, see the section “External data for query processing”.

## Response Buffering {#response-buffering}

You can enable response buffering on the server-side. The `buffer_size` and `wait_end_of_query` URL parameters are provided for this purpose.

`buffer_size` determines the number of bytes in the result to buffer in the server memory. If a result body is larger than this threshold, the buffer is written to the HTTP channel, and the remaining data is sent directly to the HTTP channel.

To ensure that the entire response is buffered, set `wait_end_of_query=1`. In this case, the data that is not stored in memory will be buffered in a temporary server file.

Example:

``` bash
$ curl -sS 'http://localhost:8123/?max_result_bytes=4000000&buffer_size=3000000&wait_end_of_query=1' -d 'SELECT toUInt8(number) FROM system.numbers LIMIT 9000000 FORMAT RowBinary'
```

Use buffering to avoid situations where a query processing error occurred after the response code and HTTP headers were sent to the client. In this situation, an error message is written at the end of the response body, and on the client-side, the error can only be detected at the parsing stage.

### Queries with Parameters {#cli-queries-with-parameters}

You can create a query with parameters and pass values for them from the corresponding HTTP request parameters. For more information, see [Queries with Parameters for CLI](cli.md#cli-queries-with-parameters).

### Example {#example}

``` bash
$ curl -sS "<address>?param_id=2&param_phrase=test" -d "SELECT * FROM table WHERE int_column = {id:UInt8} and string_column = {phrase:String}"
```

## Predefined HTTP Interface {#predefined_http_interface}

ClickHouse supports specific queries through the HTTP interface. For example, you can write data to a table as follows:

``` bash
$ echo '(4),(5),(6)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20VALUES' --data-binary @-
```

ClickHouse also supports Predefined HTTP Interface which can help you more easy integration with third party tools like [Prometheus exporter](https://github.com/percona-lab/clickhouse_exporter).

Example:

-   First of all, add this section to server configuration file:

<!-- -->

``` xml
<http_handlers>
  <predefine_query_handler>
      <url>/metrics</url>
        <method>GET</method>
        <queries>
            <query>SELECT * FROM system.metrics LIMIT 5 FORMAT Template SETTINGS format_template_resultset = 'prometheus_template_output_format_resultset', format_template_row = 'prometheus_template_output_format_row', format_template_rows_between_delimiter = '\n'</query>
        </queries>
  </predefine_query_handler>
</http_handlers>
```

-   You can now request the url directly for data in the Prometheus format:

<!-- -->

``` bash
curl -vvv 'http://localhost:8123/metrics'
*   Trying ::1...
* Connected to localhost (::1) port 8123 (#0)
> GET /metrics HTTP/1.1
> Host: localhost:8123
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Wed, 27 Nov 2019 08:54:25 GMT
< Connection: Keep-Alive
< Content-Type: text/plain; charset=UTF-8
< X-ClickHouse-Server-Display-Name: i-tl62qd0o
< Transfer-Encoding: chunked
< X-ClickHouse-Query-Id: f39235f6-6ed7-488c-ae07-c7ceafb960f6
< Keep-Alive: timeout=3
< X-ClickHouse-Summary: {"read_rows":"0","read_bytes":"0","written_rows":"0","written_bytes":"0","total_rows_to_read":"0"}
<
# HELP "Query" "Number of executing queries"
# TYPE "Query" counter
"Query" 1

# HELP "Merge" "Number of executing background merges"
# TYPE "Merge" counter
"Merge" 0

# HELP "PartMutation" "Number of mutations (ALTER DELETE/UPDATE)"
# TYPE "PartMutation" counter
"PartMutation" 0

# HELP "ReplicatedFetch" "Number of data parts being fetched from replica"
# TYPE "ReplicatedFetch" counter
"ReplicatedFetch" 0

# HELP "ReplicatedSend" "Number of data parts being sent to replicas"
# TYPE "ReplicatedSend" counter
"ReplicatedSend" 0

* Connection #0 to host localhost left intact
```

As you can see from the example, if `<http_handlers>` is configured in the config.xml file, ClickHouse will match the HTTP requests received to the predefined type in `<http_handlers>`, then ClickHouse will execute the corresponding predefined query if the match is successful.

Now `<http_handlers>` can configure `<root_handler>`, `<ping_handler>`, `<replicas_status_handler>`, `<dynamic_query_handler>` and `<no_handler_description>` .

## root\_handler {#root_handler}

`<root_handler>` returns the specified content for the root path request. The specific return content is configured by `http_server_default_response` in config.xml. if not specified, return **Ok.**

`http_server_default_response` is not defined and an HTTP request is sent to ClickHouse. The result is as follows:

``` xml
<http_handlers>
    <root_handler/>
</http_handlers>
```

    $ curl 'http://localhost:8123'
    Ok.

`http_server_default_response` is defined and an HTTP request is sent to ClickHouse. The result is as follows:

``` xml
<http_server_default_response><![CDATA[<html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>]]></http_server_default_response>

<http_handlers>
    <root_handler/>
</http_handlers>
```

    $ curl 'http://localhost:8123'
    <html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>%

## ping\_handler {#ping_handler}

`<ping_handler>` can be used to probe the health of the current ClickHouse Server. When the ClickHouse HTTP Server is normal, accessing ClickHouse through `<ping_handler>` will return **Ok.**.

Example:

``` xml
<http_handlers>
    <ping_handler>/ping</ping_handler>
</http_handlers>
```

``` bash
$ curl 'http://localhost:8123/ping'
Ok.
```

## replicas\_status\_handler {#replicas_status_handler}

`<replicas_status_handler>` is used to detect the state of the replica node and return **Ok.** if the replica node has no delay. If there is a delay, return the specific delay. The value of `<replicas_status_handler>` supports customization. If you do not specify `<replicas_status_handler>`, ClickHouse default setting `<replicas_status_handler>` is **/replicas\_status**.

Example:

``` xml
<http_handlers>
    <replicas_status_handler>/replicas_status</replicas_status_handler>
</http_handlers>
```

No delay case:

``` bash
$ curl 'http://localhost:8123/replicas_status'
Ok.
```

Delayed case:

``` bash
$ curl 'http://localhost:8123/replicas_status'
db.stats:  Absolute delay: 22. Relative delay: 22.
```

## predefined\_query\_handler {#predefined_query_handler}

You can configure `<method>`, `<headers>`, `<url>` and `<queries>` in `<predefined_query_handler>`.

`<method>` is responsible for matching the method part of the HTTP request. `<method>` fully conforms to the definition of [method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) in the HTTP protocol. It is an optional configuration. If it is not defined in the configuration file, it does not match the method portion of the HTTP request

`<url>` is responsible for matching the url part of the HTTP request. It is compatible with [RE2](https://github.com/google/re2)’s regular expressions. It is an optional configuration. If it is not defined in the configuration file, it does not match the url portion of the HTTP request

`<headers>` is responsible for matching the header part of the HTTP request. It is compatible with RE2’s regular expressions. It is an optional configuration. If it is not defined in the configuration file, it does not match the header portion of the HTTP request

`<queries>` value is a predefined query of `<predefined_query_handler>`, which is executed by ClickHouse when an HTTP request is matched and the result of the query is returned. It is a must configuration.

`<predefined_query_handler>` supports setting Settings and query\_params values.

The following example defines the values of `max_threads` and `max_alter_threads` settings, then queries the system table to check whether these settings were set successfully.

Example:

``` xml
<root_handlers>
    <predefined_query_handler>
        <method>GET</method>
        <headers>
            <XXX>TEST_HEADER_VALUE</XXX>
            <PARAMS_XXX><![CDATA[(?P<name_1>[^/]+)(/(?P<name_2>[^/]+))?]]></PARAMS_XXX>
        </headers>
        <url><![CDATA[/query_param_with_url/\w+/(?P<name_1>[^/]+)(/(?P<name_2>[^/]+))?]]></url>
        <queries>
            <query>SELECT value FROM system.settings WHERE name = {name_1:String}</query>
            <query>SELECT name, value FROM system.settings WHERE name = {name_2:String}</query>
        </queries>
    </predefined_query_handler>
</root_handlers>
```

``` bash
$ curl -H 'XXX:TEST_HEADER_VALUE' -H 'PARAMS_XXX:max_threads' 'http://localhost:8123/query_param_with_url/1/max_threads/max_alter_threads?max_threads=1&max_alter_threads=2'
1
max_alter_threads   2
```

!!! note "Note"
    In one `<predefined_query_handler>`, one `<queries>` only supports one `<query>` of an insert type.

## dynamic\_query\_handler {#dynamic_query_handler}

`<dynamic_query_handler>` than `<predefined_query_handler>` increased `<query_param_name>` .

ClickHouse extracts and executes the value corresponding to the `<query_param_name>` value in the url of the HTTP request.
ClickHouse default setting `<query_param_name>` is `/query` . It is an optional configuration. If there is no definition in the configuration file, the param is not passed in.

To experiment with this functionality, the example defines the values of max\_threads and max\_alter\_threads and queries whether the Settings were set successfully.
The difference is that in `<predefined_query_handler>`, query is wrote in the configuration file. But in `<dynamic_query_handler>`, query is written in the form of param of the HTTP request.

Example:

``` xml
<root_handlers>
    <dynamic_query_handler>
        <headers>
            <XXX>TEST_HEADER_VALUE_DYNAMIC</XXX>
            <PARAMS_XXX><![CDATA[(?P<param_name_1>[^/]+)(/(?P<param_name_2>[^/]+))?]]></PARAMS_XXX>
        </headers>
        <query_param_name>query_param</query_param_name>
    </dynamic_query_handler>
</root_handlers>
```

``` bash
$ curl  -H 'XXX:TEST_HEADER_VALUE_DYNAMIC' -H 'PARAMS_XXX:max_threads' 'http://localhost:8123/?query_param=SELECT%20value%20FROM%20system.settings%20where%20name%20=%20%7Bname_1:String%7D%20OR%20name%20=%20%7Bname_2:String%7D&max_threads=1&max_alter_threads=2&param_name_2=max_alter_threads'
1
2
```

[Original article](https://clickhouse.tech/docs/en/interfaces/http_interface/) <!--hide-->
