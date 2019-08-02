---
layout: post
title: harbor authentication process
date: 2019-8-2 12:29:30
category: 技术
tags: Docker-registry Harbor
excerpt: harbor 认证流程
---

本文概要描述harbor token authentication流程

## harbor token认证流程

![](/public/img/v2-registry-auth.png)

```bash
docker login x.x.x.x
```

login日志：

```
May  6 15:21:24 x.x.x.x proxy[xxxx]: x.x.x.x - "GET /v2/ HTTP/1.1" 401 87 "-" "docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \x5C(linux\x5C))" 0.008 0.008 .
May  6 15:21:24 x.x.x.x proxy[xxxx]: x.x.x.x - "GET /service/token?account=xxx&client_id=docker&offline_token=true&service=harbor-registry HTTP/1.1" 200 893 "-" "docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \x5C(linux\x5C))" 0.025 0.025 .
May  6 15:21:24 x.x.x.x proxy[xxxx]: x.x.x.x - "GET /v2/ HTTP/1.1" 200 2 "-" "docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \x5C(linux\x5C))" 0.004 0.004 .
```

harbor core proxy入口：

```go
beego.Router("/v2/*", &controllers.RegistryProxy{}, "*:Handle")
...
// Handle is the only entrypoint for incoming requests, all requests must go through this func.
func (p *RegistryProxy) Handle() {
	req := p.Ctx.Request
	rw := p.Ctx.ResponseWriter
	proxy.Handle(rw, req)
}
...
// Init initialize the Proxy instance and handler chain.
func Init(urls ...string) error {
	var err error
	var registryURL string
	if len(urls) > 1 {
		return fmt.Errorf("the parm, urls should have only 0 or 1 elements")
	}
	if len(urls) == 0 {
		registryURL, err = config.RegistryURL()
		if err != nil {
			return err
		}
	} else {
		registryURL = urls[0]
	}
	targetURL, err := url.Parse(registryURL)
	if err != nil {
		return err
	}
	Proxy = httputil.NewSingleHostReverseProxy(targetURL)
	handlers = handlerChain{head: readonlyHandler{next: urlHandler{next: listReposHandler{next: contentTrustHandler{next: vulnerableHandler{next: Proxy}}}}}}
	return nil
}

// Handle handles the request.
func Handle(rw http.ResponseWriter, req *http.Request) {
	handlers.head.ServeHTTP(rw, req)
}
```

这里会经过5层handler，最后到达proxy，也即：`docker distribution`

* 1、访问`docker distribution`，获取`auth server`地址

```
May  6 15:21:24 x.x.x.x proxy[xxxx]: x.x.x.x - "GET /v2/ HTTP/1.1" 401 87 "-" "docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \x5C(linux\x5C))" 0.008 0.008 .
```

`/v2/`请求会直接到`docker distribution`，根据[v2 token](https://docs.docker.com/registry/spec/auth/token/)协议返回401，并在报头`Www-Authenticate`中返回token服务器地址进行后续认证，如下：

```
HTTP/1.1 401 Unauthorized
Content-Type: application/json; charset=utf-8
Docker-Distribution-Api-Version: registry/2.0
Www-Authenticate: Bearer realm="http://x.x.x.x/service/token",service="harbor-registry"
Date: Mon, 06 May 2019 07:57:46 GMT
Content-Length: 87

{"errors":[{"code":"UNAUTHORIZED","message":"authentication required","detail":null}]}
```

* 2、访问`token server`获取`token`

```
May  6 15:21:24 x.x.x.x proxy[xxxx]: x.x.x.x - "GET /service/token?account=xxx&client_id=docker&offline_token=true&service=harbor-registry HTTP/1.1" 200 893 "-" "docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \x5C(linux\x5C))" 0.025 0.025 .
```

```
GET /service/token?account=xxx&client_id=docker&offline_token=true&service=harbor-registry HTTP/1.1
Host: x.x.x.x
User-Agent: docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \(linux\))
Authorization: Basic xxxxxxxxxxxxxx
Accept-Encoding: gzip
Connection: close
```

根据路由：

```go
beego.Router("/service/token", &token.Handler{})
...
// Handler handles request on /service/token, which is the auth provider for registry.
type Handler struct {
	beego.Controller
}

// Get handles GET request, it checks the http header for user credentials
// and parse service and scope based on docker registry v2 standard,
// checkes the permission against local DB and generates jwt token.
func (h *Handler) Get() {
	request := h.Ctx.Request
	log.Debugf("URL for token request: %s", request.URL.String())
	service := h.GetString("service")
	tokenCreator, ok := creatorMap[service]
	if !ok {
		errMsg := fmt.Sprintf("Unable to handle service: %s", service)
		log.Errorf(errMsg)
		h.CustomAbort(http.StatusBadRequest, errMsg)
	}
	token, err := tokenCreator.Create(request)
	if err != nil {
		if _, ok := err.(*unauthorizedError); ok {
			h.CustomAbort(http.StatusUnauthorized, "")
		}
		log.Errorf("Unexpected error when creating the token, error: %v", err)
		h.CustomAbort(http.StatusInternalServerError, "")
	}
	h.Data["json"] = token
	h.ServeJSON()

}
...
const (
	// Notary service
	Notary = "harbor-notary"
	// Registry service
	Registry = "harbor-registry"
)

// InitCreators initialize the token creators for different services
func InitCreators() {
	creatorMap = make(map[string]Creator)
	registryFilterMap = map[string]accessFilter{
		"repository": &repositoryFilter{
			parser: &basicParser{},
		},
		"registry": &registryFilter{},
	}
	ext, err := config.ExtURL()
	if err != nil {
		log.Warningf("Failed to get ext url, err: %v, the token service will not be functional with notary requests", err)
	} else {
		notaryFilterMap = map[string]accessFilter{
			"repository": &repositoryFilter{
				parser: &endpointParser{
					endpoint: ext,
				},
			},
		}
		creatorMap[Notary] = &generalCreator{
			service:   Notary,
			filterMap: notaryFilterMap,
		}
	}

	creatorMap[Registry] = &generalCreator{
		service:   Registry,
		filterMap: registryFilterMap,
	}
}
```

根据service值:`harbor-registry`，会跳转到`generalCreator`:

```go
func (g generalCreator) Create(r *http.Request) (*models.Token, error) {
	var err error
	scopes := parseScopes(r.URL)
	log.Debugf("scopes: %v", scopes)

	ctx, err := filter.GetSecurityContext(r)
	if err != nil {
		return nil, fmt.Errorf("failed to  get security context from request")
	}

	pm, err := filter.GetProjectManager(r)
	if err != nil {
		return nil, fmt.Errorf("failed to  get project manager from request")
	}

	// for docker login
	if !ctx.IsAuthenticated() {
		if len(scopes) == 0 {
			return nil, &unauthorizedError{}
		}
	}
	access := GetResourceActions(scopes)
	err = filterAccess(access, ctx, pm, g.filterMap)
	if err != nil {
		return nil, err
	}
	return MakeToken(ctx.GetUsername(), g.service, access)
}
```

按照逻辑是会返回一个`token`，如下：

```
{"token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImtpZCI6IlBZWU86VEVXVTpWN0pIOjI2SlY6QVFUWjpMSkMzOlNYVko6WEdIQTozNEYyOjJMQVE6WlJNSzpaN1E2In0.eyJpc3MiOiJhdXRoLmRvY2tlci5jb20iLCJzdWIiOiJqbGhhd24iLCJhdWQiOiJyZWdpc3RyeS5kb2NrZXIuY29tIiwiZXhwIjoxNDE1Mzg3MzE1LCJuYmYiOjE0MTUzODcwMTUsImlhdCI6MTQxNTM4NzAxNSwianRpIjoidFlKQ08xYzZjbnl5N2tBbjBjN3JLUGdiVjFIMWJGd3MiLCJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6InNhbWFsYmEvbXktYXBwIiwiYWN0aW9ucyI6WyJwdXNoIl19XX0.QhflHPfbd6eVF4lM9bwYpFZIV0PfikbyXuLx959ykRTBpe3CYnzs6YBK8FToVb5R47920PVLrh8zuLzdCr9t3w", "expires_in": 3600,"issued_at": "2019-8-2T23:00:00Z"}
```

* 3、携带`token`访问`docker distribution`

```
May  6 15:21:24 x.x.x.x proxy[xxxx]: x.x.x.x - "GET /v2/ HTTP/1.1" 200 2 "-" "docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \x5C(linux\x5C))" 0.004 0.004 .
```

```
GET /v2/ HTTP/1.1
Host: x.x.x.x
User-Agent: docker/18.09.5 go/go1.10.8 git-commit/e8ff056 kernel/3.10.0-957.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.5 \(linux\))
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImtpZCI6IlBZWU86VEVXVTpWN0pIOjI2SlY6QVFUWjpMSkMzOlNYVko6WEdIQTozNEYyOjJMQVE6WlJNSzpaN1E2In0.eyJpc3MiOiJhdXRoLmRvY2tlci5jb20iLCJzdWIiOiJqbGhhd24iLCJhdWQiOiJyZWdpc3RyeS5kb2NrZXIuY29tIiwiZXhwIjoxNDE1Mzg3MzE1LCJuYmYiOjE0MTUzODcwMTUsImlhdCI6MTQxNTM4NzAxNSwianRpIjoidFlKQ08xYzZjbnl5N2tBbjBjN3JLUGdiVjFIMWJGd3MiLCJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6InNhbWFsYmEvbXktYXBwIiwiYWN0aW9ucyI6WyJwdXNoIl19XX0.QhflHPfbd6eVF4lM9bwYpFZIV0PfikbyXuLx959ykRTBpe3CYnzs6YBK8FToVb5R47920PVLrh8zuLzdCr9t3w
Accept-Encoding: gzip
Connection: close
```

## Refs 

* [token protocol](https://docs.docker.com/registry/spec/auth/token/)
