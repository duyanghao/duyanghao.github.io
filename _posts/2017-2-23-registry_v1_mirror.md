---
layout: post
title: Registry v1 Mirror
date: 2017-2-23 19:10:31
category: 技术
tags: Docker-registry Docker
excerpt: Registry v1 Mirror原理……
---

参考[文档](https://github.com/docker/docker-registry#mirroring-options)，[源码](https://github.com/docker/docker-registry)

下面以image Layer <GET>分析Mirror原理（local filesystem storage）……

private registry mirror configuration example:

```bash
# The `common' part is automatically included (and possibly overriden by all
# other flavors)
common: &common
    loglevel: debug
    mirroring:
        source: https://your_private_registry_domain
        #source_index: https://index.docker.io
        tags_cache_ttl: 172800  # seconds

local:
    <<: *common
    storage: local
    storage_path: /tmp/registry
```

layer请求：

```python
@app.route('/v1/images/<image_id>/layer', methods=['GET'])
@toolkit.requires_auth
@require_completion
@toolkit.valid_image_id
@set_cache_headers
@mirroring.source_lookup(cache=True, stream=True)
def get_image_layer(image_id, headers):
    try:
        bytes_range = None
        if store.supports_bytes_range:
            headers['Accept-Ranges'] = 'bytes'
            bytes_range = _parse_bytes_range()
        repository = toolkit.get_repository()
        if repository and store.is_private(*repository):
            if not toolkit.validate_parent_access(image_id):
                return toolkit.api_error('Image not found', 404)
        # If no auth token found, either standalone registry or privileged
        # access. In both cases, access is always "public".
        return _get_image_layer(image_id, headers, bytes_range)
    except exceptions.FileNotFoundError:
        return toolkit.api_error('Image not found', 404)

def _get_image_layer(image_id, headers=None, bytes_range=None):
    if headers is None:
        headers = {}

    headers['Content-Type'] = 'application/octet-stream'
    accel_uri_prefix = cfg.nginx_x_accel_redirect
    path = store.image_layer_path(image_id)
    if accel_uri_prefix:
        if store.scheme == 'file':
            accel_uri = '/'.join([accel_uri_prefix, path])
            headers['X-Accel-Redirect'] = accel_uri
            logger.debug('send accelerated {0} ({1})'.format(
                accel_uri, headers))
            return flask.Response('', headers=headers)
        else:
            logger.warn('nginx_x_accel_redirect config set,'
                        ' but storage is not LocalStorage')

    # If store allows us to just redirect the client let's do that, we'll
    # offload a lot of expensive I/O and get faster I/O
    if cfg.storage_redirect:
        try:
            content_redirect_url = store.content_redirect_url(path)
            if content_redirect_url:
                return flask.redirect(content_redirect_url, 302)
        except IOError as e:
            logger.debug(str(e))

    status = None
    layer_size = 0

    if not store.exists(path):
        raise exceptions.FileNotFoundError("Image layer absent from store")
    try:
        layer_size = store.get_size(path)
    except exceptions.FileNotFoundError:
        # XXX why would that fail given we know the layer exists?
        pass
    if bytes_range and bytes_range[1] == -1 and not layer_size == 0:
        bytes_range = (bytes_range[0], layer_size)

    if bytes_range:
        content_length = bytes_range[1] - bytes_range[0] + 1
        if not _valid_bytes_range(bytes_range):
            return flask.Response(status=416, headers=headers)
        status = 206
        content_range = (bytes_range[0], bytes_range[1], layer_size)
        headers['Content-Range'] = '{0}-{1}/{2}'.format(*content_range)
        headers['Content-Length'] = content_length
    elif layer_size > 0:
        headers['Content-Length'] = layer_size
    else:
        return flask.Response(status=416, headers=headers)
    return flask.Response(store.stream_read(path, bytes_range),
                          headers=headers, status=status)
```

注意装饰器：`@mirroring.source_lookup(cache=True, stream=True)`

```python
def source_lookup(cache=False, stream=False, index_route=False,
                  merge_results=False):
    def decorator(f):
        @functools.wraps(f)
        def wrapper(*args, **kwargs):
            mirroring_cfg = cfg.mirroring
            resp = f(*args, **kwargs)
            if not is_mirror():
                return resp
            source = mirroring_cfg.source
            if index_route and mirroring_cfg.source_index:
                source = mirroring_cfg.source_index
            logger.debug('Source provided, registry acts as mirror')
            if resp.status_code != 404 and not merge_results:
                logger.debug('Status code is not 404, no source '
                             'lookup required')
                return resp
            source_resp = lookup_source(
                flask.request.path, stream=stream, source=source
            )
            if not source_resp:
                return resp

            store = storage.load()

            headers = _response_headers(source_resp.headers)
            if index_route and 'x-docker-endpoints' in headers:
                headers['x-docker-endpoints'] = toolkit.get_endpoints()

            if not stream:
                logger.debug('JSON data found on source, writing response')
                resp_data = source_resp.content
                if merge_results:
                    mjson = json.loads(resp_data)
                    pjson = json.loads(resp.data)
                    for mr in mjson["results"]:
                        replaced = False
                        for pi, pr in enumerate(pjson["results"]):
                            if pr["name"] == mr["name"]:
                                pjson["results"][pi] = mr
                                replaced = True
                        if not replaced:
                            pjson["results"].extend([mr])
                    pjson['num_results'] = len(pjson["results"])
                    resp_data = json.dumps(pjson)
                if cache:
                    store_mirrored_data(
                        resp_data, flask.request.url_rule.rule, kwargs,
                        store
                    )
                return toolkit.response(
                    data=resp_data,
                    headers=headers,
                    raw=True
                )
            logger.debug('Layer data found on source, preparing to '
                         'stream response...')
            layer_path = store.image_layer_path(kwargs['image_id'])
            return _handle_mirrored_layer(source_resp, layer_path, store,
                                          headers)

        return wrapper
    return decorator
```

如果本地已经存在该Layer文件，则直接返回给docker：

```python
        def wrapper(*args, **kwargs):
            mirroring_cfg = cfg.mirroring
            resp = f(*args, **kwargs)
            if not is_mirror():
                return resp
            source = mirroring_cfg.source
            if index_route and mirroring_cfg.source_index:
                source = mirroring_cfg.source_index
            logger.debug('Source provided, registry acts as mirror')
            if resp.status_code != 404 and not merge_results:
                logger.debug('Status code is not 404, no source '
                             'lookup required')
                return resp
```

否则，向mirror source请求Layer数据，若后端存在该Layer，则返回response，否则返回None，如下：

```python
def lookup_source(path, stream=False, source=None):
    if not source:
        if not is_mirror():
            return
        source = cfg.mirroring.source
    source_url = '{0}{1}'.format(source, path)
    headers = {}
    for k, v in flask.request.headers.iteritems():
        if k.lower() != 'location' and k.lower() != 'host':
            headers[k] = v
    logger.debug('Request: GET {0}\nHeaders: {1}\nArgs: {2}'.format(
        source_url, headers, flask.request.args
    ))
    source_resp = requests.get(
        source_url,
        params=flask.request.args,
        headers=headers,
        cookies=flask.request.cookies,
        stream=stream
    )
    if source_resp.status_code != 200:
        logger.debug('Source responded to request with non-200'
                     ' status')
        logger.debug('Response: {0}\n{1}\n'.format(
            source_resp.status_code, source_resp.text
        ))
        return None

    return source_resp
```

若source不存在Layer,则向docker返回404 Not found，如下：

```python
            if not source_resp:
                return resp
```

在得到Layer数据后，若为二进制流：也即Layer数据，则调用：`_handle_mirrored_layer`：

```python
            if not stream:
                logger.debug('JSON data found on source, writing response')
                resp_data = source_resp.content
                if merge_results:
                    mjson = json.loads(resp_data)
                    pjson = json.loads(resp.data)
                    for mr in mjson["results"]:
                        replaced = False
                        for pi, pr in enumerate(pjson["results"]):
                            if pr["name"] == mr["name"]:
                                pjson["results"][pi] = mr
                                replaced = True
                        if not replaced:
                            pjson["results"].extend([mr])
                    pjson['num_results'] = len(pjson["results"])
                    resp_data = json.dumps(pjson)
                if cache:
                    store_mirrored_data(
                        resp_data, flask.request.url_rule.rule, kwargs,
                        store
                    )
                return toolkit.response(
                    data=resp_data,
                    headers=headers,
                    raw=True
                )
            logger.debug('Layer data found on source, preparing to '
                         'stream response...')
            layer_path = store.image_layer_path(kwargs['image_id'])
            return _handle_mirrored_layer(source_resp, layer_path, store,
                                          headers)
```

`_handle_mirrored_layer`函数将Layer数据返回给docker并写入到本地文件系统，如下：

```python
def _handle_mirrored_layer(source_resp, layer_path, store, headers):
    sr = toolkit.SocketReader(source_resp)
    tmp, hndlr = storage.temp_store_handler()
    sr.add_handler(hndlr)

    def generate():
        for chunk in sr.iterate(store.buffer_size):
            yield chunk
        # FIXME: this could be done outside of the request context
        tmp.seek(0)
        store.stream_write(layer_path, tmp)
        tmp.close()
    return flask.Response(generate(), headers=dict(headers))

def stream_write(self, path, fp):
    # Size is mandatory
    path = self._init_path(path, create=True)
    with open(path, mode='wb') as f:
        try:
            while True:
                buf = fp.read(self.buffer_size)
                if not buf:
                    break
                f.write(buf)
        except IOError:
            pass
```

在本地文件系统会存在如下文件目录：

```bash
/data/docker-registry/images/xxxxxxxxxxxxxxxxxid/
|-- ancestry
|-- json
|-- layer
``` 

若为layer流数据，则处理如上；若为json或者ancestry json数据请求，如下：

```python
@app.route('/v1/images/<image_id>/json', methods=['GET'])
@toolkit.requires_auth
@toolkit.valid_image_id
@require_completion
@set_cache_headers
@mirroring.source_lookup(cache=True, stream=False)
def get_image_json(image_id, headers):
    try:
        repository = toolkit.get_repository()
        if repository and store.is_private(*repository):
            if not toolkit.validate_parent_access(image_id):
                return toolkit.api_error('Image not found', 404)
        # If no auth token found, either standalone registry or privileged
        # access. In both cases, access is always "public".
        return _get_image_json(image_id, headers)
    except exceptions.FileNotFoundError:
        return toolkit.api_error('Image not found', 404)

def _get_image_json(image_id, headers=None):
    if headers is None:
        headers = {}
    data = store.get_content(store.image_json_path(image_id))
    try:
        size = store.get_size(store.image_layer_path(image_id))
        headers['X-Docker-Size'] = str(size)
    except exceptions.FileNotFoundError:
        pass
    try:
        csums = load_checksums(image_id)
        headers['X-Docker-Checksum-Payload'] = csums
    except exceptions.FileNotFoundError:
        pass
    return toolkit.response(data, headers=headers, raw=True)

@app.route('/v1/images/<image_id>/ancestry', methods=['GET'])
@toolkit.requires_auth
@toolkit.valid_image_id
@require_completion
@set_cache_headers
@mirroring.source_lookup(cache=True, stream=False)
def get_image_ancestry(image_id, headers):
    ancestry_path = store.image_ancestry_path(image_id)
    try:
        # Note(dmp): unicode patch
        data = store.get_json(ancestry_path)
    except exceptions.FileNotFoundError:
        return toolkit.api_error('Image not found', 404)
    return toolkit.response(data, headers=headers)
```

则请求处理如下：

```python
            if not stream:
                logger.debug('JSON data found on source, writing response')
                resp_data = source_resp.content
                if merge_results:
                    mjson = json.loads(resp_data)
                    pjson = json.loads(resp.data)
                    for mr in mjson["results"]:
                        replaced = False
                        for pi, pr in enumerate(pjson["results"]):
                            if pr["name"] == mr["name"]:
                                pjson["results"][pi] = mr
                                replaced = True
                        if not replaced:
                            pjson["results"].extend([mr])
                    pjson['num_results'] = len(pjson["results"])
                    resp_data = json.dumps(pjson)
                if cache:
                    store_mirrored_data(
                        resp_data, flask.request.url_rule.rule, kwargs,
                        store
                    )
                return toolkit.response(
                    data=resp_data,
                    headers=headers,
                    raw=True
                )
```

```python
def store_mirrored_data(data, endpoint, args, store):
    logger.debug('Endpoint: {0}'.format(endpoint))
    path_method, arglist = ({
        '/v1/images/<image_id>/json': ('image_json_path', ('image_id',)),
        '/v1/images/<image_id>/ancestry': (
            'image_ancestry_path', ('image_id',)
        ),
        '/v1/repositories/<path:repository>/json': (
            'registry_json_path', ('namespace', 'repository')
        ),
    }).get(endpoint, (None, None))
    if not path_method:
        return
    logger.debug('Path method: {0}'.format(path_method))
    pm_args = {}
    for arg in arglist:
        pm_args[arg] = args[arg]
    logger.debug('Path method args: {0}'.format(pm_args))
    storage_path = getattr(store, path_method)(**pm_args)
    logger.debug('Storage path: {0}'.format(storage_path))
    store.put_content(storage_path, data)

@lru.set
def put_content(self, path, content):
    path = self._init_path(path, create=True)
    with open(path, mode='wb') as f:
        f.write(content)
    return path
```

日志如下：

```bash
2017-02-08 17:28:33,799 DEBUG: JSON data found on source, writing response
2017-02-08 17:28:33,799 DEBUG: Endpoint: /v1/images/<image_id>/ancestry
2017-02-08 17:28:33,799 DEBUG: Path method: image_ancestry_path
2017-02-08 17:28:33,800 DEBUG: Path method args: {'image_id': u'xxxxxxxxxxxxxxxxxid'}
2017-02-08 17:28:33,800 DEBUG: Storage path: images/xxxxxxxxxxxxxxxxxid/ancestry
```