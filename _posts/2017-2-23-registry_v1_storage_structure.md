---
layout: post
title: Registry v1 Storage Structure
date: 2017-2-23 20:10:31
category: 技术
tags: Docker-registry Docker
excerpt: Registry v1 Storage Structure……
---

本文从源码角度描述Registry v1 Storage Structure，如下是目录结构：

```
/data/docker-registry-local/
|-- images
|   |-- xxxxxxxxxxxxxxxxxxxxxxid
|   |   |-- _checksum
|   |   |-- ancestry
|   |   |-- json
|   |   `-- layer
|   |-- xxxxxxxxxxxxxxxxxxxxxxid
|   |   |-- _checksum
|   |   |-- ancestry
|   |   |-- json
|   |   `-- layer
|   |-- xxxxxxxxxxxxxxxxxxxxxxid
|   |   |-- _checksum
|   |   |-- ancestry
|   |   |-- json
|   |   `-- layer
`-- repositories
    `-- duyanghao
        `-- centos
            |-- _index_images
            |-- tag_latest
            |-- taglatest_json
            `-- json
```

下面分别说明每个文件含义：

1、repositories/namespace/repository/`tag_xxx` —— `tag_latest`

```python
@app.route('/v1/repositories/<path:repository>/tags/<tag>',
           methods=['PUT'])
@toolkit.parse_repository_name
@toolkit.requires_auth
def put_tag(namespace, repository, tag):
    logger.debug("[put_tag] namespace={0}; repository={1}; tag={2}".format(
                 namespace, repository, tag))
    if not RE_VALID_TAG.match(tag):
        return toolkit.api_error('Invalid tag name (must match {0})'.format(
            RE_VALID_TAG.pattern
        ))
    data = None
    try:
        # Note(dmp): unicode patch
        data = json.loads(flask.request.data.decode('utf8'))
    except ValueError:
        pass
    if not data or not isinstance(data, basestring):
        return toolkit.api_error('Invalid data')
    if not store.exists(store.image_json_path(data)):
        return toolkit.api_error('Image not found', 404)
    store.put_content(store.tag_path(namespace, repository, tag), data)
    sender = flask.current_app._get_current_object()
    signals.tag_created.send(sender, namespace=namespace,
                             repository=repository, tag=tag, value=data)
    # Write some meta-data about the repos
    ua = flask.request.headers.get('user-agent', '')
    data = create_tag_json(user_agent=ua)
    json_path = store.repository_tag_json_path(namespace, repository, tag)
    store.put_content(json_path, data)
    if tag == "latest":  # TODO(dustinlacewell) : deprecate this for v2
        json_path = store.repository_json_path(namespace, repository)
        store.put_content(json_path, data)
    return toolkit.response()

@filter_args
def tag_path(self, namespace, repository, tagname=None):
    repository_path = self._repository_path(
        namespace=namespace, repository=repository)
    if not tagname:
        return repository_path
    return '{0}/tag_{1}'.format(repository_path, tagname)

def _repository_path(self, namespace, repository):
    return '{0}/{1}/{2}'.format(
        self.repositories, namespace, repository)

# Useful if we want to change those locations later without rewriting
# the code which uses Storage
repositories = 'repositories'
images = 'images'
```

文件内容为该tag:xxx对应镜像imageID，如下：

```python
    data = None
    try:
        # Note(dmp): unicode patch
        data = json.loads(flask.request.data.decode('utf8'))
    except ValueError:
        pass
    if not data or not isinstance(data, basestring):
        return toolkit.api_error('Invalid data')
    if not store.exists(store.image_json_path(data)):
        return toolkit.api_error('Image not found', 404)
    store.put_content(store.tag_path(namespace, repository, tag), data)
    sender = flask.current_app._get_current_object()
    signals.tag_created.send(sender, namespace=namespace,
                             repository=repository, tag=tag, value=data)
```

2、repositories/namespace/repository/`tagxxx_json` —— `taglatest_json`

文件内容为tag:xxx对应镜像的`meta-data`信息，文件内容如下：

>>{"kernel": "3.10.5-3.el6.x86_64", "arch": "amd64", "docker_go_version": "go1.3.3", "last_update": 1487028653, "docker_version": "1.4", "os": "linux"}


对应代码如下：

```python
    # Write some meta-data about the repos
    ua = flask.request.headers.get('user-agent', '')
    data = create_tag_json(user_agent=ua)
    json_path = store.repository_tag_json_path(namespace, repository, tag)
    store.put_content(json_path, data)
    if tag == "latest":  # TODO(dustinlacewell) : deprecate this for v2
        json_path = store.repository_json_path(namespace, repository)
        store.put_content(json_path, data)
    return toolkit.response()
```

```python
def create_tag_json(user_agent):
    props = {
        'last_update': int(time.mktime(datetime.datetime.utcnow().timetuple()))
    }
    ua = dict(RE_USER_AGENT.findall(user_agent))
    if 'docker' in ua:
        props['docker_version'] = ua['docker']
    if 'go' in ua:
        props['docker_go_version'] = ua['go']
    for k in ['arch', 'kernel', 'os']:
        if k in ua:
            props[k] = ua[k].lower()
    return json.dumps(props)

@filter_args
    def repository_tag_json_path(self, namespace, repository, tag):
        repository_path = self._repository_path(
            namespace=namespace, repository=repository)
        return '{0}/tag{1}_json'.format(repository_path, tag)
```

3、repositories/namespace/repository/json —— json

文件内容为tag:`latest`对应镜像的`meta-data`信息，文件内容如下：

>>{"kernel": "3.10.5-3.el6.x86_64", "arch": "amd64", "docker_go_version": "go1.3.3", "last_update": 1487028653, "docker_version": "1.4", "os": "linux"}

对应代码如下：

```python
    # Write some meta-data about the repos
    ua = flask.request.headers.get('user-agent', '')
    data = create_tag_json(user_agent=ua)
    json_path = store.repository_tag_json_path(namespace, repository, tag)
    store.put_content(json_path, data)
    if tag == "latest":  # TODO(dustinlacewell) : deprecate this for v2
        json_path = store.repository_json_path(namespace, repository)
        store.put_content(json_path, data)
```

```python
@filter_args
def repository_json_path(self, namespace, repository):
    repository_path = self._repository_path(
        namespace=namespace, repository=repository)
    return '{0}/json'.format(repository_path)
```

4、repositories/namespace/repository/`_index_images` —— `_index_images`

文件内容为：`namespace/repository`所有tags对应镜像的layer id集合（还需通过docker源码验证），文件内容如下：

>>[{"id": "xxxxxxxxxxxxxxxxxxxxxxid"}, {"id": "xxxxxxxxxxxxxxxxxxxxxxid"},{"id": "xxxxxxxxxxxxxxxxxxxxxxid"}]

对应代码如下：

```python
@app.route('/v1/repositories/<path:repository>', methods=['PUT'])
@app.route('/v1/repositories/<path:repository>/images',
           defaults={'images': True},
           methods=['PUT'])
@toolkit.parse_repository_name
@toolkit.requires_auth
def put_repository(namespace, repository, images=False):
    data = None
    try:
        # Note(dmp): unicode patch
        data = json.loads(flask.request.data.decode('utf8'))
    except ValueError:
        return toolkit.api_error('Error Decoding JSON', 400)
    if not isinstance(data, list):
        return toolkit.api_error('Invalid data')
    update_index_images(namespace, repository, flask.request.data)
    headers = generate_headers(namespace, repository, 'write')
    code = 204 if images is True else 200
    return toolkit.response('', code, headers)

def update_index_images(namespace, repository, data_arg):
    path = store.index_images_path(namespace, repository)
    sender = flask.current_app._get_current_object()
    try:
        images = {}
        # Note(dmp): unicode patch
        data = json.loads(data_arg.decode('utf8')) + store.get_json(path)
        for i in data:
            iid = i['id']
            if iid in images and 'checksum' in images[iid]:
                continue
            i_data = {'id': iid}
            for key in ['checksum']:
                if key in i:
                    i_data[key] = i[key]
            images[iid] = i_data
        data = images.values()
        # Note(dmp): unicode patch
        store.put_json(path, data)
        signals.repository_updated.send(
            sender, namespace=namespace, repository=repository, value=data)
    except exceptions.FileNotFoundError:
        signals.repository_created.send(
            sender, namespace=namespace, repository=repository,
            # Note(dmp): unicode patch
            value=json.loads(data_arg.decode('utf8')))
        store.put_content(path, data_arg)

@filter_args
def index_images_path(self, namespace, repository):
    repository_path = self._repository_path(
        namespace=namespace, repository=repository)
    return '{0}/_index_images'.format(repository_path)
```

5、images/image_id/json —— images/xxxxxxxxxxxxxxxxxxxxxxid/json

文件内容是layer对应的元数据，内容如下：

>>{"id":"xxxxxxxxxxxxxxxxxxxxxxid","comment":"Imported from -","created":"2015-02-09T09:44:28.227848546Z","container_config":{"Hostname":"","Domainname":"","User":"","Memory":0,"MemorySwap":0,"CpuShares":0,"Cpuset":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"PortSpecs":null,"ExposedPorts":null,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":null,"Cmd":null,"Image":"","Volumes":null,"WorkingDir":"","Entrypoint":null,"NetworkDisabled":false,"OnBuild":null,"SecurityOpt":null},"docker_version":"1.3.0","architecture":"amd64","os":"linux","Size":1373474869}

对应代码如下：

```python
@app.route('/v1/images/<image_id>/json', methods=['PUT'])
@toolkit.requires_auth
@toolkit.valid_image_id
def put_image_json(image_id):
    data = None
    try:
        # Note(dmp): unicode patch
        data = json.loads(flask.request.data.decode('utf8'))
    except ValueError:
        pass
    if not data or not isinstance(data, dict):
        return toolkit.api_error('Invalid JSON')
    if 'id' not in data:
        return toolkit.api_error('Missing key `id\' in JSON')
    if image_id != data['id']:
        return toolkit.api_error('JSON data contains invalid id')
    if check_images_list(image_id) is False:
        return toolkit.api_error('This image does not belong to the '
                                 'repository')
    parent_id = data.get('parent')
    if parent_id and not store.exists(store.image_json_path(data['parent'])):
        return toolkit.api_error('Image depends on a non existing parent')
    elif parent_id and not toolkit.validate_parent_access(parent_id):
        return toolkit.api_error('Image depends on an unauthorized parent')
    json_path = store.image_json_path(image_id)
    mark_path = store.image_mark_path(image_id)
    if store.exists(json_path) and not store.exists(mark_path):
        return toolkit.api_error('Image already exists', 409)

    sender = flask.current_app._get_current_object()
    signal_result = signals.before_put_image_json.send(sender, image_json=data)
    for result_pair in signal_result:
        if result_pair[1] is not None:
            return toolkit.api_error(result_pair[1])

    # If we reach that point, it means that this is a new image or a retry
    # on a failed push
    store.put_content(mark_path, 'true')
    # We cleanup any old checksum in case it's a retry after a fail
    try:
        store.remove(store.image_checksum_path(image_id))
    except Exception:
        pass
    store.put_content(json_path, flask.request.data)
    layers.generate_ancestry(image_id, parent_id)
    return toolkit.response()

@filter_args
def image_json_path(self, image_id):
    return '{0}/{1}/json'.format(self.images, image_id)

def generate_ancestry(image_id, parent_id=None):
    if not parent_id:
        store.put_content(store.image_ancestry_path(image_id),
                          json.dumps([image_id]))
        return
    # Note(dmp): unicode patch
    data = store.get_json(store.image_ancestry_path(parent_id))
    data.insert(0, image_id)
    # Note(dmp): unicode patch
    store.put_json(store.image_ancestry_path(image_id), data)

@filter_args
def image_mark_path(self, image_id):
    return '{0}/{1}/_inprogress'.format(self.images, image_id)
```

处理逻辑为：

step1：检测json数据的有效性：

* json文件中必须有id，且为该image_id
* 若有parent字段，则parent对应layer的json文件必须存在

step2：创建_inprogress文件

step3：删除_checksum文件

step4：写json文件

step5：创建ancestry文件，若为base镜像则直接创建（文件内容即为该`image_id`），否则获取parent Layer ancestry文件，然后将该`image_id`插入到文件最开始（__也即ancestry文件表示了镜像的layer继承id，并且按照top-to-bottom顺序排序__）

6、images/image_id/ancestry —— images/xxxxxxxxxxxxxxxxxxxxxxid/ancestry

ancestry文件表示了镜像的layer继承id，并且按照top-to-bottom顺序排序，文件内容如下：

>>["xxxxxxxxxxxxxxxxxxxxxxid", "xxxxxxxxxxxxxxxxxxxxxxid", "xxxxxxxxxxxxxxxxxxxxxxid"]

7、images/image_id/layer —— images/xxxxxxxxxxxxxxxxxxxxxxid/layer

文件内容为id=image_id的layer数据（gzip compressed data），代码如下：

```python
@app.route('/v1/images/<image_id>/layer', methods=['PUT'])
@toolkit.requires_auth
@toolkit.valid_image_id
def put_image_layer(image_id):
    client_version = toolkit.docker_client_version()
    if client_version and client_version < (0, 10):
        return toolkit.api_error(
            'This endpoint does not support Docker daemons older than 0.10',
            412)
    try:
        json_data = store.get_content(store.image_json_path(image_id))
    except exceptions.FileNotFoundError:
        return toolkit.api_error('Image not found', 404)
    layer_path = store.image_layer_path(image_id)
    mark_path = store.image_mark_path(image_id)
    if store.exists(layer_path) and not store.exists(mark_path):
        return toolkit.api_error('Image already exists', 409)
    input_stream = flask.request.stream
    if flask.request.headers.get('transfer-encoding') == 'chunked':
        # Careful, might work only with WSGI servers supporting chunked
        # encoding (Gunicorn)
        input_stream = flask.request.environ['wsgi.input']
    # compute checksums
    csums = []
    sr = toolkit.SocketReader(input_stream)
    h, sum_hndlr = checksums.simple_checksum_handler(json_data)
    sr.add_handler(sum_hndlr)
    store.stream_write(layer_path, sr)
    csums.append('sha256:{0}'.format(h.hexdigest()))

    # We store the computed checksums for a later check
    save_checksums(image_id, csums)
    return toolkit.response()

def save_checksums(image_id, checksums):
    for checksum in checksums:
        checksum_parts = checksum.split(':')
        if len(checksum_parts) != 2:
            return 'Invalid checksum format'
    # We store the checksum
    checksum_path = store.image_checksum_path(image_id)
    store.put_content(checksum_path, json.dumps(checksums))

@filter_args
def image_checksum_path(self, image_id):
    return '{0}/{1}/_checksum'.format(self.images, image_id)

@filter_args
def image_json_path(self, image_id):
    return '{0}/{1}/json'.format(self.images, image_id)

def simple_checksum_handler(json_data):
    h = hashlib.sha256(json_data + '\n')

    def fn(buf):
        h.update(buf)
    return h, fn
```

对images/image_id/json进行sha256 hash，生成`_checksum`文件

8、images/image_id/`_checksum` —— images/xxxxxxxxxxxxxxxxxxxxxxid/`_checksum`

文件内容为id=image_id的json文件的sha256 hash值，内容如下：

>>["sha256:xxxxxxxxxxxxxxxxxxxxxxxx"]

对应代码如下：

```python
@app.route('/v1/images/<image_id>/checksum', methods=['PUT'])
@toolkit.requires_auth
@toolkit.valid_image_id
def put_image_checksum(image_id):
    checksum = flask.request.headers.get('X-Docker-Checksum-Payload')
    if checksum is None:
        return toolkit.api_error(
            ('X-Docker-Checksum-Payload not set.  If you are using the Docker '
             'daemon, you should upgrade to version 0.10 or later'),
            412)
    if not checksum:
        return toolkit.api_error('Missing Image\'s checksum')
    if not store.exists(store.image_json_path(image_id)):
        return toolkit.api_error('Image not found', 404)
    mark_path = store.image_mark_path(image_id)
    if not store.exists(mark_path):
        return toolkit.api_error('Cannot set this image checksum', 409)
    checksums = load_checksums(image_id)
    if checksum not in checksums:
        logger.debug('put_image_checksum: Wrong checksum. '
                     'Provided: {0}; Expected: {1}'.format(
                         checksum, checksums))
        return toolkit.api_error('Checksum mismatch')
    # Checksum is ok, we remove the marker
    store.remove(mark_path)
    # We trigger a task on the diff worker if it's running
    layers.enqueue_diff(image_id)
    return toolkit.response()

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
```