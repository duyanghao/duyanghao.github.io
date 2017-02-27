---
layout: post
title: Registry v1 Pull Flow
date: 2017-2-27 14:10:31
category: 技术
tags: Docker-registry Docker
excerpt: Registry v1 Pull Flow……
---

本文从[源码](https://github.com/docker/docker-registry)角度描述Registry v1 Pull Flow……

如下是docker pull流程：

```
Pulling repository xxxx/library/centos
333333333333: Download complete 
111111111111: Download complete 
222222222222: Download complete 
Status: Downloaded newer image for xxxx/library/centos:latest

"GET /v1/_ping HTTP/1.1"
"GET /v1/repositories/library/centos/images HTTP/1.1"
"GET /v1/repositories/library/centos/tags HTTP/1.1"
"GET /v1/images/3333333333333333333333333333333333333333333333333333333333333333/ancestry HTTP/1.1"
"GET /v1/images/1111111111111111111111111111111111111111111111111111111111111111/json HTTP/1.1"
"GET /v1/images/1111111111111111111111111111111111111111111111111111111111111111/layer HTTP/1.1"
"GET /v1/images/2222222222222222222222222222222222222222222222222222222222222222/json HTTP/1.1"
"GET /v1/images/2222222222222222222222222222222222222222222222222222222222222222/layer HTTP/1.1"
"GET /v1/images/3333333333333333333333333333333333333333333333333333333333333333/json HTTP/1.1"
"GET /v1/images/3333333333333333333333333333333333333333333333333333333333333333/layer HTTP/1.1"
```

## `GET /v1/_ping`

请求如下：

```python
@app.route('/_ping')
@app.route('/v1/_ping')
def ping():
    headers = {
        'X-Docker-Registry-Standalone': 'mirror' if mirroring.is_mirror()
                                        else (cfg.standalone is True)
    }
    infos = {}
    if cfg.debug:
        # Versions
        versions = infos['versions'] = {}
        headers['X-Docker-Registry-Config'] = cfg.flavor

        for name, module in sys.modules.items():
            if name.startswith('_'):
                continue
            try:
                version = module.__version__
            except AttributeError:
                continue
            versions[name] = version
        versions['python'] = sys.version

        # Hosts infos
        infos['host'] = platform.uname()
        infos['launch'] = sys.argv

    return toolkit.response(infos, headers=headers)
```

## `GET /v1/repositories/library/centos/images`

请求结果如下：

```bash
curl http://xxxx/v1/repositories/library/centos/images
[{"id": "3333333333333333333333333333333333333333333333333333333333333333"}, {"id": "2222222222222222222222222222222222222222222222222222222222222222"}, {"id": "1111111111111111111111111111111111111111111111111111111111111111"}]
```

对应代码：

```python
@app.route('/v1/repositories/<path:repository>/images', methods=['GET'])
@toolkit.parse_repository_name
@toolkit.requires_auth
@mirroring.source_lookup(index_route=True)
def get_repository_images(namespace, repository):
    data = None
    try:
        path = store.index_images_path(namespace, repository)
        data = store.get_content(path)
    except exceptions.FileNotFoundError:
        return toolkit.api_error('images not found', 404)
    headers = generate_headers(namespace, repository, 'read')
    return toolkit.response(data, 200, headers, True)

@filter_args
    def index_images_path(self, namespace, repository):
        repository_path = self._repository_path(
            namespace=namespace, repository=repository)
        return '{0}/_index_images'.format(repository_path)
```

也即请求对应repositories/library/centos/`_index_images`文件，若为mirror，则执行如下`lookup_source`函数：

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

执行流程（local filesystem）：检查本地有无`_index_images`文件，若不存在则从source请求该文件（事实上就是从source请求该文件，因为`cache`为`False`）

## `GET /v1/repositories/library/centos/tags`

请求结果如下：

```bash
curl http://xxxx/v1/repositories/library/centos/tags

{"latest": "3333333333333333333333333333333333333333333333333333333333333333"}
```

对应代码:

```python
@app.route('/v1/repositories/<path:repository>/tags', methods=['GET'])
@toolkit.parse_repository_name
@toolkit.requires_auth
@mirroring.source_lookup_tag
def _get_tags(namespace, repository):
    logger.debug("[get_tags] namespace={0}; repository={1}".format(namespace,
                 repository))
    try:
        data = get_tags(namespace=namespace, repository=repository)
    except exceptions.FileNotFoundError:
        return toolkit.api_error('Repository not found', 404)
    return toolkit.response(data)

def get_tags(namespace, repository):
    tag_path = store.tag_path(namespace, repository)
    greenlets = {}
    for fname in store.list_directory(tag_path):
        full_tag_name = fname.split('/').pop()
        if not full_tag_name.startswith('tag_'):
            continue
        tag_name = full_tag_name[4:]
        greenlets[tag_name] = gevent.spawn(
            store.get_content,
            store.tag_path(namespace, repository, tag_name),
        )
    gevent.joinall(greenlets.values())
    return dict((k, g.value) for (k, g) in greenlets.items())

@filter_args
def tag_path(self, namespace, repository, tagname=None):
    repository_path = self._repository_path(
        namespace=namespace, repository=repository)
    if not tagname:
        return repository_path
    return '{0}/tag_{1}'.format(repository_path, tagname)

# Useful if we want to change those locations later without rewriting
# the code which uses Storage
repositories = 'repositories'
images = 'images'

def _repository_path(self, namespace, repository):
    return '{0}/{1}/{2}'.format(
        self.repositories, namespace, repository)
```

执行流程：
1、列举出目录repositories/library/centos下的所有文件
2、请求每个以`tag_`开头的文件内容
3、将结果合并

若为mirror，则执行如下`lookup_source`函数：

```python
def source_lookup_tag(f):
    @functools.wraps(f)
    def wrapper(namespace, repository, *args, **kwargs):
        mirroring_cfg = cfg.mirroring
        resp = f(namespace, repository, *args, **kwargs)
        if not is_mirror():
            return resp
        source = mirroring_cfg.source
        tags_cache_ttl = mirroring_cfg.tags_cache_ttl

        if resp.status_code != 404:
            logger.debug('Status code is not 404, no source '
                         'lookup required')
            return resp

        if not cache.redis_conn:
            # No tags cache, just return
            logger.warning('mirroring: Tags cache is disabled, please set a '
                           'valid `cache\' directive in the config.')
            source_resp = lookup_source(
                flask.request.path, stream=False, source=source
            )
            if not source_resp:
                return resp

            headers = _response_headers(source_resp.headers)
            return toolkit.response(data=source_resp.content, headers=headers,
                                    raw=True)

        store = storage.load()
        request_path = flask.request.path

        if request_path.endswith('/tags'):
            # client GETs a list of tags
            tag_path = store.tag_path(namespace, repository)
        else:
            # client GETs a single tag
            tag_path = store.tag_path(namespace, repository, kwargs['tag'])

        try:
            data = cache.redis_conn.get('{0}:{1}'.format(
                cache.cache_prefix, tag_path
            ))
        except cache.redis.exceptions.ConnectionError as e:
            data = None
            logger.warning("Diff queue: Redis connection error: {0}".format(
                e
            ))

        if data is not None:
            return toolkit.response(data=data, raw=True)
        source_resp = lookup_source(
            flask.request.path, stream=False, source=source
        )
        if not source_resp:
            return resp
        data = source_resp.content
        headers = _response_headers(source_resp.headers)
        try:
            cache.redis_conn.setex('{0}:{1}'.format(
                cache.cache_prefix, tag_path
            ), tags_cache_ttl, data)
        except cache.redis.exceptions.ConnectionError as e:
            logger.warning("Diff queue: Redis connection error: {0}".format(
                e
            ))

        return toolkit.response(data=data, headers=headers,
                                raw=True)
    return wrapper

def list_directory(self, path=None):
    prefix = ''
    if path:
        prefix = '%s/' % path
    path = self._init_path(path)
    exists = False
    try:
        for d in os.listdir(path):
            exists = True
            yield prefix + d
    except Exception:
        pass
    if not exists:
        raise exceptions.FileNotFoundError('%s is not there' % path)
```

执行流程（local filesystem）：检查本地有无repositories/library/centos目录，若不存在则向source发出该请求（事实上就是从source发出该请求，`cache.redis_conn`为`None`）

## `GET /v1/images/3333333333333333333333333333333333333333333333333333333333333333/ancestry`

请求结果如下：

```bash
curl http://xxxx/v1/images/3333333333333333333333333333333333333333333333333333333333333333/ancestry
["3333333333333333333333333333333333333333333333333333333333333333", "2222222222222222222222222222222222222222222222222222222222222222", "1111111111111111111111111111111111111111111111111111111111111111"]
```

对应代码:

```python
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

@filter_args
def image_ancestry_path(self, image_id):
    return '{0}/{1}/ancestry'.format(self.images, image_id)
```

对应文件：images/3333333333333333333333333333333333333333333333333333333333333333/ancestry，若为mirror，则执行：

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

执行流程（local filesystem）：
1、检查本地是否存在images/3333333333333333333333333333333333333333333333333333333333333333/ancestry
2、若不存在，则向source发出请求获取文件内容
3、将获取的文件内容存储到本地文件系统

## `GET /v1/images/1111111111111111111111111111111111111111111111111111111111111111/json`

请求结果如下：

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

@filter_args
def image_json_path(self, image_id):
    return '{0}/{1}/json'.format(self.images, image_id)

@filter_args
def image_layer_path(self, image_id):
    return '{0}/{1}/layer'.format(self.images, image_id)

def load_checksums(image_id):
    checksum_path = store.image_checksum_path(image_id)
    data = store.get_content(checksum_path)
    try:
        # Note(dmp): unicode patch NOT applied here
        return json.loads(data)
    except ValueError:
        # NOTE(sam): For backward compatibility only, existing data may not be
        # a valid json but a simple string.
        return [data]

@filter_args
def image_checksum_path(self, image_id):
    return '{0}/{1}/_checksum'.format(self.images, image_id)
```

执行流程：
1、获取images/1111111111111111111111111111111111111111111111111111111111111111/json文件内容
2、获取images/1111111111111111111111111111111111111111111111111111111111111111/layer文件大小，并添加到`X-Docker-Size`回应报头字段
3、获取images/1111111111111111111111111111111111111111111111111111111111111111/_checksum文件内容，并添加到`X-Docker-Checksum-Payload`回应报头字段

若为mirror，则执行：

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

执行流程（local filesystem）：
1、检查本地是否存在images/3333333333333333333333333333333333333333333333333333333333333333/json
2、若不存在，则向source发出请求获取文件内容
3、将获取的文件内容存储到本地文件系统

## `GET /v1/images/1111111111111111111111111111111111111111111111111111111111111111/layer`

代码如下：

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


@filter_args
def image_layer_path(self, image_id):
    return '{0}/{1}/layer'.format(self.images, image_id)
```

获取文件images/1111111111111111111111111111111111111111111111111111111111111111/layer内容

若为mirror，则执行：

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

执行流程（local filesystem）：
1、检查本地是否存在images/3333333333333333333333333333333333333333333333333333333333333333/layer
2、若不存在，则向source发出请求获取文件内容
3、将获取的文件内容存储到本地文件系统

## 补充
images/2222222222222222222222222222222222222222222222222222222222222222/json
images/2222222222222222222222222222222222222222222222222222222222222222/layer
images/3333333333333333333333333333333333333333333333333333333333333333/json
images/3333333333333333333333333333333333333333333333333333333333333333/layer
请求过程如:images/1111111111111111111111111111111111111111111111111111111111111111