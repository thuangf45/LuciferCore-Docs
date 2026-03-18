# Table of Contents

## Overview
* [Introduction](getting-started/README.md)

## Getting Started
* [Installation](getting-started/installation.md)
* [Quick Start](getting-started/quick-start.md)
* [Entry Point](getting-started/entry-point.md)

## Guides
* [Defining a Server](guides/defining-a-server.md)
* [Configuring a Session](guides/configuring-a-session.md)
* [WebSocket Handler](guides/websocket-handler.md)
* [HTTP Handler](guides/http-handler.md)
* [Manager System](guides/manager-system.md)
* [Console Commands](guides/console-commands.md)

## API Reference
* Attributes
    * [Server](api-reference/attribute/attr-server.md)
    * [Handler](api-reference/attribute/attr-handler.md)
    * [Manager](api-reference/attribute/attr-manager.md)
    * [Config](api-reference/attribute/attr-config.md)
    * [RateLimiter](api-reference/attribute/attr-ratelimiter.md)
    * [Authorize](api-reference/attribute/attr-authorize.md)
    * [Safe](api-reference/attribute/attr-safe.md)
    * [Routing Attributes](api-reference/attribute/attr-routing.md)
    * [Parameter Attributes](api-reference/attribute/attr-parameters.md)
    * [Utility Attributes](api-reference/attribute/attr-utility.md)
* Handlers
    * [HandlerBase](api-reference/handler/handler-base.md)
    * [WebSocket Handler Base](api-reference/handler/handler-ws.md)
    * [HTTP Handler Base](api-reference/handler/handler-http.md)
* Managers
    * [Manager Base](api-reference/manager/manager-base.md)
    * [Manager Guide](api-reference/manager/manager-guide.md)
* Models
    * [Packet Model](api-reference/model/model-packet.md)
    * [Request Model](api-reference/model/model-request.md)
    * [Response Model](api-reference/model/model-response.md)

* NetCoreServer
    * Transport
        * Core
            * [SessionTransport](api-reference/netcoreserver/Transport/Core/transport-session.md)
            * [ServerTransport](api-reference/netcoreserver/Transport/Core/transport-server.md)
            * [ClientTransport](api-reference/netcoreserver/Transport/Core/transport-client.md)
            * [StreamSessionTransport](api-reference/netcoreserver/Transport/Core/transport-stream-session.md)
            * [StreamClientTransport](api-reference/netcoreserver/Transport/Core/transport-stream-client.md)
        * TCP
            * [TcpServer](api-reference/netcoreserver/Transport/Tcp/tcp-server.md)
            * [TcpSession](api-reference/netcoreserver/Transport/Tcp/tcp-session.md)
            * [TcpClient](api-reference/netcoreserver/Transport/Tcp/tcp-client.md)
        * SSL
            * [SslContext](api-reference/netcoreserver/Transport/Ssl/ssl-context.md)
            * [SslServer](api-reference/netcoreserver/Transport/Ssl/ssl-server.md)
            * [SslSession](api-reference/netcoreserver/Transport/Ssl/ssl-session.md)
            * [SslClient](api-reference/netcoreserver/Transport/Ssl/ssl-client.md)
        * UDP
            * [UdpServer](api-reference/netcoreserver/Transport/Udp/udp-server.md)
            * [UdpSession](api-reference/netcoreserver/Transport/Udp/udp-session.md)
            * [UdpClient](api-reference/netcoreserver/Transport/Udp/udp-client.md)
        * UDS
            * [UdsServer](api-reference/netcoreserver/Transport/Uds/uds-server.md)
            * [UdsSession](api-reference/netcoreserver/Transport/Uds/uds-session.md)
            * [UdsClient](api-reference/netcoreserver/Transport/Uds/uds-client.md)
    * Protocols
        * [HTTP Protocol](api-reference/netcoreserver/Protocol/http-protocol.md)
        * [WebSocket Protocol](api-reference/netcoreserver/Protocol/ws-protocol.md)
    * Servers
        * [HttpServer](api-reference/netcoreserver/Server/server-http.md)
        * [HttpsServer](api-reference/netcoreserver/Server/server-https.md)
        * [WsServer](api-reference/netcoreserver/Server/server-ws.md)
        * [WssServer](api-reference/netcoreserver/Server/server-wss.md)
    * Sessions
        * [HttpSession](api-reference/netcoreserver/Session/session-http.md)
        * [HttpsSession](api-reference/netcoreserver/Session/session-https.md)
        * [WsSession](api-reference/netcoreserver/Session/session-ws.md)
        * [WssSession](api-reference/netcoreserver/Session/session-wss.md)
    * Clients
        * [HttpClient](api-reference/netcoreserver/Client/client-http.md)
        * [HttpsClient](api-reference/netcoreserver/Client/client-https.md)
        * [WsClient](api-reference/netcoreserver/Client/client-ws.md)
        * [WssClient](api-reference/netcoreserver/Client/client-wss.md)

* Pools
    * [PooledObject](api-reference/pool/pooled-object.md)
* Services
    * [Service Base](api-reference/service/service-base.md)
    * [Service Guide](api-reference/service/service-guide.md)
* Storage
    * [Buffer](api-reference/storage/storage-buffer.md)
    * [FileCache](api-reference/storage/storage-filecache.md)
    * [StorageData](api-reference/storage/storage-storagedata.md)
    * [HeapQueue & Event](api-reference/storage/storage-scheduling.md)
* UTF-8 Utility
    * [ByteString & Comparer](api-reference/utf8/utf8-bytestring.md)
    * [Utf8Builder](api-reference/utf8/utf8-builder.md)
    * [UTF-8 Collections](api-reference/utf8/utf8-collections.md)