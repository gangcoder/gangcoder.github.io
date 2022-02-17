<!-- ---
title: traefix
date: 2020-07-11 16:03:09
category: showcode, gateway, traefix
--- -->

# traefix 配置

配置示例：

## 静态配置

```toml
[global]
  checkNewVersion = true
  sendAnonymousUsage = true
  insecureSNI = false

[serversTransport]
  insecureSkipVerify = true
  rootCAs = ["foobar", "foobar"]
  maxIdleConnsPerHost = 42
  [serversTransport.forwardingTimeouts]
    dialTimeout = 42
    responseHeaderTimeout = 42
    idleConnTimeout = 42

[entryPoints]
  [entryPoints.EntryPoint0]
    address = "foobar"
    [entryPoints.EntryPoint0.transport]
      [entryPoints.EntryPoint0.transport.lifeCycle]
        requestAcceptGraceTimeout = 42
        graceTimeOut = 42
      [entryPoints.EntryPoint0.transport.respondingTimeouts]
        readTimeout = 42
        writeTimeout = 42
        idleTimeout = 42
    [entryPoints.EntryPoint0.proxyProtocol]
      insecure = true
      trustedIPs = ["foobar", "foobar"]
    [entryPoints.EntryPoint0.forwardedHeaders]
      insecure = true
      trustedIPs = ["foobar", "foobar"]
    [entryPoints.EntryPoint0.http]
      middlewares = ["foobar", "foobar"]
      [entryPoints.EntryPoint0.http.redirections]
        [entryPoints.EntryPoint0.http.redirections.entryPoint]
          to = "foobar"
          scheme = "foobar"
          permanent = true
          priority = 42
      [entryPoints.EntryPoint0.http.tls]
        options = "foobar"
        certResolver = "foobar"

        [[entryPoints.EntryPoint0.http.tls.domains]]
          main = "foobar"
          sans = ["foobar", "foobar"]

        [[entryPoints.EntryPoint0.http.tls.domains]]
          main = "foobar"
          sans = ["foobar", "foobar"]

[providers]
  providersThrottleDuration = 42
  [providers.file]
    directory = "foobar"
    watch = true
    filename = "foobar"
    debugLogGeneratedTemplate = true
[api]
  insecure = true
  dashboard = true
  debug = true

[metrics]
  [metrics.prometheus]
    buckets = [42.0, 42.0]
    addEntryPointsLabels = true
    addServicesLabels = true
    entryPoint = "foobar"
    manualRouting = true
[ping]
  entryPoint = "foobar"
  manualRouting = true

[log]
  level = "foobar"
  filePath = "foobar"
  format = "foobar"

[accessLog]
  filePath = "foobar"
  format = "foobar"
  bufferingSize = 42
  [accessLog.filters]
    statusCodes = ["foobar", "foobar"]
    retryAttempts = true
    minDuration = 42
  [accessLog.fields]
    defaultMode = "foobar"
    [accessLog.fields.names]
      name0 = "foobar"
      name1 = "foobar"
    [accessLog.fields.headers]
      defaultMode = "foobar"
      [accessLog.fields.headers.names]
        name0 = "foobar"
        name1 = "foobar"

[tracing]
  serviceName = "foobar"
  spanNameLimit = 42
  [tracing.jaeger]
    samplingServerURL = "foobar"
    samplingType = "foobar"
    samplingParam = 42.0
    localAgentHostPort = "foobar"
    gen128Bit = true
    propagation = "foobar"
    traceContextHeaderName = "foobar"
    [tracing.jaeger.collector]
      endpoint = "foobar"
      user = "foobar"
      password = "foobar"
[hostResolver]
  cnameFlattening = true
  resolvConfig = "foobar"
  resolvDepth = 42
```

## 动态配置

```toml
[http]
  [http.routers]
    [http.routers.Router0]
      entryPoints = ["foobar", "foobar"]
      middlewares = ["foobar", "foobar"]
      service = "foobar"
      rule = "foobar"
      priority = 42
      [http.routers.Router0.tls]
        options = "foobar"
        certResolver = "foobar"

        [[http.routers.Router0.tls.domains]]
          main = "foobar"
          sans = ["foobar", "foobar"]

        [[http.routers.Router0.tls.domains]]
          main = "foobar"
          sans = ["foobar", "foobar"]
  [http.services]
    [http.services.Service01]
      [http.services.Service01.loadBalancer]
        passHostHeader = true
        [http.services.Service01.loadBalancer.sticky]
          [http.services.Service01.loadBalancer.sticky.cookie]
            name = "foobar"
            secure = true
            httpOnly = true
            sameSite = "foobar"
        [[http.services.Service01.loadBalancer.servers]]
          url = "foobar"
        [[http.services.Service01.loadBalancer.servers]]
          url = "foobar"
        [http.services.Service01.loadBalancer.healthCheck]
          scheme = "foobar"
          path = "foobar"
          port = 42
          interval = "foobar"
          timeout = "foobar"
          hostname = "foobar"
          followRedirects = true
          [http.services.Service01.loadBalancer.healthCheck.headers]
            name0 = "foobar"
            name1 = "foobar"
        [http.services.Service01.loadBalancer.responseForwarding]
          flushInterval = "foobar"
  [http.middlewares]
    [http.middlewares.Middleware00]
      [http.middlewares.Middleware00.addPrefix]
        prefix = "foobar"
    [http.middlewares.Middleware01]
      [http.middlewares.Middleware01.basicAuth]
        users = ["foobar", "foobar"]
        usersFile = "foobar"
        realm = "foobar"
        removeHeader = true
        headerField = "foobar"
```

## providers.file

```toml
[http]
    [http.routers]
        [http.routers.Router0]
            entryPoints = ["web"]
            service = "foobar"
            rule = "Path(`/foo`)"
    [http.services]
        [http.services.foobar]
            [http.services.foobar.loadBalancer]
                [[http.services.foobar.loadBalancer.servers]]
                    url="http://127.0.0.1:8092"
                [[http.services.foobar.loadBalancer.servers]]
                    url="http://127.0.0.1:8091"
```

## 参考资料

- https://docs.traefik.io/reference/static-configuration/overview/

