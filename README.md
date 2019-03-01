# lnav HAProxy Log Format

Format file for parsing [HAProxy](http://www.haproxy.org/) log files with [The Log File Navigator (lnav)](http://lnav.org/).

* Author: Berengar W. Lehr <Berengar.Lehr@kompetenztest.de>
* Date: 2019-03-01
* Licence: AGPL-3.0-or-later

There is an [official log format](http://cbonte.github.io/haproxy-dconv/1.9/configuration.html#8.2). *My* HAProxy log file consists of two parts, a base and an individual body. Regular expressions in this document will be acompanied by an example line.

## Base
The base consists of three parts:

1. Timestamp: two formats are currently supported by this format file, other can be added easily.
    ```regexp
    (?<timestamp>\w{3} \d{2} \d{2}:\d{2}:\d{2}|\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\+\d{2}:\d{2)
                 Jan 02 15:04:05               2006-01-02T15:04:05+07:00
    ```

2. Logging server: can either be the servers name or it's IP address.
    ```regexp
    (?<logging_host>[^ ]+)
                    192.168.1.3, rsyslog_server
    ```

3. Process name and PID.    
    ```regexp
    (?<process_name>\w+)\[(?<pid>\d+)\]:
                    haproxy[7]:
    ```

## Body
Again *my* HAProxy log contains entries about events or connections which again are divided into *ssl* error, *tcp* or *http* logs.
```regexp
(?<body>(%events%|%connections%)
```

### Event logs
Events are logged when a Server is a) started b) getting stopped or c) stopped.
```regexp
Proxy (?<proxy_name>[^ ]+) started.|
Stopping frontend (?<proxy_name>[^ ]+) in (?<stopping_timeout>\d+) ms.|
Proxy (?<proxy_name>[^ ]+) stopped \(FE: (?<frontend_connections>\d+) conns, BE: (?<backend_connections>\d+) conns\).
```    

### Connection logs
Connection logs start with client identification, connection timestamp, front- and backend name.

1. Client identification: consists of an IP and port number (sorry, I never sa an IPv6 entry, maybe someone could add that).
    ```regexp
    (?<client_ip>[^:]+):(?<client_port>\d+)
                 141.35.244.175:45678
    ```

2. Timestamp: again, one format is currently defined, others can be added. It's embraced by square brackets.
    ```regexp
     \[(?<accept_date>\d{2}\/\w{3}\/\d{4}:\d{2}:\d{2}:\d{2}.\d{3})\]
     [                 02/Jan/2006:15:04:05.008                    ]
    ```

3. Frontend: the name of the frontend as defined in the config file. HTTP entries may contain a trailing tilde (`~`) to indicate a SSL/HTTPS connection. SSL error logs skipt this part
    ```regexp
    ((?<frontend_name>[^ ]+)(?<ssl>~)?)?
                     http_prod_front    
    ```

4. Backend: consists of the backend and the server name separated by a slash
    ```regexp
    (?<backend_name>[^ ]+)\/(?<server_name>[^ ]+)
                     http_prod_back/nginx_1
    ```

### Further entries
All further log parts are explained in detail in the HAProxy Documentation for the [TCP log format](http://cbonte.github.io/haproxy-dconv/1.9/configuration.html#8.2.2) and the [ log format](http://cbonte.github.io/haproxy-dconv/1.9/configuration.html#8.2.3) respectively. Custom logs can be defined but are outside the scope of this explenation.
```regexp
(?:(: %ssl%| %tcp%| %http%)
```

#### SSL errors
Skipping the frontend entry, the backend/server-name is followed by colon and afer that runs the SSL eror text, examples entries are:
* Timeout during SSL handshake
* SSL handshake failure
* Connection closed during SSL handshake
* Connection error during SSL handshake
```regexp
(?<ssl_error>.*)
```

#### TCP
```regexp
(?<tw>\\d+)\\/(?<tc>\\d+)\\/(?<tt>\\d+)
0/0/0
(?<bytes_read>\\d+)
161
(?<termination_state>..)
--
(?<actconn>\\d+)\\/(?<feconn>\\d+)\\/(?<beconn>\\d+)\\/(?<srv_conn>\\d+)\\/(?<retries>\\d+)
0/7/7/4/3
(?<srv_queue>\\d+)\\/(?<backend_queue>\\d+)
1/0
```

#### HTTP
Captured request and response headers are optional and embraced by curly braces, the http command is quoted. Missformed requests are only shown as `<BADREQ>`. Long http-urls might lead to incomplete lines in which the http-version and the closing quote are missing.

```regexp
(?<tq>-?\\d+)\\/(?<tw>-?\\d+)\\/(?<tc>-?\\d+)\\/(?<tr>-?\\d+)\\/(?<tt>\\d+)
6/9/1/1/5
(?<status_code>\\d{3}|-1)
418
(?<bytes_read>\\d+)
161
(?<captured_request_cookie>.*)
OMM=nomnomnom
(?<captured_response_cookie>.*)
balance=a_cookie_in_every_hand
(?<termination_state>....)
....
(?<actconn>\\d+)\\/(?<feconn>\\d+)\\/(?<beconn>\\d+)\\/(?<srv_conn>\\d+)\\/(?<retries>\\d+)
1/6/7/9/8
(?<srv_queue>\\d+)\\/(?<backend_queue>\\d+)
1/0
\\{(?<captured_request_headers>.*)\\}
{empty}
\\{(?<captured_response_headers>.*)\\}
{full}
\"(?<http_method>[A-Z<>]+)(?: (?<http_url>.*?))?(?: (?<http_version>HTTP\\/\\d+.\\d+))?\"
"POST index.php?id=e4f58a805a6e1fd0f6bef58c86f9ceb3#top HTTP/1.1"
```

## Acknowledgement
This format file ist partly based on Work by [Adam Petrovic](https://gist.github.com/adampetrovic/7c87134c46b3db5b8e82) and is inteded to be used with [Tim Stacks](https://github.com/tstack/lnav) excellent log file navigator that can also be used to extract and reform log file entries.