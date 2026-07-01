# Table of Contents

## Getting Started
* [Introduction](README.md)
* [Installation](getting-started/installation.md)
* [Quick Start](getting-started/quick-start.md)
* [Entry Point](getting-started/entry-point.md)

## Guides
* [Defining a Server](guides/defining-a-server.md)
* [Configuring a Session](guides/configuring-a-session.md)
* [Defining a Handler](guides/handler.md)
* [Defining a Middleware](guides/middleware.md)
* [Manager System](guides/manager-system.md)

## API Reference
* Attributes
    * [Server](api-reference/attribute/attr-server.md)
    * [Handler](api-reference/attribute/attr-handler.md)
    * [Manager](api-reference/attribute/attr-manager.md)
    * [Route](api-reference/attribute/attr-routing.md)
    * [Middleware](api-reference/attribute/attr-middleware.md)
    * [Parameter](api-reference/attribute/attr-parameters.md)
    * [Utility](api-reference/attribute/attr-utility.md)
* Handlers
    * [RouteHandler](api-reference/handler/route-handler.md)
* Middlewares
    * [MiddlewareHandler](api-reference/middleware/middleware-handler.md)
* Managers
    * [ManagerBase](api-reference/manager/manager-base.md)
* Models
    * [HttpModel](api-reference/model/http-model.md)
    * [Custom Http Model](api-reference/model/custom-http-models.md)
    * [RequestModel](api-reference/model/request-model.md)
    * [ResponseModel](api-reference/model/response-model.md)
* Pools
    * [PooledObject](api-reference/pool/pooled-object.md)
* Storage
    * [Buffer](api-reference/storage/storage-buffer.md)
    * [FileCache](api-reference/storage/storage-filecache.md)
    * [HeapQueue & Event](api-reference/storage/storage-scheduling.md)
* UTF-8
    * [ByteString & Comparer](api-reference/utf8/utf8-bytestring.md)
    * [Utf8Builder](api-reference/utf8/utf8-builder.md)
    * [UTF-8 Collections](api-reference/utf8/utf8-collections.md)
* Views
    * [Socket Info](api-reference/view/socket-info.md)
* NetCoreServer
    * Transport
        * Core
            * [ServerTransport](api-reference/netcoreserver/Transport/Core/transport-server.md)
            * [SessionTransport](api-reference/netcoreserver/Transport/Core/transport-session.md)
            * [ClientTransport](api-reference/netcoreserver/Transport/Core/transport-client.md)
            * [StreamSessionTransport](api-reference/netcoreserver/Transport/Core/transport-sessionstream.md)
            * [StreamClientTransport](api-reference/netcoreserver/Transport/Core/transport-clientstream.md)
        * TCP
            * [TcpServer](api-reference/netcoreserver/Transport/Tcp/tcp-server.md)
            * [TcpSession](api-reference/netcoreserver/Transport/Tcp/tcp-session.md)
            * [TcpClient](api-reference/netcoreserver/Transport/Tcp/tcp-client.md)
        * SSL
            * [SslServer](api-reference/netcoreserver/Transport/Ssl/ssl-server.md)
            * [SslSession](api-reference/netcoreserver/Transport/Ssl/ssl-session.md)
            * [SslClient](api-reference/netcoreserver/Transport/Ssl/ssl-client.md)
            * [SslContext](api-reference/netcoreserver/Transport/Ssl/ssl-context.md)
        <!-- * UDP
            * [UdpServer](api-reference/netcoreserver/Transport/Udp/udp-server.md)
            * [UdpSession](api-reference/netcoreserver/Transport/Udp/udp-session.md)
            * [UdpClient](api-reference/netcoreserver/Transport/Udp/udp-client.md) -->
        * UDS
            * [UdsServer](api-reference/netcoreserver/Transport/Uds/uds-server.md)
            * [UdsSession](api-reference/netcoreserver/Transport/Uds/uds-session.md)
            * [UdsClient](api-reference/netcoreserver/Transport/Uds/uds-client.md)
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
