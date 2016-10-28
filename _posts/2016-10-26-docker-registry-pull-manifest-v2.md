---
layout: post
title: docker pull分析
date: 2016-10-26 17:36:11
category: 技术
tags: Docker-registry Docker Docker-Manifest-v2
excerpt: Image Manifest Version 2, Schema 2
---

总结`docker pull`流程中[`Image Manifest Version 2, Schema 2`](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md)原理（以[docker 1.11.0](https://github.com/docker/docker/tree/v1.11.0)为分析版本）

### docker pull

docker pull日志

```sh
x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/manifests/v0 HTTP/1.1" 200 923 "" "docker/1.1
1.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.el6.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(li
nux\\))"

x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/blobs/sha256:2b519bd204483370e81176d98fd0c9bc
4632e156da7b2cc752fa383b96e7c042 HTTP/1.1" 200 1756 "" "docker/1.11.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.el6.x8
6_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(linux\\))"

x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/blobs/sha256:a3ed95caeb02ffe68cdd9fd84406680a
e93d633cb16422d00e8a7c22955b46d4 HTTP/1.1" 200 32 "" "docker/1.11.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.el6.x86_
64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(linux\\))"

x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/blobs/sha256:93eea0ce9921b81687ad054452396461
f29baf653157c368cd347f9caa6e58f7 HTTP/1.1" 200 10289 "" "docker/1.11.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.el6.x
86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(linux\\))"

x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/blobs/sha256:c0a04912aa5afc0b4fd4c34390e526d5
47e67431f6bc122084f1e692dcb7d34e HTTP/1.1" 200 224153958 "" "docker/1.11.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.e
l6.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(linux\\))"
```

入口Pull

```go
func (p *v2Puller) Pull(ctx context.Context, ref reference.Named) (err error) {
    // TODO(tiborvass): was ReceiveTimeout
    p.repo, p.confirmedV2, err = NewV2Repository(ctx, p.repoInfo, p.endpoint, p.config.MetaHeaders, p.config.AuthConfig, "pull")
    if err != nil {
        logrus.Warnf("Error getting v2 registry: %v", err)
        return err
    }

    if err = p.pullV2Repository(ctx, ref); err != nil {
        if _, ok := err.(fallbackError); ok {
            return err
        }
        if continueOnError(err) {
            logrus.Errorf("Error trying v2 registry: %v", err)
            return fallbackError{
                err:         err,
                confirmedV2: p.confirmedV2,
                transportOK: true,
            }
        }
    }
    return err
}
```

调用pullV2Repository

```go
func (p *v2Puller) pullV2Repository(ctx context.Context, ref reference.Named) (err error) {
    var layersDownloaded bool
    if !reference.IsNameOnly(ref) {
        layersDownloaded, err = p.pullV2Tag(ctx, ref)
        if err != nil {
            return err
        }
    } else {
        tags, err := p.repo.Tags(ctx).All(ctx)
        if err != nil {
            // If this repository doesn't exist on V2, we should
            // permit a fallback to V1.
            return allowV1Fallback(err)
        }

        // The v2 registry knows about this repository, so we will not
        // allow fallback to the v1 protocol even if we encounter an
        // error later on.
        p.confirmedV2 = true

        for _, tag := range tags {
            tagRef, err := reference.WithTag(ref, tag)
            if err != nil {
                return err
            }
            pulledNew, err := p.pullV2Tag(ctx, tagRef)
            if err != nil {
                // Since this is the pull-all-tags case, don't
                // allow an error pulling a particular tag to
                // make the whole pull fall back to v1.
                if fallbackErr, ok := err.(fallbackError); ok {
                    return fallbackErr.err
                }
                return err
            }
            // pulledNew is true if either new layers were downloaded OR if existing images were newly tagged
            // TODO(tiborvass): should we change the name of `layersDownload`? What about message in WriteStatus?
            layersDownloaded = layersDownloaded || pulledNew
        }
    }

    writeStatus(ref.String(), p.config.ProgressOutput, layersDownloaded)

    return nil
}
```

调用pullV2Tag

```go
func (p *v2Puller) pullV2Tag(ctx context.Context, ref reference.Named) (tagUpdated bool, err error) {
    manSvc, err := p.repo.Manifests(ctx)
    if err != nil {
        return false, err
    }

    var (
        manifest    distribution.Manifest
        tagOrDigest string // Used for logging/progress only
    )
    if tagged, isTagged := ref.(reference.NamedTagged); isTagged {
        // NOTE: not using TagService.Get, since it uses HEAD requests
        // against the manifests endpoint, which are not supported by
        // all registry versions.
        manifest, err = manSvc.Get(ctx, "", client.WithTag(tagged.Tag()))
        if err != nil {
            return false, allowV1Fallback(err)
        }
        tagOrDigest = tagged.Tag()
    } else if digested, isDigested := ref.(reference.Canonical); isDigested {
        manifest, err = manSvc.Get(ctx, digested.Digest())
        if err != nil {
            return false, err
        }
        tagOrDigest = digested.Digest().String()
    } else {
        return false, fmt.Errorf("internal error: reference has neither a tag nor a digest: %s", ref.String())
    }

    if manifest == nil {
        return false, fmt.Errorf("image manifest does not exist for tag or digest %q", tagOrDigest)
    }

    // If manSvc.Get succeeded, we can be confident that the registry on
    // the other side speaks the v2 protocol.
    p.confirmedV2 = true

    logrus.Debugf("Pulling ref from V2 registry: %s", ref.String())
    progress.Message(p.config.ProgressOutput, tagOrDigest, "Pulling from "+p.repo.Named().Name())

    var (
        imageID        image.ID
        manifestDigest digest.Digest
    )

    switch v := manifest.(type) {
    case *schema1.SignedManifest:
        imageID, manifestDigest, err = p.pullSchema1(ctx, ref, v)
        if err != nil {
            return false, err
        }
    case *schema2.DeserializedManifest:
        imageID, manifestDigest, err = p.pullSchema2(ctx, ref, v)
        if err != nil {
            return false, err
        }
    case *manifestlist.DeserializedManifestList:
        imageID, manifestDigest, err = p.pullManifestList(ctx, ref, v)
        if err != nil {
            return false, err
        }
    default:
        return false, errors.New("unsupported manifest format")
    }

    progress.Message(p.config.ProgressOutput, "", "Digest: "+manifestDigest.String())

    oldTagImageID, err := p.config.ReferenceStore.Get(ref)
    if err == nil {
        if oldTagImageID == imageID {
            return false, nil
        }
    } else if err != reference.ErrDoesNotExist {
        return false, err
    }

    if canonical, ok := ref.(reference.Canonical); ok {
        if err = p.config.ReferenceStore.AddDigest(canonical, imageID, true); err != nil {
            return false, err
        }
    } else if err = p.config.ReferenceStore.AddTag(ref, imageID, true); err != nil {
        return false, err
    }

    return true, nil
}

type v2Puller struct {
    V2MetadataService *metadata.V2MetadataService
    endpoint          registry.APIEndpoint
    config            *ImagePullConfig
    repoInfo          *registry.RepositoryInfo
    repo              distribution.Repository
    // confirmedV2 is set to true if we confirm we're talking to a v2
    // registry. This is used to limit fallbacks to the v1 protocol.
    confirmedV2 bool
}

// Repository is a named collection of manifests and layers.
type Repository interface {
    // Name returns the name of the repository.
    Name() reference.Named

    // Manifests returns a reference to this repository's manifest service.
    // with the supplied options applied.
    Manifests(ctx context.Context, options ...ManifestServiceOption) (ManifestService, error)

    // Blobs returns a reference to this repository's blob service.
    Blobs(ctx context.Context) BlobStore

    // TODO(stevvooe): The above BlobStore return can probably be relaxed to
    // be a BlobService for use with clients. This will allow such
    // implementations to avoid implementing ServeBlob.

    // Tags returns a reference to this repositories tag service
    Tags(ctx context.Context) TagService
}
```

Get Manifest

```go
func (ms *manifests) Get(ctx context.Context, dgst digest.Digest, options ...distribution.ManifestServiceOption) (distribution.Manifest, error) {
    var (
        digestOrTag string
        ref         reference.Named
        err         error
    )

    for _, option := range options {
        if opt, ok := option.(withTagOption); ok {
            digestOrTag = opt.tag
            ref, err = reference.WithTag(ms.name, opt.tag)
            if err != nil {
                return nil, err
            }
        } else {
            err := option.Apply(ms)
            if err != nil {
                return nil, err
            }
        }
    }

    if digestOrTag == "" {
        digestOrTag = dgst.String()
        ref, err = reference.WithDigest(ms.name, dgst)
        if err != nil {
            return nil, err
        }
    }

    u, err := ms.ub.BuildManifestURL(ref)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequest("GET", u, nil)
    if err != nil {
        return nil, err
    }

    for _, t := range distribution.ManifestMediaTypes() {
        req.Header.Add("Accept", t)
    }

    if _, ok := ms.etags[digestOrTag]; ok {
        req.Header.Set("If-None-Match", ms.etags[digestOrTag])
    }

    resp, err := ms.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    if resp.StatusCode == http.StatusNotModified {
        return nil, distribution.ErrManifestNotModified
    } else if SuccessStatus(resp.StatusCode) {
        mt := resp.Header.Get("Content-Type")
        body, err := ioutil.ReadAll(resp.Body)

        if err != nil {
            return nil, err
        }
        m, _, err := distribution.UnmarshalManifest(mt, body)
        if err != nil {
            return nil, err
        }
        return m, nil
    }
    return nil, HandleErrorResponse(resp)
}
// BuildManifestURL constructs a url for the manifest identified by name and
// reference. The argument reference may be either a tag or digest.
func (ub *URLBuilder) BuildManifestURL(ref reference.Named) (string, error) {
    route := ub.cloneRoute(RouteNameManifest)

    tagOrDigest := ""
    switch v := ref.(type) {
    case reference.Tagged:
        tagOrDigest = v.Tag()
    case reference.Digested:
        tagOrDigest = v.Digest().String()
    }

    manifestURL, err := route.URL("name", ref.Name(), "reference", tagOrDigest)
    if err != nil {
        return "", err
    }

    return manifestURL.String(), nil
}
```

如下：

```sh
curl -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' http://xxxx/v2/duyanghao/busybox/manifests/v0
```

```sh
< Content-Type: application/vnd.docker.distribution.manifest.v2+json

{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/octet-stream",
      "size": 1756,
      "digest": "sha256:2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 224153958,
         "digest": "sha256:c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 32,
         "digest": "sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 10289,
         "digest": "sha256:93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7"
      }
   ]
}
```

转到UnmarshalManifest函数

```go
// UnmarshalManifest looks up manifest unmarshal functions based on
// MediaType
func UnmarshalManifest(ctHeader string, p []byte) (Manifest, Descriptor, error) {
    // Need to look up by the actual media type, not the raw contents of
    // the header. Strip semicolons and anything following them.
    var mediatype string
    if ctHeader != "" {
        var err error
        mediatype, _, err = mime.ParseMediaType(ctHeader)
        if err != nil {
            return nil, Descriptor{}, err
        }
    }

    unmarshalFunc, ok := mappings[mediatype]
    if !ok {
        unmarshalFunc, ok = mappings[""]
        if !ok {
            return nil, Descriptor{}, fmt.Errorf("unsupported manifest mediatype and no default available: %s", mediatype)
        }
    }

    return unmarshalFunc(p)
}

// UnmarshalFunc implements manifest unmarshalling a given MediaType
type UnmarshalFunc func([]byte) (Manifest, Descriptor, error)

func init() {
    schema2Func := func(b []byte) (distribution.Manifest, distribution.Descriptor, error) {
        m := new(DeserializedManifest)
        err := m.UnmarshalJSON(b)
        if err != nil {
            return nil, distribution.Descriptor{}, err
        }

        dgst := digest.FromBytes(b)
        return m, distribution.Descriptor{Digest: dgst, Size: int64(len(b)), MediaType: MediaTypeManifest}, err
    }
    err := distribution.RegisterManifestSchema(MediaTypeManifest, schema2Func)
    if err != nil {
        panic(fmt.Sprintf("Unable to register manifest: %s", err))
    }
}
const (
    // MediaTypeManifest specifies the mediaType for the current version.
    MediaTypeManifest = "application/vnd.docker.distribution.manifest.v2+json"

    // MediaTypeConfig specifies the mediaType for the image configuration.
    MediaTypeConfig = "application/vnd.docker.container.image.v1+json"

    // MediaTypeLayer is the mediaType used for layers referenced by the
    // manifest.
    MediaTypeLayer = "application/vnd.docker.image.rootfs.diff.tar.gzip"
)
```

pullSchema2函数

```go
func (p *v2Puller) pullSchema2(ctx context.Context, ref reference.Named, mfst *schema2.DeserializedManifest) (imageID image.ID, manifestDigest digest.Digest, err error) {
    manifestDigest, err = schema2ManifestDigest(ref, mfst)
    if err != nil {
        return "", "", err
    }

    target := mfst.Target()
    imageID = image.ID(target.Digest)
    if _, err := p.config.ImageStore.Get(imageID); err == nil {
        // If the image already exists locally, no need to pull
        // anything.
        return imageID, manifestDigest, nil
    }

    configChan := make(chan []byte, 1)
    errChan := make(chan error, 1)
    var cancel func()
    ctx, cancel = context.WithCancel(ctx)

    // Pull the image config
    go func() {
        configJSON, err := p.pullSchema2ImageConfig(ctx, target.Digest)
        if err != nil {
            errChan <- ImageConfigPullError{Err: err}
            cancel()
            return
        }
        configChan <- configJSON
    }()

    var descriptors []xfer.DownloadDescriptor

    // Note that the order of this loop is in the direction of bottom-most
    // to top-most, so that the downloads slice gets ordered correctly.
    for _, d := range mfst.References() {
        layerDescriptor := &v2LayerDescriptor{
            digest:            d.Digest,
            repo:              p.repo,
            repoInfo:          p.repoInfo,
            V2MetadataService: p.V2MetadataService,
        }

        descriptors = append(descriptors, layerDescriptor)
    }

    var (
        configJSON         []byte       // raw serialized image config
        unmarshalledConfig image.Image  // deserialized image config
        downloadRootFS     image.RootFS // rootFS to use for registering layers.
    )
    if runtime.GOOS == "windows" {
        configJSON, unmarshalledConfig, err = receiveConfig(configChan, errChan)
        if err != nil {
            return "", "", err
        }
        if unmarshalledConfig.RootFS == nil {
            return "", "", errors.New("image config has no rootfs section")
        }
        downloadRootFS = *unmarshalledConfig.RootFS
        downloadRootFS.DiffIDs = []layer.DiffID{}
    } else {
        downloadRootFS = *image.NewRootFS()
    }

    rootFS, release, err := p.config.DownloadManager.Download(ctx, downloadRootFS, descriptors, p.config.ProgressOutput)
    if err != nil {
        if configJSON != nil {
            // Already received the config
            return "", "", err
        }
        select {
        case err = <-errChan:
            return "", "", err
        default:
            cancel()
            select {
            case <-configChan:
            case <-errChan:
            }
            return "", "", err
        }
    }
    defer release()

    if configJSON == nil {
        configJSON, unmarshalledConfig, err = receiveConfig(configChan, errChan)
        if err != nil {
            return "", "", err
        }
    }

    // The DiffIDs returned in rootFS MUST match those in the config.
    // Otherwise the image config could be referencing layers that aren't
    // included in the manifest.
    if len(rootFS.DiffIDs) != len(unmarshalledConfig.RootFS.DiffIDs) {
        return "", "", errRootFSMismatch
    }

    for i := range rootFS.DiffIDs {
        if rootFS.DiffIDs[i] != unmarshalledConfig.RootFS.DiffIDs[i] {
            return "", "", errRootFSMismatch
        }
    }

    imageID, err = p.config.ImageStore.Create(configJSON)
    if err != nil {
        return "", "", err
    }

    return imageID, manifestDigest, nil
}
```

docker pull逻辑：

* step1：由镜像名请求Manifest Schema v2

* step2：解析Manifest获取镜像Configuration

* step3：下载各Layer gzip压缩文件

* step4：验证Configuration中的RootFS.DiffIDs是否与下载（解压后）hash相同

解析Manifest获取镜像Configuration

```sh
curl -L http://xxxx/v2/duyanghao/busybox/blobs/sha256:2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042
```

```go
func (p *v2Puller) pullSchema2ImageConfig(ctx context.Context, dgst digest.Digest) (configJSON []byte, err error) {
    blobs := p.repo.Blobs(ctx)
    configJSON, err = blobs.Get(ctx, dgst)
    if err != nil {
        return nil, err
    }

    // Verify image config digest
    verifier, err := digest.NewDigestVerifier(dgst)
    if err != nil {
        return nil, err
    }
    if _, err := verifier.Write(configJSON); err != nil {
        return nil, err
    }
    if !verifier.Verified() {
        err := fmt.Errorf("image config verification failed for digest %s", dgst)
        logrus.Error(err)
        return nil, err
    }

    return configJSON, nil
}
// NewDigestVerifier returns a verifier that compares the written bytes
// against a passed in digest.
func NewDigestVerifier(d Digest) (Verifier, error) {
    if err := d.Validate(); err != nil {
        return nil, err
    }

    return hashVerifier{
        hash:   d.Algorithm().Hash(),
        digest: d,
    }, nil
}
type hashVerifier struct {
    digest Digest
    hash   hash.Hash
}

func (hv hashVerifier) Write(p []byte) (n int, err error) {
    return hv.hash.Write(p)
}

func (hv hashVerifier) Verified() bool {
    return hv.digest == NewDigest(hv.digest.Algorithm(), hv.hash)
}
type Digest string

// NewDigest returns a Digest from alg and a hash.Hash object.
func NewDigest(alg Algorithm, h hash.Hash) Digest {
    return NewDigestFromBytes(alg, h.Sum(nil))
}

// NewDigestFromBytes returns a new digest from the byte contents of p.
// Typically, this can come from hash.Hash.Sum(...) or xxx.SumXXX(...)
// functions. This is also useful for rebuilding digests from binary
// serializations.
func NewDigestFromBytes(alg Algorithm, p []byte) Digest {
    return Digest(fmt.Sprintf("%s:%x", alg, p))
}
func receiveConfig(configChan <-chan []byte, errChan <-chan error) ([]byte, image.Image, error) {
    select {
    case configJSON := <-configChan:
        var unmarshalledConfig image.Image
        if err := json.Unmarshal(configJSON, &unmarshalledConfig); err != nil {
            return nil, image.Image{}, err
        }
        return configJSON, unmarshalledConfig, nil
    case err := <-errChan:
        return nil, image.Image{}, err
        // Don't need a case for ctx.Done in the select because cancellation
        // will trigger an error in p.pullSchema2ImageConfig.
    }
}
// Image stores the image configuration
type Image struct {
    V1Image
    Parent  ID        `json:"parent,omitempty"`
    RootFS  *RootFS   `json:"rootfs,omitempty"`
    History []History `json:"history,omitempty"`

    // rawJSON caches the immutable JSON associated with this image.
    rawJSON []byte

    // computedID is the ID computed from the hash of the image config.
    // Not to be confused with the legacy V1 ID in V1Image.
    computedID ID
}
// V1Image stores the V1 image configuration.
type V1Image struct {
    // ID a unique 64 character identifier of the image
    ID string `json:"id,omitempty"`
    // Parent id of the image
    Parent string `json:"parent,omitempty"`
    // Comment user added comment
    Comment string `json:"comment,omitempty"`
    // Created timestamp when image was created
    Created time.Time `json:"created"`
    // Container is the id of the container used to commit
    Container string `json:"container,omitempty"`
    // ContainerConfig is the configuration of the container that is committed into the image
    ContainerConfig container.Config `json:"container_config,omitempty"`
    // DockerVersion specifies version on which image is built
    DockerVersion string `json:"docker_version,omitempty"`
    // Author of the image
    Author string `json:"author,omitempty"`
    // Config is the configuration of the container received from the client
    Config *container.Config `json:"config,omitempty"`
    // Architecture is the hardware that the image is build and runs on
    Architecture string `json:"architecture,omitempty"`
    // OS is the operating system used to build and run the image
    OS string `json:"os,omitempty"`
    // Size is the total size of the image including all layers it is composed of
    Size int64 `json:",omitempty"`
}
// History stores build commands that were used to create an image
type History struct {
    // Created timestamp for build point
    Created time.Time `json:"created"`
    // Author of the build point
    Author string `json:"author,omitempty"`
    // CreatedBy keeps the Dockerfile command used while building image.
    CreatedBy string `json:"created_by,omitempty"`
    // Comment is custom message set by the user when creating the image.
    Comment string `json:"comment,omitempty"`
    // EmptyLayer is set to true if this history item did not generate a
    // layer. Otherwise, the history item is associated with the next
    // layer in the RootFS section.
    EmptyLayer bool `json:"empty_layer,omitempty"`
}
func (bs *blobs) Get(ctx context.Context, dgst digest.Digest) ([]byte, error) {
    reader, err := bs.Open(ctx, dgst)
    if err != nil {
        return nil, err
    }
    defer reader.Close()

    return ioutil.ReadAll(reader)
}
func (bs *blobs) Open(ctx context.Context, dgst digest.Digest) (distribution.ReadSeekCloser, error) {
    ref, err := reference.WithDigest(bs.name, dgst)
    if err != nil {
        return nil, err
    }
    blobURL, err := bs.ub.BuildBlobURL(ref)
    if err != nil {
        return nil, err
    }

    return transport.NewHTTPReadSeeker(bs.client, blobURL,
        func(resp *http.Response) error {
            if resp.StatusCode == http.StatusNotFound {
                return distribution.ErrBlobUnknown
            }
            return HandleErrorResponse(resp)
        }), nil
}
```

镜像configuration文件存储位置：


>/data/docker/image/aufs/imagedb/content/sha256/2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042

```go
imageID, err = p.config.ImageStore.Create(configJSON)
```

configuration写入操作

```go
// Store is an interface for creating and accessing images
type Store interface {
    Create(config []byte) (ID, error)
    Get(id ID) (*Image, error)
    Delete(id ID) ([]layer.Metadata, error)
    Search(partialID string) (ID, error)
    SetParent(id ID, parent ID) error
    GetParent(id ID) (ID, error)
    Children(id ID) []ID
    Map() map[ID]*Image
    Heads() map[ID]*Image
}
type store struct {
    sync.Mutex
    ls        LayerGetReleaser
    images    map[ID]*imageMeta
    fs        StoreBackend
    digestSet *digest.Set
}
func (is *store) Create(config []byte) (ID, error) {
    var img Image
    err := json.Unmarshal(config, &img)
    if err != nil {
        return "", err
    }

    // Must reject any config that references diffIDs from the history
    // which aren't among the rootfs layers.
    rootFSLayers := make(map[layer.DiffID]struct{})
    for _, diffID := range img.RootFS.DiffIDs {
        rootFSLayers[diffID] = struct{}{}
    }

    layerCounter := 0
    for _, h := range img.History {
        if !h.EmptyLayer {
            layerCounter++
        }
    }
    if layerCounter > len(img.RootFS.DiffIDs) {
        return "", errors.New("too many non-empty layers in History section")
    }

    dgst, err := is.fs.Set(config)
    if err != nil {
        return "", err
    }
    imageID := ID(dgst)

    is.Lock()
    defer is.Unlock()

    if _, exists := is.images[imageID]; exists {
        return imageID, nil
    }

    layerID := img.RootFS.ChainID()

    var l layer.Layer
    if layerID != "" {
        l, err = is.ls.Get(layerID)
        if err != nil {
            return "", err
        }
    }

    imageMeta := &imageMeta{
        layer:    l,
        children: make(map[ID]struct{}),
    }

    is.images[imageID] = imageMeta
    if err := is.digestSet.Add(digest.Digest(imageID)); err != nil {
        delete(is.images, imageID)
        return "", err
    }

    return imageID, nil
}
// Set stores content under a given ID.
func (s *fs) Set(data []byte) (ID, error) {
    s.Lock()
    defer s.Unlock()

    if len(data) == 0 {
        return "", fmt.Errorf("Invalid empty data")
    }

    id := ID(digest.FromBytes(data))
    filePath := s.contentFile(id)
    tempFilePath := s.contentFile(id) + ".tmp"
    if err := ioutil.WriteFile(tempFilePath, data, 0600); err != nil {
        return "", err
    }
    if err := os.Rename(tempFilePath, filePath); err != nil {
        return "", err
    }

    return id, nil
}
// FromBytes digests the input and returns a Digest.
func FromBytes(p []byte) Digest {
    return Canonical.FromBytes(p)
}
// supported digest types
const (
    SHA256 Algorithm = "sha256" // sha256 with hex encoding
    SHA384 Algorithm = "sha384" // sha384 with hex encoding
    SHA512 Algorithm = "sha512" // sha512 with hex encoding

    // Canonical is the primary digest algorithm used with the distribution
    // project. Other digests may be used but this one is the primary storage
    // digest.
    Canonical = SHA256
)
// FromBytes digests the input and returns a Digest.
func (a Algorithm) FromBytes(p []byte) Digest {
    digester := a.New()

    if _, err := digester.Hash().Write(p); err != nil {
        // Writes to a Hash should never fail. None of the existing
        // hash implementations in the stdlib or hashes vendored
        // here can return errors from Write. Having a panic in this
        // condition instead of having FromBytes return an error value
        // avoids unnecessary error handling paths in all callers.
        panic("write to hash function returned error: " + err.Error())
    }

    return digester.Digest()
}
// Digester calculates the digest of written data. Writes should go directly
// to the return value of Hash, while calling Digest will return the current
// value of the digest.
type Digester interface {
    Hash() hash.Hash // provides direct access to underlying hash instance.
    Digest() Digest
}

// digester provides a simple digester definition that embeds a hasher.
type digester struct {
    alg  Algorithm
    hash hash.Hash
}
func (s *fs) contentFile(id ID) string {
    dgst := digest.Digest(id)
    return filepath.Join(s.root, contentDirName, string(dgst.Algorithm()), dgst.Hex())
}
```

获取镜像`configuration`后，进行验证：

* Manifest中的`sha256 hash`与对configuration内容`sha256 hash`进行对比(要相同)

* configuration文件中`History`数目与`RootFS.DiffIDs`数据进行对比（要相同）

验证成功后将configuration内容写入文件

manifestDigest生成

```go
manifestDigest, err = schema2ManifestDigest(ref, mfst)

// schema2ManifestDigest computes the manifest digest, and, if pulling by
// digest, ensures that it matches the requested digest.
func schema2ManifestDigest(ref reference.Named, mfst distribution.Manifest) (digest.Digest, error) {
    _, canonical, err := mfst.Payload()
    if err != nil {
        return "", err
    }

    // If pull by digest, then verify the manifest digest.
    if digested, isDigested := ref.(reference.Canonical); isDigested {
        verifier, err := digest.NewDigestVerifier(digested.Digest())
        if err != nil {
            return "", err
        }
        if _, err := verifier.Write(canonical); err != nil {
            return "", err
        }
        if !verifier.Verified() {
            err := fmt.Errorf("manifest verification failed for digest %s", digested.Digest())
            logrus.Error(err)
            return "", err
        }
        return digested.Digest(), nil
    }

    return digest.FromBytes(canonical), nil
}
// Payload returns the raw content of the manifest. The contents can be used to
// calculate the content identifier.
func (m DeserializedManifest) Payload() (string, []byte, error) {
    return m.MediaType, m.canonical, nil
}
```

对Manifest v2内容进行sha256 hash得到manifestDigest(**d5ab5a18ba5a252216a930976e7a1d22ec6c4bb40d600df5dcea8714ca7973bc**)

```sh
[root@CentOS-64-duyanghao ~]# docker pull xxxx/duyanghao/busybox:v0
v0: Pulling from duyanghao/busybox
c0a04912aa5a: Pull complete 
a3ed95caeb02: Pull complete 
93eea0ce9921: Pull complete 
Digest: sha256:d5ab5a18ba5a252216a930976e7a1d22ec6c4bb40d600df5dcea8714ca7973bc
Status: Downloaded newer image for xxxx/duyanghao/busybox:v0
```

下载各layer文件

```go
    var descriptors []xfer.DownloadDescriptor

    // Note that the order of this loop is in the direction of bottom-most
    // to top-most, so that the downloads slice gets ordered correctly.
    for _, d := range mfst.References() {
        layerDescriptor := &v2LayerDescriptor{
            digest:            d.Digest,
            repo:              p.repo,
            repoInfo:          p.repoInfo,
            V2MetadataService: p.V2MetadataService,
        }

        descriptors = append(descriptors, layerDescriptor)
    }
...
rootFS, release, err := p.config.DownloadManager.Download(ctx, downloadRootFS, descriptors, p.config.ProgressOutput)


// References returnes the descriptors of this manifests references.
func (m Manifest) References() []distribution.Descriptor {
    return m.Layers

}
// ImagePullConfig stores pull configuration.
type ImagePullConfig struct {
    // MetaHeaders stores HTTP headers with metadata about the image
    MetaHeaders map[string][]string
    // AuthConfig holds authentication credentials for authenticating with
    // the registry.
    AuthConfig *types.AuthConfig
    // ProgressOutput is the interface for showing the status of the pull
    // operation.
    ProgressOutput progress.Output
    // RegistryService is the registry service to use for TLS configuration
    // and endpoint lookup.
    RegistryService *registry.Service
    // ImageEventLogger notifies events for a given image
    ImageEventLogger func(id, name, action string)
    // MetadataStore is the storage backend for distribution-specific
    // metadata.
    MetadataStore metadata.Store
    // ImageStore manages images.
    ImageStore image.Store
    // ReferenceStore manages tags.
    ReferenceStore reference.Store
    // DownloadManager manages concurrent pulls.
    DownloadManager *xfer.LayerDownloadManager
}
// LayerDownloadManager figures out which layers need to be downloaded, then
// registers and downloads those, taking into account dependencies between
// layers.
type LayerDownloadManager struct {
    layerStore layer.Store
    tm         TransferManager
}
// A DownloadDescriptor references a layer that may need to be downloaded.
type DownloadDescriptor interface {
    // Key returns the key used to deduplicate downloads.
    Key() string
    // ID returns the ID for display purposes.
    ID() string
    // DiffID should return the DiffID for this layer, or an error
    // if it is unknown (for example, if it has not been downloaded
    // before).
    DiffID() (layer.DiffID, error)
    // Download is called to perform the download.
    Download(ctx context.Context, progressOutput progress.Output) (io.ReadCloser, int64, error)
    // Close is called when the download manager is finished with this
    // descriptor and will not call Download again or read from the reader
    // that Download returned.
    Close()
}
// Download is a blocking function which ensures the requested layers are
// present in the layer store. It uses the string returned by the Key method to
// deduplicate downloads. If a given layer is not already known to present in
// the layer store, and the key is not used by an in-progress download, the
// Download method is called to get the layer tar data. Layers are then
// registered in the appropriate order.  The caller must call the returned
// release function once it is is done with the returned RootFS object.
func (ldm *LayerDownloadManager) Download(ctx context.Context, initialRootFS image.RootFS, layers []DownloadDescriptor, progressOutput progress.Output) (image.RootFS, func(), error) {
    var (
        topLayer       layer.Layer
        topDownload    *downloadTransfer
        watcher        *Watcher
        missingLayer   bool
        transferKey    = ""
        downloadsByKey = make(map[string]*downloadTransfer)
    )

    rootFS := initialRootFS
    for _, descriptor := range layers {
        key := descriptor.Key()
        transferKey += key

        if !missingLayer {
            missingLayer = true
            diffID, err := descriptor.DiffID()
            if err == nil {
                getRootFS := rootFS
                getRootFS.Append(diffID)
                l, err := ldm.layerStore.Get(getRootFS.ChainID())
                if err == nil {
                    // Layer already exists.
                    logrus.Debugf("Layer already exists: %s", descriptor.ID())
                    progress.Update(progressOutput, descriptor.ID(), "Already exists")
                    if topLayer != nil {
                        layer.ReleaseAndLog(ldm.layerStore, topLayer)
                    }
                    topLayer = l
                    missingLayer = false
                    rootFS.Append(diffID)
                    continue
                }
            }
        }

        // Does this layer have the same data as a previous layer in
        // the stack? If so, avoid downloading it more than once.
        var topDownloadUncasted Transfer
        if existingDownload, ok := downloadsByKey[key]; ok {
            xferFunc := ldm.makeDownloadFuncFromDownload(descriptor, existingDownload, topDownload)
            defer topDownload.Transfer.Release(watcher)
            topDownloadUncasted, watcher = ldm.tm.Transfer(transferKey, xferFunc, progressOutput)
            topDownload = topDownloadUncasted.(*downloadTransfer)
            continue
        }

        // Layer is not known to exist - download and register it.
        progress.Update(progressOutput, descriptor.ID(), "Pulling fs layer")

        var xferFunc DoFunc
        if topDownload != nil {
            xferFunc = ldm.makeDownloadFunc(descriptor, "", topDownload)
            defer topDownload.Transfer.Release(watcher)
        } else {
            xferFunc = ldm.makeDownloadFunc(descriptor, rootFS.ChainID(), nil)
        }
        topDownloadUncasted, watcher = ldm.tm.Transfer(transferKey, xferFunc, progressOutput)
        topDownload = topDownloadUncasted.(*downloadTransfer)
        downloadsByKey[key] = topDownload
    }

    if topDownload == nil {
        return rootFS, func() {
            if topLayer != nil {
                layer.ReleaseAndLog(ldm.layerStore, topLayer)
            }
        }, nil
    }

    // Won't be using the list built up so far - will generate it
    // from downloaded layers instead.
    rootFS.DiffIDs = []layer.DiffID{}

    defer func() {
        if topLayer != nil {
            layer.ReleaseAndLog(ldm.layerStore, topLayer)
        }
    }()

    select {
    case <-ctx.Done():
        topDownload.Transfer.Release(watcher)
        return rootFS, func() {}, ctx.Err()
    case <-topDownload.Done():
        break
    }

    l, err := topDownload.result()
    if err != nil {
        topDownload.Transfer.Release(watcher)
        return rootFS, func() {}, err
    }

    // Must do this exactly len(layers) times, so we don't include the
    // base layer on Windows.
    for range layers {
        if l == nil {
            topDownload.Transfer.Release(watcher)
            return rootFS, func() {}, errors.New("internal error: too few parent layers")
        }
        rootFS.DiffIDs = append([]layer.DiffID{l.DiffID()}, rootFS.DiffIDs...)
        l = l.Parent()
    }
    return rootFS, func() { topDownload.Transfer.Release(watcher) }, err
}
```

makeDownloadFunc函数

```go
// makeDownloadFunc returns a function that performs the layer download and
// registration. If parentDownload is non-nil, it waits for that download to
// complete before the registration step, and registers the downloaded data
// on top of parentDownload's resulting layer. Otherwise, it registers the
// layer on top of the ChainID given by parentLayer.
func (ldm *LayerDownloadManager) makeDownloadFunc(descriptor DownloadDescriptor, parentLayer layer.ChainID, parentDownload *downloadTransfer) DoFunc {
    return func(progressChan chan<- progress.Progress, start <-chan struct{}, inactive chan<- struct{}) Transfer {
        d := &downloadTransfer{
            Transfer:   NewTransfer(),
            layerStore: ldm.layerStore,
        }

        go func() {
            defer func() {
                close(progressChan)
            }()

            progressOutput := progress.ChanOutput(progressChan)

            select {
            case <-start:
            default:
                progress.Update(progressOutput, descriptor.ID(), "Waiting")
                <-start
            }

            if parentDownload != nil {
                // Did the parent download already fail or get
                // cancelled?
                select {
                case <-parentDownload.Done():
                    _, err := parentDownload.result()
                    if err != nil {
                        d.err = err
                        return
                    }
                default:
                }
            }

            var (
                downloadReader io.ReadCloser
                size           int64
                err            error
                retries        int
            )

            defer descriptor.Close()

            for {
                downloadReader, size, err = descriptor.Download(d.Transfer.Context(), progressOutput)
                if err == nil {
                    break
                }

                // If an error was returned because the context
                // was cancelled, we shouldn't retry.
                select {
                case <-d.Transfer.Context().Done():
                    d.err = err
                    return
                default:
                }

                retries++
                if _, isDNR := err.(DoNotRetry); isDNR || retries == maxDownloadAttempts {
                    logrus.Errorf("Download failed: %v", err)
                    d.err = err
                    return
                }

                logrus.Errorf("Download failed, retrying: %v", err)
                delay := retries * 5
                ticker := time.NewTicker(time.Second)

            selectLoop:
                for {
                    progress.Updatef(progressOutput, descriptor.ID(), "Retrying in %d second%s", delay, (map[bool]string{true: "s"})[delay != 1])
                    select {
                    case <-ticker.C:
                        delay--
                        if delay == 0 {
                            ticker.Stop()
                            break selectLoop
                        }
                    case <-d.Transfer.Context().Done():
                        ticker.Stop()
                        d.err = errors.New("download cancelled during retry delay")
                        return
                    }

                }
            }

            close(inactive)

            if parentDownload != nil {
                select {
                case <-d.Transfer.Context().Done():
                    d.err = errors.New("layer registration cancelled")
                    downloadReader.Close()
                    return
                case <-parentDownload.Done():
                }

                l, err := parentDownload.result()
                if err != nil {
                    d.err = err
                    downloadReader.Close()
                    return
                }
                parentLayer = l.ChainID()
            }

            reader := progress.NewProgressReader(ioutils.NewCancelReadCloser(d.Transfer.Context(), downloadReader), progressOutput, size, descriptor.ID(), "Extracting")
            defer reader.Close()

            inflatedLayerData, err := archive.DecompressStream(reader)
            if err != nil {
                d.err = fmt.Errorf("could not get decompression stream: %v", err)
                return
            }

            d.layer, err = d.layerStore.Register(inflatedLayerData, parentLayer)
            if err != nil {
                select {
                case <-d.Transfer.Context().Done():
                    d.err = errors.New("layer registration cancelled")
                default:
                    d.err = fmt.Errorf("failed to register layer: %v", err)
                }
                return
            }

            progress.Update(progressOutput, descriptor.ID(), "Pull complete")
            withRegistered, hasRegistered := descriptor.(DownloadDescriptorWithRegistered)
            if hasRegistered {
                withRegistered.Registered(d.layer.DiffID())
            }

            // Doesn't actually need to be its own goroutine, but
            // done like this so we can defer close(c).
            go func() {
                <-d.Transfer.Released()
                if d.layer != nil {
                    layer.ReleaseAndLog(d.layerStore, d.layer)
                }
            }()
        }()

        return d
    }
}
```

Download

```go
downloadReader, size, err = descriptor.Download(d.Transfer.Context(), progressOutput)

func (ld *v2LayerDescriptor) Download(ctx context.Context, progressOutput progress.Output) (io.ReadCloser, int64, error) {
    logrus.Debugf("pulling blob %q", ld.digest)

    var (
        err    error
        offset int64
    )

    if ld.tmpFile == nil {
        ld.tmpFile, err = createDownloadFile()
        if err != nil {
            return nil, 0, xfer.DoNotRetry{Err: err}
        }
    } else {
        offset, err = ld.tmpFile.Seek(0, os.SEEK_END)
        if err != nil {
            logrus.Debugf("error seeking to end of download file: %v", err)
            offset = 0

            ld.tmpFile.Close()
            if err := os.Remove(ld.tmpFile.Name()); err != nil {
                logrus.Errorf("Failed to remove temp file: %s", ld.tmpFile.Name())
            }
            ld.tmpFile, err = createDownloadFile()
            if err != nil {
                return nil, 0, xfer.DoNotRetry{Err: err}
            }
        } else if offset != 0 {
            logrus.Debugf("attempting to resume download of %q from %d bytes", ld.digest, offset)
        }
    }

    tmpFile := ld.tmpFile
    blobs := ld.repo.Blobs(ctx)

    layerDownload, err := blobs.Open(ctx, ld.digest)
    if err != nil {
        logrus.Errorf("Error initiating layer download: %v", err)
        if err == distribution.ErrBlobUnknown {
            return nil, 0, xfer.DoNotRetry{Err: err}
        }
        return nil, 0, retryOnError(err)
    }

    if offset != 0 {
        _, err := layerDownload.Seek(offset, os.SEEK_SET)
        if err != nil {
            if err := ld.truncateDownloadFile(); err != nil {
                return nil, 0, xfer.DoNotRetry{Err: err}
            }
            return nil, 0, err
        }
    }
    size, err := layerDownload.Seek(0, os.SEEK_END)
    if err != nil {
        // Seek failed, perhaps because there was no Content-Length
        // header. This shouldn't fail the download, because we can
        // still continue without a progress bar.
        size = 0
    } else {
        if size != 0 && offset > size {
            logrus.Debugf("Partial download is larger than full blob. Starting over")
            offset = 0
            if err := ld.truncateDownloadFile(); err != nil {
                return nil, 0, xfer.DoNotRetry{Err: err}
            }
        }

        // Restore the seek offset either at the beginning of the
        // stream, or just after the last byte we have from previous
        // attempts.
        _, err = layerDownload.Seek(offset, os.SEEK_SET)
        if err != nil {
            return nil, 0, err
        }
    }

    reader := progress.NewProgressReader(ioutils.NewCancelReadCloser(ctx, layerDownload), progressOutput, size-offset, ld.ID(), "Downloading")
    defer reader.Close()

    if ld.verifier == nil {
        ld.verifier, err = digest.NewDigestVerifier(ld.digest)
        if err != nil {
            return nil, 0, xfer.DoNotRetry{Err: err}
        }
    }

    _, err = io.Copy(tmpFile, io.TeeReader(reader, ld.verifier))
    if err != nil {
        if err == transport.ErrWrongCodeForByteRange {
            if err := ld.truncateDownloadFile(); err != nil {
                return nil, 0, xfer.DoNotRetry{Err: err}
            }
            return nil, 0, err
        }
        return nil, 0, retryOnError(err)
    }

    progress.Update(progressOutput, ld.ID(), "Verifying Checksum")

    if !ld.verifier.Verified() {
        err = fmt.Errorf("filesystem layer verification failed for digest %s", ld.digest)
        logrus.Error(err)

        // Allow a retry if this digest verification error happened
        // after a resumed download.
        if offset != 0 {
            if err := ld.truncateDownloadFile(); err != nil {
                return nil, 0, xfer.DoNotRetry{Err: err}
            }

            return nil, 0, err
        }
        return nil, 0, xfer.DoNotRetry{Err: err}
    }

    progress.Update(progressOutput, ld.ID(), "Download complete")

    logrus.Debugf("Downloaded %s to tempfile %s", ld.ID(), tmpFile.Name())

    _, err = tmpFile.Seek(0, os.SEEK_SET)
    if err != nil {
        tmpFile.Close()
        if err := os.Remove(tmpFile.Name()); err != nil {
            logrus.Errorf("Failed to remove temp file: %s", tmpFile.Name())
        }
        ld.tmpFile = nil
        ld.verifier = nil
        return nil, 0, xfer.DoNotRetry{Err: err}
    }

    // hand off the temporary file to the download manager, so it will only
    // be closed once
    ld.tmpFile = nil

    return ioutils.NewReadCloserWrapper(tmpFile, func() error {
        tmpFile.Close()
        err := os.RemoveAll(tmpFile.Name())
        if err != nil {
            logrus.Errorf("Failed to remove temp file: %s", tmpFile.Name())
        }
        return err
    }), size, nil
}
func createDownloadFile() (*os.File, error) {
    return ioutil.TempFile("", "GetImageBlob")
}
// NewProgressReader creates a new ProgressReader.
func NewProgressReader(in io.ReadCloser, out Output, size int64, id, action string) *Reader {
    return &Reader{
        in:     in,
        out:    out,
        size:   size,
        id:     id,
        action: action,
    }
}
// Reader is a Reader with progress bar.
type Reader struct {
    in         io.ReadCloser // Stream to read from
    out        Output        // Where to send progress bar to
    size       int64
    current    int64
    lastUpdate int64
    id         string
    action     string
}
```


```go
ioutils.NewCancelReadCloser(ctx, layerDownload)

// NewCancelReadCloser creates a wrapper that closes the ReadCloser when the
// context is cancelled. The returned io.ReadCloser must be closed when it is
// no longer needed.
func NewCancelReadCloser(ctx context.Context, in io.ReadCloser) io.ReadCloser {
    pR, pW := io.Pipe()

    // Create a context used to signal when the pipe is closed
    doneCtx, cancel := context.WithCancel(context.Background())

    p := &cancelReadCloser{
        cancel: cancel,
        pR:     pR,
        pW:     pW,
    }

    go func() {
        _, err := io.Copy(pW, in)
        select {
        case <-ctx.Done():
            // If the context was closed, p.closeWithError
            // was already called. Calling it again would
            // change the error that Read returns.
        default:
            p.closeWithError(err)
        }
        in.Close()
    }()
    go func() {
        for {
            select {
            case <-ctx.Done():
                p.closeWithError(ctx.Err())
            case <-doneCtx.Done():
                return
            }
        }
    }()

    return p
}
```

```go
layerDownload, err := blobs.Open(ctx, ld.digest)

type v2LayerDescriptor struct {
    digest            digest.Digest
    repoInfo          *registry.RepositoryInfo
    repo              distribution.Repository
    V2MetadataService *metadata.V2MetadataService
    tmpFile           *os.File
    verifier          digest.Verifier
}
func (r *repository) Blobs(ctx context.Context) distribution.BlobStore {
    statter := &blobStatter{
        name:   r.name,
        ub:     r.ub,
        client: r.client,
    }
    return &blobs{
        name:    r.name,
        ub:      r.ub,
        client:  r.client,
        statter: cache.NewCachedBlobStatter(memory.NewInMemoryBlobDescriptorCacheProvider(), statter),
    }
}
type blobs struct {
    name   reference.Named
    ub     *v2.URLBuilder
    client *http.Client

    statter distribution.BlobDescriptorService
    distribution.BlobDeleter
}
func (bs *blobs) Open(ctx context.Context, dgst digest.Digest) (distribution.ReadSeekCloser, error) {
    ref, err := reference.WithDigest(bs.name, dgst)
    if err != nil {
        return nil, err
    }
    blobURL, err := bs.ub.BuildBlobURL(ref)
    if err != nil {
        return nil, err
    }

    return transport.NewHTTPReadSeeker(bs.client, blobURL,
        func(resp *http.Response) error {
            if resp.StatusCode == http.StatusNotFound {
                return distribution.ErrBlobUnknown
            }
            return HandleErrorResponse(resp)
        }), nil
}
// BuildBlobURL constructs the url for the blob identified by name and dgst.
func (ub *URLBuilder) BuildBlobURL(ref reference.Canonical) (string, error) {
    route := ub.cloneRoute(RouteNameBlob)

    layerURL, err := route.URL("name", ref.Name(), "digest", ref.Digest().String())
    if err != nil {
        return "", err
    }

    return layerURL.String(), nil
}
// clondedRoute returns a clone of the named route from the router. Routes
// must be cloned to avoid modifying them during url generation.
func (ub *URLBuilder) cloneRoute(name string) clonedRoute {
    route := new(mux.Route)
    root := new(url.URL)

    *route = *ub.router.GetRoute(name) // clone the route
    *root = *ub.root

    return clonedRoute{Route: route, root: root}
}
// The following are definitions of the name under which all V2 routes are
// registered. These symbols can be used to look up a route based on the name.
const (
    RouteNameBase            = "base"
    RouteNameManifest        = "manifest"
    RouteNameTags            = "tags"
    RouteNameBlob            = "blob"
    RouteNameBlobUpload      = "blob-upload"
    RouteNameBlobUploadChunk = "blob-upload-chunk"
    RouteNameCatalog         = "catalog"
)
```

Layer拉取日志

```sh
x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/blobs/sha256:a3ed95caeb02ffe68cdd9fd84406680a
e93d633cb16422d00e8a7c22955b46d4 HTTP/1.1" 200 32 "" "docker/1.11.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.el6.x86_
64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(linux\\))"

x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/blobs/sha256:93eea0ce9921b81687ad054452396461
f29baf653157c368cd347f9caa6e58f7 HTTP/1.1" 200 10289 "" "docker/1.11.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.el6.x
86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(linux\\))"

x.x.x.x - - [30/Sep/2016:14:01:45 +0800] "GET /v2/duyanghao/busybox/blobs/sha256:c0a04912aa5afc0b4fd4c34390e526d5
47e67431f6bc122084f1e692dcb7d34e HTTP/1.1" 200 224153958 "" "docker/1.11.0 go/go1.5.4 git-commit/4dc5990 kernel/3.10.5-3.e
l6.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.11.0 \\(linux\\))"
```

```go
_, err = io.Copy(tmpFile, io.TeeReader(reader, ld.verifier))

type v2LayerDescriptor struct {
    digest            digest.Digest
    repoInfo          *registry.RepositoryInfo
    repo              distribution.Repository
    V2MetadataService *metadata.V2MetadataService
    tmpFile           *os.File
    verifier          digest.Verifier
}
```

TeeReader函数

>TeeReader returns a Reader that writes to w what it reads from r. All reads from r performed through it are matched with corresponding writes to w. There is no internal buffering - the write must complete before the read completes. Any error encountered while writing is reported as a read error.

将Layer文件下载完后存储到临时文件中，之后进行验证sha256 hash与对下载内容sha 256 hash结果对比（应该相同）

```go
    progress.Update(progressOutput, ld.ID(), "Verifying Checksum")

    if !ld.verifier.Verified() {
        err = fmt.Errorf("filesystem layer verification failed for digest %s", ld.digest)
        logrus.Error(err)

        // Allow a retry if this digest verification error happened
        // after a resumed download.
        if offset != 0 {
            if err := ld.truncateDownloadFile(); err != nil {
                return nil, 0, xfer.DoNotRetry{Err: err}
            }

            return nil, 0, err
        }
        return nil, 0, xfer.DoNotRetry{Err: err}
    }

    progress.Update(progressOutput, ld.ID(), "Download complete")
``` 

Seek函数

>Seek sets the offset for the next Read or Write on file to offset, interpreted according to whence: 0 means relative to the origin of the file, 1 means relative to the current offset, and 2 means relative to the end. It returns the new offset and an error, if any. The behavior of Seek on a file opened with O_APPEND is not specified.

```go
    size, err := layerDownload.Seek(0, os.SEEK_END)
    ...

    _, err = tmpFile.Seek(0, os.SEEK_SET)
    if err != nil {
        tmpFile.Close()
        if err := os.Remove(tmpFile.Name()); err != nil {
            logrus.Errorf("Failed to remove temp file: %s", tmpFile.Name())
        }
        ld.tmpFile = nil
        ld.verifier = nil
        return nil, 0, xfer.DoNotRetry{Err: err}
    }

    // hand off the temporary file to the download manager, so it will only
    // be closed once
    ld.tmpFile = nil

    return ioutils.NewReadCloserWrapper(tmpFile, func() error {
        tmpFile.Close()
        err := os.RemoveAll(tmpFile.Name())
        if err != nil {
            logrus.Errorf("Failed to remove temp file: %s", tmpFile.Name())
        }
        return err
    }), size, nil
```

下载完后进行解压(Extracting)

```go
            reader := progress.NewProgressReader(ioutils.NewCancelReadCloser(d.Transfer.Context(), downloadReader), progressOutput, size, descriptor.ID(), "Extracting")
            defer reader.Close()

            inflatedLayerData, err := archive.DecompressStream(reader)
            if err != nil {
                d.err = fmt.Errorf("could not get decompression stream: %v", err)
                return
            }


// DecompressStream decompress the archive and returns a ReaderCloser with the decompressed archive.
func DecompressStream(archive io.Reader) (io.ReadCloser, error) {
    p := pools.BufioReader32KPool
    buf := p.Get(archive)
    bs, err := buf.Peek(10)
    if err != nil && err != io.EOF {
        // Note: we'll ignore any io.EOF error because there are some odd
        // cases where the layer.tar file will be empty (zero bytes) and
        // that results in an io.EOF from the Peek() call. So, in those
        // cases we'll just treat it as a non-compressed stream and
        // that means just create an empty layer.
        // See Issue 18170
        return nil, err
    }

    compression := DetectCompression(bs)
    switch compression {
    case Uncompressed:
        readBufWrapper := p.NewReadCloserWrapper(buf, buf)
        return readBufWrapper, nil
    case Gzip:
        gzReader, err := gzip.NewReader(buf)
        if err != nil {
            return nil, err
        }
        readBufWrapper := p.NewReadCloserWrapper(buf, gzReader)
        return readBufWrapper, nil
    case Bzip2:
        bz2Reader := bzip2.NewReader(buf)
        readBufWrapper := p.NewReadCloserWrapper(buf, bz2Reader)
        return readBufWrapper, nil
    case Xz:
        xzReader, chdone, err := xzDecompress(buf)
        if err != nil {
            return nil, err
        }
        readBufWrapper := p.NewReadCloserWrapper(buf, xzReader)
        return ioutils.NewReadCloserWrapper(readBufWrapper, func() error {
            <-chdone
            return readBufWrapper.Close()
        }), nil
    default:
        return nil, fmt.Errorf("Unsupported compression format %s", (&compression).Extension())
    }
}
```

Register

![](/public/img/docker-registry/2016-10-26-docker-registry-pull-manifest-v2/relation.png)

```go
            d.layer, err = d.layerStore.Register(inflatedLayerData, parentLayer)
            if err != nil {
                select {
                case <-d.Transfer.Context().Done():
                    d.err = errors.New("layer registration cancelled")
                default:
                    d.err = fmt.Errorf("failed to register layer: %v", err)
                }
                return
            }

            progress.Update(progressOutput, descriptor.ID(), "Pull complete")
            withRegistered, hasRegistered := descriptor.(DownloadDescriptorWithRegistered)
            if hasRegistered {
                withRegistered.Registered(d.layer.DiffID())
            }

            // Doesn't actually need to be its own goroutine, but
            // done like this so we can defer close(c).
            go func() {
                <-d.Transfer.Released()
                if d.layer != nil {
                    layer.ReleaseAndLog(d.layerStore, d.layer)
                }
            }()

type downloadTransfer struct {
    Transfer

    layerStore layer.Store
    layer      layer.Layer
    err        error
}

func (ls *layerStore) Register(ts io.Reader, parent ChainID) (Layer, error) {
    // err is used to hold the error which will always trigger
    // cleanup of creates sources but may not be an error returned
    // to the caller (already exists).
    var err error
    var pid string
    var p *roLayer
    if string(parent) != "" {
        p = ls.get(parent)
        if p == nil {
            return nil, ErrLayerDoesNotExist
        }
        pid = p.cacheID
        // Release parent chain if error
        defer func() {
            if err != nil {
                ls.layerL.Lock()
                ls.releaseLayer(p)
                ls.layerL.Unlock()
            }
        }()
        if p.depth() >= maxLayerDepth {
            err = ErrMaxDepthExceeded
            return nil, err
        }
    }

    // Create new roLayer
    layer := &roLayer{
        parent:         p,
        cacheID:        stringid.GenerateRandomID(),
        referenceCount: 1,
        layerStore:     ls,
        references:     map[Layer]struct{}{},
    }

    if err = ls.driver.Create(layer.cacheID, pid, ""); err != nil {
        return nil, err
    }

    tx, err := ls.store.StartTransaction()
    if err != nil {
        return nil, err
    }

    defer func() {
        if err != nil {
            logrus.Debugf("Cleaning up layer %s: %v", layer.cacheID, err)
            if err := ls.driver.Remove(layer.cacheID); err != nil {
                logrus.Errorf("Error cleaning up cache layer %s: %v", layer.cacheID, err)
            }
            if err := tx.Cancel(); err != nil {
                logrus.Errorf("Error canceling metadata transaction %q: %s", tx.String(), err)
            }
        }
    }()

    if err = ls.applyTar(tx, ts, pid, layer); err != nil {
        return nil, err
    }

    if layer.parent == nil {
        layer.chainID = ChainID(layer.diffID)
    } else {
        layer.chainID = createChainIDFromParent(layer.parent.chainID, layer.diffID)
    }

    if err = storeLayer(tx, layer); err != nil {
        return nil, err
    }

    ls.layerL.Lock()
    defer ls.layerL.Unlock()

    if existingLayer := ls.getWithoutLock(layer.chainID); existingLayer != nil {
        // Set error for cleanup, but do not return the error
        err = errors.New("layer already exists")
        return existingLayer.getReference(), nil
    }

    if err = tx.Commit(layer.chainID); err != nil {
        return nil, err
    }

    ls.layerMap[layer.chainID] = layer

    return layer.getReference(), nil
}

```

docker端存储

```sh
===================
[root@CentOS-64-duyanghao ~]# find /* -name ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
/data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
/data/docker/image/aufs/layerdb/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a

[root@CentOS-64-duyanghao ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a]# ls -l
总用量 24
-rw-r--r-- 1 root root    64 10月 25 17:30 cache-id
-rw-r--r-- 1 root root    71 10月 25 17:30 diff
-rw-r--r-- 1 root root     9 10月 25 17:30 size
-rw-r--r-- 1 root root 11277 10月 25 17:30 tar-split.json.gz

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
[{"Digest":"sha256:c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]

===================
[root@CentOS-64-duyanghao ~]# find /* -name 5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
/data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef

[{"Digest":"sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]

===================
[root@CentOS-64-duyanghao ~]# find /* -name d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06  
/data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06
[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06

[{"Digest":"sha256:93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]
```

applyTar函数

```go
func (ls *layerStore) applyTar(tx MetadataTransaction, ts io.Reader, parent string, layer *roLayer) error {
    digester := digest.Canonical.New()
    tr := io.TeeReader(ts, digester.Hash())

    tsw, err := tx.TarSplitWriter(true)
    if err != nil {
        return err
    }
    metaPacker := storage.NewJSONPacker(tsw)
    defer tsw.Close()

    // we're passing nil here for the file putter, because the ApplyDiff will
    // handle the extraction of the archive
    rdr, err := asm.NewInputTarStream(tr, metaPacker, nil)
    if err != nil {
        return err
    }

    applySize, err := ls.driver.ApplyDiff(layer.cacheID, parent, archive.Reader(rdr))
    if err != nil {
        return err
    }

    // Discard trailing data but ensure metadata is picked up to reconstruct stream
    io.Copy(ioutil.Discard, rdr) // ignore error as reader may be closed

    layer.size = applySize
    layer.diffID = DiffID(digester.Digest())

    logrus.Debugf("Applied tar %s to %s, size: %d", layer.diffID, layer.cacheID, applySize)

    return nil
}
// NewJSONPacker provides a Packer that writes each Entry (SegmentType and
// FileType) as a json document.
//
// The Entries are delimited by new line.
func NewJSONPacker(w io.Writer) Packer {
    return &jsonPacker{
        w:    w,
        e:    json.NewEncoder(w),
        seen: seenNames{},
    }
}
// NewInputTarStream wraps the Reader stream of a tar archive and provides a
// Reader stream of the same.
//
// In the middle it will pack the segments and file metadata to storage.Packer
// `p`.
//
// The the storage.FilePutter is where payload of files in the stream are
// stashed. If this stashing is not needed, you can provide a nil
// storage.FilePutter. Since the checksumming is still needed, then a default
// of NewDiscardFilePutter will be used internally
func NewInputTarStream(r io.Reader, p storage.Packer, fp storage.FilePutter) (io.Reader, error) {
    // What to do here... folks will want their own access to the Reader that is
    // their tar archive stream, but we'll need that same stream to use our
    // forked 'archive/tar'.
    // Perhaps do an io.TeeReader that hands back an io.Reader for them to read
    // from, and we'll MITM the stream to store metadata.
    // We'll need a storage.FilePutter too ...

    // Another concern, whether to do any storage.FilePutter operations, such that we
    // don't extract any amount of the archive. But then again, we're not making
    // files/directories, hardlinks, etc. Just writing the io to the storage.FilePutter.
    // Perhaps we have a DiscardFilePutter that is a bit bucket.

    // we'll return the pipe reader, since TeeReader does not buffer and will
    // only read what the outputRdr Read's. Since Tar archives have padding on
    // the end, we want to be the one reading the padding, even if the user's
    // `archive/tar` doesn't care.
    pR, pW := io.Pipe()
    outputRdr := io.TeeReader(r, pW)

    // we need a putter that will generate the crc64 sums of file payloads
    if fp == nil {
        fp = storage.NewDiscardFilePutter()
    }

    go func() {
        tr := tar.NewReader(outputRdr)
        tr.RawAccounting = true
        for {
            hdr, err := tr.Next()
            if err != nil {
                if err != io.EOF {
                    pW.CloseWithError(err)
                    return
                }
                // even when an EOF is reached, there is often 1024 null bytes on
                // the end of an archive. Collect them too.
                if b := tr.RawBytes(); len(b) > 0 {
                    _, err := p.AddEntry(storage.Entry{
                        Type:    storage.SegmentType,
                        Payload: b,
                    })
                    if err != nil {
                        pW.CloseWithError(err)
                        return
                    }
                }
                break // not return. We need the end of the reader.
            }
            if hdr == nil {
                break // not return. We need the end of the reader.
            }

            if b := tr.RawBytes(); len(b) > 0 {
                _, err := p.AddEntry(storage.Entry{
                    Type:    storage.SegmentType,
                    Payload: b,
                })
                if err != nil {
                    pW.CloseWithError(err)
                    return
                }
            }

            var csum []byte
            if hdr.Size > 0 {
                var err error
                _, csum, err = fp.Put(hdr.Name, tr)
                if err != nil {
                    pW.CloseWithError(err)
                    return
                }
            }

            entry := storage.Entry{
                Type:    storage.FileType,
                Size:    hdr.Size,
                Payload: csum,
            }
            // For proper marshalling of non-utf8 characters
            entry.SetName(hdr.Name)

            // File entries added, regardless of size
            _, err = p.AddEntry(entry)
            if err != nil {
                pW.CloseWithError(err)
                return
            }

            if b := tr.RawBytes(); len(b) > 0 {
                _, err = p.AddEntry(storage.Entry{
                    Type:    storage.SegmentType,
                    Payload: b,
                })
                if err != nil {
                    pW.CloseWithError(err)
                    return
                }
            }
        }

        // it is allowable, and not uncommon that there is further padding on the
        // end of an archive, apart from the expected 1024 null bytes.
        remainder, err := ioutil.ReadAll(outputRdr)
        if err != nil && err != io.EOF {
            pW.CloseWithError(err)
            return
        }
        _, err = p.AddEntry(storage.Entry{
            Type:    storage.SegmentType,
            Payload: remainder,
        })
        if err != nil {
            pW.CloseWithError(err)
            return
        }
        pW.Close()
    }()

    return pR, nil
}
type layerStore struct {
    store  MetadataStore
    driver graphdriver.Driver

    layerMap map[ChainID]*roLayer
    layerL   sync.Mutex

    mounts map[string]*mountedLayer
    mountL sync.Mutex
}
// Driver is the interface for layered/snapshot file system drivers.
type Driver interface {
    ProtoDriver
    // Diff produces an archive of the changes between the specified
    // layer and its parent layer which may be "".
    Diff(id, parent string) (archive.Archive, error)
    // Changes produces a list of changes between the specified layer
    // and its parent layer. If parent is "", then all changes will be ADD changes.
    Changes(id, parent string) ([]archive.Change, error)
    // ApplyDiff extracts the changeset from the given diff into the
    // layer with the specified id and parent, returning the size of the
    // new layer in bytes.
    // The archive.Reader must be an uncompressed stream.
    ApplyDiff(id, parent string, diff archive.Reader) (size int64, err error)
    // DiffSize calculates the changes between the specified id
    // and its parent and returns the size in bytes of the changes
    // relative to its base filesystem directory.
    DiffSize(id, parent string) (size int64, err error)
}
```

**TarSplitWriter函数创建tar-split.json.gz文件，NewInputTarStream函数写tar-split.json.gz文件**

![](/public/img/docker-registry/2016-10-26-docker-registry-pull-manifest-v2/tar_split_json_gz.png)

storeLayer函数写diff、size、cache-id文件，如下：

```go
func storeLayer(tx MetadataTransaction, layer *roLayer) error {
    if err := tx.SetDiffID(layer.diffID); err != nil {
        return err
    }
    if err := tx.SetSize(layer.size); err != nil {
        return err
    }
    if err := tx.SetCacheID(layer.cacheID); err != nil {
        return err
    }
    if layer.parent != nil {
        if err := tx.SetParent(layer.parent.chainID); err != nil {
            return err
        }
    }

    return nil
}
func (fm *fileMetadataTransaction) SetDiffID(diff DiffID) error {
    return ioutil.WriteFile(filepath.Join(fm.root, "diff"), []byte(digest.Digest(diff).String()), 0644)
}
func (fm *fileMetadataTransaction) SetSize(size int64) error {
    content := fmt.Sprintf("%d", size)
    return ioutil.WriteFile(filepath.Join(fm.root, "size"), []byte(content), 0644)
}
func (fm *fileMetadataTransaction) SetCacheID(cacheID string) error {
    return ioutil.WriteFile(filepath.Join(fm.root, "cache-id"), []byte(cacheID), 0644)
}
func (fm *fileMetadataTransaction) SetParent(parent ChainID) error {
    return ioutil.WriteFile(filepath.Join(fm.root, "parent"), []byte(digest.Digest(parent).String()), 0644)
}
func (fm *fileMetadataTransaction) TarSplitWriter(compressInput bool) (io.WriteCloser, error) {
    f, err := os.OpenFile(filepath.Join(fm.root, "tar-split.json.gz"), os.O_TRUNC|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return nil, err
    }
    var wc io.WriteCloser
    if compressInput {
        wc = gzip.NewWriter(f)
    } else {
        wc = f
    }

    return ioutils.NewWriteCloserWrapper(wc, func() error {
        wc.Close()
        return f.Close()
    }), nil
}
```

/data/docker/image/aufs目录结构

```sh
[root@CentOS-64-duyanghao ~]# tree /data/docker/image/aufs
/data/docker/image/aufs
├── distribution
│   ├── diffid-by-digest
│   │   └── sha256
│   │       ├── 93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7
│   │       ├── a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
│   │       └── c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e
│   └── v2metadata-by-diffid
│       └── sha256
│           ├── 5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
│           ├── ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
│           └── d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06
├── imagedb
│   ├── content
│   │   └── sha256
│   │       └── 2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042
│   └── metadata
│       └── sha256
├── layerdb
│   ├── sha256
│   │   ├── 0af1c8e643b5b1985c93a0004b1e6b091e30d349bb7f005271d1d9ff23b70119
│   │   │   ├── cache-id
│   │   │   ├── diff
│   │   │   ├── parent
│   │   │   ├── size
│   │   │   └── tar-split.json.gz
│   │   ├── 75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6
│   │   │   ├── cache-id
│   │   │   ├── diff
│   │   │   ├── parent
│   │   │   ├── size
│   │   │   └── tar-split.json.gz
│   │   └── ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
│   │       ├── cache-id
│   │       ├── diff
│   │       ├── size
│   │       └── tar-split.json.gz
│   └── tmp
└── repositories.json

16 directories, 22 files
```

```sh
======
[root@CentOS-64-duyanghao aufs]# cat imagedb/content/sha256/2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042 
{"architecture":"amd64","config":{"Hostname":"xxxx","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["sh"],"ArgsEscaped":true,"Image":"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"container":"7dfa08cb9cbf2962b2362b1845b6657895685576015a8121652872fea56a7509","container_config":{"Hostname":"xxxx","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sh","-c","dd if=/dev/zero of=file bs=10M count=1"],"ArgsEscaped":true,"Image":"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"created":"2016-08-18T06:13:28.269459769Z","docker_version":"1.11.0-dev","history":[{"created":"2015-09-21T20:15:47.433616227Z","created_by":"/bin/sh -c #(nop) ADD file:6cccb5f0a3b3947116a0c0f55d071980d94427ba0d6dad17bc68ead832cc0a8f in /"},{"created":"2015-09-21T20:15:47.866196515Z","created_by":"/bin/sh -c #(nop) CMD [\"sh\"]"},{"created":"2016-08-18T06:13:28.269459769Z","created_by":"/bin/sh -c dd if=/dev/zero of=file bs=10M count=1"}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a","sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef","sha256:d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06"]}}

=====
[root@CentOS-64-duyanghao aufs]# cat distribution/diffid-by-digest/sha256/93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7 
sha256:d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06

[root@CentOS-64-duyanghao aufs]# cat distribution/diffid-by-digest/sha256/a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4 
sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef

[root@CentOS-64-duyanghao aufs]# cat distribution/diffid-by-digest/sha256/c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e 
sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a

=====
[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
[{"Digest":"sha256:c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
[{"Digest":"sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06
[{"Digest":"sha256:93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]

=====
[root@CentOS-64-duyanghao 0af1c8e643b5b1985c93a0004b1e6b091e30d349bb7f005271d1d9ff23b70119]# cat cache-id 
2be304721c0f40e5a4a3afc4081c5225e86077b7e2229b35676d14324fc5208f

[root@CentOS-64-duyanghao 0af1c8e643b5b1985c93a0004b1e6b091e30d349bb7f005271d1d9ff23b70119]# cat diff 
sha256:d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06

[root@CentOS-64-duyanghao 0af1c8e643b5b1985c93a0004b1e6b091e30d349bb7f005271d1d9ff23b70119]# cat parent 
sha256:75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6

[root@CentOS-64-duyanghao 0af1c8e643b5b1985c93a0004b1e6b091e30d349bb7f005271d1d9ff23b70119]# cat size 
10485760

[root@CentOS-64-duyanghao 0af1c8e643b5b1985c93a0004b1e6b091e30d349bb7f005271d1d9ff23b70119]# ls -l tar-split.json.gz 
-rw-r--r-- 1 root root 213 10月 25 17:30 tar-split.json.gz

=====
[root@CentOS-64-duyanghao 75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6]# cat cache-id 
43e3ac3e2c7b905f1b54ce7cdf2560e0e70457d1d1177c874739d788423fde83

[root@CentOS-64-duyanghao 75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6]# cat diff 
sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef

[root@CentOS-64-duyanghao 75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6]# cat parent 
sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a

[root@CentOS-64-duyanghao 75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6]# cat size 
0

[root@CentOS-64-duyanghao 75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6]# ls -l tar-split.json.gz 
-rw-r--r-- 1 root root 82 10月 25 17:30 tar-split.json.gz

=====
[root@CentOS-64-duyanghao ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a]# ls -l
总用量 24
-rw-r--r-- 1 root root    64 10月 25 17:30 cache-id
-rw-r--r-- 1 root root    71 10月 25 17:30 diff
-rw-r--r-- 1 root root     9 10月 25 17:30 size
-rw-r--r-- 1 root root 11277 10月 25 17:30 tar-split.json.gz

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/layerdb/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a/diff    
sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/layerdb/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a/cache-id 
1291dc82f80bd68a5ddb79db7164cf786209fe352394dc9e3db37d5acde44404

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/layerdb/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a/size 
364348077

=====
[root@CentOS-64-duyanghao aufs]# cat repositories.json    
{"Repositories":{"x.x.x.x:5000/duyanghao/busybox":{"x.x.x.x:5000/duyanghao/busybox:v0":"sha256:2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042"}}}
```

归纳；

```sh
├── layerdb
│   ├── sha256
│   │   ├── 0af1c8e643b5b1985c93a0004b1e6b091e30d349bb7f005271d1d9ff23b70119
│   │   │   ├── cache-id 2be304721c0f40e5a4a3afc4081c5225e86077b7e2229b35676d14324fc5208f
│   │   │   ├── diff sha256:d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06
│   │   │   ├── parent sha256:75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6
│   │   │   ├── size 10485760
│   │   │   └── tar-split.json.gz 213(文件大小)
│   │   ├── 75a46a4a46d9b53d8bbd70d52a26dc08858961f51156372edf6e8084ba9cfdb6
│   │   │   ├── cache-id 43e3ac3e2c7b905f1b54ce7cdf2560e0e70457d1d1177c874739d788423fde83
│   │   │   ├── diff sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
│   │   │   ├── parent sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
│   │   │   ├── size 0
│   │   │   └── tar-split.json.gz 82(文件大小)
│   │   └── ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
│   │       ├── cache-id 1291dc82f80bd68a5ddb79db7164cf786209fe352394dc9e3db37d5acde44404
│   │       ├── diff sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
│   │       ├── size 364348077
│   │       └── tar-split.json.gz 11277(文件大小)

三个sha256 hash，对应关系分别如下：
------------
ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
|
c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e

------------
5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
|
a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4

------------
d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06
|
93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7

[root@CentOS-64-duyanghao aufs]# cat repositories.json    
{"Repositories":{"x.x.x.x:5000/duyanghao/busybox":{"x.x.x.x:5000/duyanghao/busybox:v0":"sha256:2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042"}}}
```

关系图：

![](/public/img/docker-registry/2016-10-26-docker-registry-pull-manifest-v2/relation.png)

几个问题：

* parent sha256 hash怎么得到的？
* 数据文件内容如何写入？

**parent SHA256怎么得到的？**

```go
func (ls *layerStore) Register(ts io.Reader, parent ChainID) (Layer, error) {
    ...
    if layer.parent == nil {
        layer.chainID = ChainID(layer.diffID)
    } else {
        layer.chainID = createChainIDFromParent(layer.parent.chainID, layer.diffID)
    }
    ...
}
func createChainIDFromParent(parent ChainID, dgsts ...DiffID) ChainID {
    if len(dgsts) == 0 {
        return parent
    }
    if parent == "" {
        return createChainIDFromParent(ChainID(dgsts[0]), dgsts[1:]...)
    }
    // H = "H(n-1) SHA256(n)"
    dgst := digest.FromBytes([]byte(string(parent) + " " + string(dgsts[0])))
    return createChainIDFromParent(ChainID(dgst), dgsts[1:]...)
}
// ChainID is the content-addressable ID of a layer.
type ChainID digest.Digest

```

#### ID definitions and calculations

This table summarizes the different types of IDs involved and how they are calculated:

![](/public/img/docker-registry/2016-10-26-docker-registry-pull-manifest-v2/ID_definitions_and_calculations.png)

**Size大小**

```go
func (ls *layerStore) applyTar(tx MetadataTransaction, ts io.Reader, parent string, layer *roLayer) error {
    ...
    applySize, err := ls.driver.ApplyDiff(layer.cacheID, parent, archive.Reader(rdr))
    ...
}

// ApplyDiff extracts the changeset from the given diff into the
// layer with the specified id and parent, returning the size of the
// new layer in bytes.
func (a *Driver) ApplyDiff(id, parent string, diff archive.Reader) (size int64, err error) {
    // AUFS doesn't need the parent id to apply the diff.
    if err = a.applyDiff(id, diff); err != nil {
        return
    }

    return a.DiffSize(id, parent)
}
func (a *Driver) applyDiff(id string, diff archive.Reader) error {
    return chrootarchive.UntarUncompressed(diff, path.Join(a.rootPath(), "diff", id), &archive.TarOptions{
        UIDMaps: a.uidMaps,
        GIDMaps: a.gidMaps,
    })
}
// UntarUncompressed reads a stream of bytes from `archive`, parses it as a tar archive,
// and unpacks it into the directory at `dest`.
// The archive must be an uncompressed stream.
func UntarUncompressed(tarArchive io.Reader, dest string, options *archive.TarOptions) error {
    return untarHandler(tarArchive, dest, options, false)
}
// Handler for teasing out the automatic decompression
func untarHandler(tarArchive io.Reader, dest string, options *archive.TarOptions, decompress bool) error {

    if tarArchive == nil {
        return fmt.Errorf("Empty archive")
    }
    if options == nil {
        options = &archive.TarOptions{}
    }
    if options.ExcludePatterns == nil {
        options.ExcludePatterns = []string{}
    }

    rootUID, rootGID, err := idtools.GetRootUIDGID(options.UIDMaps, options.GIDMaps)
    if err != nil {
        return err
    }

    dest = filepath.Clean(dest)
    if _, err := os.Stat(dest); os.IsNotExist(err) {
        if err := idtools.MkdirAllNewAs(dest, 0755, rootUID, rootGID); err != nil {
            return err
        }
    }

    r := ioutil.NopCloser(tarArchive)
    if decompress {
        decompressedArchive, err := archive.DecompressStream(tarArchive)
        if err != nil {
            return err
        }
        defer decompressedArchive.Close()
        r = decompressedArchive
    }

    return invokeUnpack(r, dest, options)
}
func invokeUnpack(decompressedArchive io.ReadCloser,
    dest string,
    options *archive.TarOptions) error {
    // Windows is different to Linux here because Windows does not support
    // chroot. Hence there is no point sandboxing a chrooted process to
    // do the unpack. We call inline instead within the daemon process.
    return archive.Unpack(decompressedArchive, longpath.AddPrefix(dest), options)
}
// Unpack unpacks the decompressedArchive to dest with options.
func Unpack(decompressedArchive io.Reader, dest string, options *TarOptions) error {
    tr := tar.NewReader(decompressedArchive)
    trBuf := pools.BufioReader32KPool.Get(nil)
    defer pools.BufioReader32KPool.Put(trBuf)

    var dirs []*tar.Header
    remappedRootUID, remappedRootGID, err := idtools.GetRootUIDGID(options.UIDMaps, options.GIDMaps)
    if err != nil {
        return err
    }

    // Iterate through the files in the archive.
loop:
    for {
        hdr, err := tr.Next()
        if err == io.EOF {
            // end of tar archive
            break
        }
        if err != nil {
            return err
        }

        // Normalize name, for safety and for a simple is-root check
        // This keeps "../" as-is, but normalizes "/../" to "/". Or Windows:
        // This keeps "..\" as-is, but normalizes "\..\" to "\".
        hdr.Name = filepath.Clean(hdr.Name)

        for _, exclude := range options.ExcludePatterns {
            if strings.HasPrefix(hdr.Name, exclude) {
                continue loop
            }
        }

        // After calling filepath.Clean(hdr.Name) above, hdr.Name will now be in
        // the filepath format for the OS on which the daemon is running. Hence
        // the check for a slash-suffix MUST be done in an OS-agnostic way.
        if !strings.HasSuffix(hdr.Name, string(os.PathSeparator)) {
            // Not the root directory, ensure that the parent directory exists
            parent := filepath.Dir(hdr.Name)
            parentPath := filepath.Join(dest, parent)
            if _, err := os.Lstat(parentPath); err != nil && os.IsNotExist(err) {
                err = idtools.MkdirAllNewAs(parentPath, 0777, remappedRootUID, remappedRootGID)
                if err != nil {
                    return err
                }
            }
        }

        path := filepath.Join(dest, hdr.Name)
        rel, err := filepath.Rel(dest, path)
        if err != nil {
            return err
        }
        if strings.HasPrefix(rel, ".."+string(os.PathSeparator)) {
            return breakoutError(fmt.Errorf("%q is outside of %q", hdr.Name, dest))
        }

        // If path exits we almost always just want to remove and replace it
        // The only exception is when it is a directory *and* the file from
        // the layer is also a directory. Then we want to merge them (i.e.
        // just apply the metadata from the layer).
        if fi, err := os.Lstat(path); err == nil {
            if options.NoOverwriteDirNonDir && fi.IsDir() && hdr.Typeflag != tar.TypeDir {
                // If NoOverwriteDirNonDir is true then we cannot replace
                // an existing directory with a non-directory from the archive.
                return fmt.Errorf("cannot overwrite directory %q with non-directory %q", path, dest)
            }

            if options.NoOverwriteDirNonDir && !fi.IsDir() && hdr.Typeflag == tar.TypeDir {
                // If NoOverwriteDirNonDir is true then we cannot replace
                // an existing non-directory with a directory from the archive.
                return fmt.Errorf("cannot overwrite non-directory %q with directory %q", path, dest)
            }

            if fi.IsDir() && hdr.Name == "." {
                continue
            }

            if !(fi.IsDir() && hdr.Typeflag == tar.TypeDir) {
                if err := os.RemoveAll(path); err != nil {
                    return err
                }
            }
        }
        trBuf.Reset(tr)

        // if the options contain a uid & gid maps, convert header uid/gid
        // entries using the maps such that lchown sets the proper mapped
        // uid/gid after writing the file. We only perform this mapping if
        // the file isn't already owned by the remapped root UID or GID, as
        // that specific uid/gid has no mapping from container -> host, and
        // those files already have the proper ownership for inside the
        // container.
        if hdr.Uid != remappedRootUID {
            xUID, err := idtools.ToHost(hdr.Uid, options.UIDMaps)
            if err != nil {
                return err
            }
            hdr.Uid = xUID
        }
        if hdr.Gid != remappedRootGID {
            xGID, err := idtools.ToHost(hdr.Gid, options.GIDMaps)
            if err != nil {
                return err
            }
            hdr.Gid = xGID
        }

        if err := createTarFile(path, dest, hdr, trBuf, !options.NoLchown, options.ChownOpts); err != nil {
            return err
        }

        // Directory mtimes must be handled at the end to avoid further
        // file creation in them to modify the directory mtime
        if hdr.Typeflag == tar.TypeDir {
            dirs = append(dirs, hdr)
        }
    }

    for _, hdr := range dirs {
        path := filepath.Join(dest, hdr.Name)

        if err := system.Chtimes(path, hdr.AccessTime, hdr.ModTime); err != nil {
            return err
        }
    }
    return nil
}
// DiffSize calculates the changes between the specified layer
// and its parent and returns the size in bytes of the changes
// relative to its base filesystem directory.
func (d *Driver) DiffSize(id, parent string) (size int64, err error) {
    rPId, err := d.resolveID(parent)
    if err != nil {
        return
    }

    changes, err := d.Changes(id, rPId)
    if err != nil {
        return
    }

    layerFs, err := d.Get(id, "")
    if err != nil {
        return
    }
    defer d.Put(id)

    return archive.ChangesSize(layerFs, changes), nil
}
// ChangesSize calculates the size in bytes of the provided changes, based on newDir.
func ChangesSize(newDir string, changes []Change) int64 {
    var (
        size int64
        sf   = make(map[uint64]struct{})
    )
    for _, change := range changes {
        if change.Kind == ChangeModify || change.Kind == ChangeAdd {
            file := filepath.Join(newDir, change.Path)
            fileInfo, err := os.Lstat(file)
            if err != nil {
                logrus.Errorf("Can not stat %q: %s", file, err)
                continue
            }

            if fileInfo != nil && !fileInfo.IsDir() {
                if hasHardlinks(fileInfo) {
                    inode := getIno(fileInfo)
                    if _, ok := sf[inode]; !ok {
                        size += fileInfo.Size()
                        sf[inode] = struct{}{}
                    }
                } else {
                    size += fileInfo.Size()
                }
            }
        }
    }
    return size
}
```

/data/docker目录

```sh
[root@CentOS-64-duyanghao docker]# tree -L 4 -C
.
├── aufs
│   ├── diff
│   │   ├── 1291dc82f80bd68a5ddb79db7164cf786209fe352394dc9e3db37d5acde44404
│   │   │   ├── bin
│   │   │   ├── dev
│   │   │   ├── etc
│   │   │   ├── home
│   │   │   ├── root
│   │   │   ├── tmp
│   │   │   ├── usr
│   │   │   └── var
│   │   ├── 2be304721c0f40e5a4a3afc4081c5225e86077b7e2229b35676d14324fc5208f
│   │   │   └── file
│   │   └── 43e3ac3e2c7b905f1b54ce7cdf2560e0e70457d1d1177c874739d788423fde83
│   ├── layers
│   │   ├── 1291dc82f80bd68a5ddb79db7164cf786209fe352394dc9e3db37d5acde44404
│   │   ├── 2be304721c0f40e5a4a3afc4081c5225e86077b7e2229b35676d14324fc5208f
│   │   └── 43e3ac3e2c7b905f1b54ce7cdf2560e0e70457d1d1177c874739d788423fde83
│   └── mnt
│       ├── 1291dc82f80bd68a5ddb79db7164cf786209fe352394dc9e3db37d5acde44404
│       ├── 2be304721c0f40e5a4a3afc4081c5225e86077b7e2229b35676d14324fc5208f
│       └── 43e3ac3e2c7b905f1b54ce7cdf2560e0e70457d1d1177c874739d788423fde83
├── containers
├── image
│   └── aufs
│       ├── distribution
│       │   ├── diffid-by-digest
│       │   └── v2metadata-by-diffid
│       ├── imagedb
│       │   ├── content
│       │   └── metadata
│       ├── layerdb
│       │   ├── sha256
│       │   └── tmp
│       └── repositories.json
├── network
│   └── files
│       └── local-kv.db
├── tmp
├── trust
└── volumes
    └── metadata.db
```

registry v2目录结构

```sh
[root@CentOS-64-duyanghao registry_test]# tree
.
└── docker
    └── registry
        └── v2
            ├── blobs
            │   └── sha256
            │       ├── 2b
            │       │   └── 2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042
            │       │       └── data
            │       ├── 93
            │       │   └── 93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7
            │       │       └── data
            │       ├── a3
            │       │   └── a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
            │       │       └── data
            │       ├── c0
            │       │   └── c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e
            │       │       └── data
            │       └── d5
            │           └── d5ab5a18ba5a252216a930976e7a1d22ec6c4bb40d600df5dcea8714ca7973bc
            │               └── data
            └── repositories
                └── duyanghao
                    └── busybox
                        ├── _layers
                        │   └── sha256
                        │       ├── 2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042
                        │       │   └── link
                        │       ├── 93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7
                        │       │   └── link
                        │       ├── a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
                        │       │   └── link
                        │       └── c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e
                        │           └── link
                        ├── _manifests
                        │   ├── revisions
                        │   │   └── sha256
                        │   │       └── d5ab5a18ba5a252216a930976e7a1d22ec6c4bb40d600df5dcea8714ca7973bc
                        │   │           └── link
                        │   └── tags
                        │       └── v0
                        │           ├── current
                        │           │   └── link
                        │           └── index
                        │               └── sha256
                        │                   └── d5ab5a18ba5a252216a930976e7a1d22ec6c4bb40d600df5dcea8714ca7973bc
                        │                       └── link
                        └── _uploads

35 directories, 12 files
```

docker pull函数调用流程：

![](/public/img/docker-registry/2016-10-26-docker-registry-pull-manifest-v2/pull_function_process.png)

Cache mapping from this layer's DiffID to the blobsum

```sh
[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
[{"Digest":"sha256:c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
[{"Digest":"sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]

[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06
[{"Digest":"sha256:93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]
```

```go
func (ld *v2LayerDescriptor) Registered(diffID layer.DiffID) {
    // Cache mapping from this layer's DiffID to the blobsum
    ld.V2MetadataService.Add(diffID, metadata.V2Metadata{Digest: ld.digest, SourceRepository: ld.repoInfo.FullName()})
}

```

总结：

* 1 **<font color="#8B0000">layer.DiffID</font>**表示单个layer的ID（唯一标识该layer），计算公式：

>**`DiffID = SHA256hex(uncompressed layer tar data)`**

* 2 **<font color="#8B0000">layer.Digest</font>**表示单个layer的**压缩**ID，计算公式：

>**`DiffID = SHA256hex(compressed layer tar data)`**

* 3 **<font color="#8B0000">layer.ChainID也即parent</font>**，表示该layer以及`parent layer`的ID（唯一标识以该layer为叶子的`layers tree`）计算公式：

>For bottom layer: **`ChainID(layer0) = DiffID(layer0)`**

>For other layers：**`ChainID(layerN) = SHA256hex(ChainID(layerN-1) + " " + DiffID(layerN))`**

* 4 **<font color="#8B0000">image.ID</font>**表示镜像配置的ID，由于镜像配置包含了镜像layer和使用信息，所以`image.ID`也唯一标识该镜像，计算公式：

>**`SHA256hex(imageConfigJSON)`**

* 5 `/data/docker/image/aufs/distribution/v2metadata-by-diffid`目录下记录map：`layer.DiffID`-> `blobsum`

```sh
[root@CentOS-64-duyanghao ~]# cat /data/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
[{"Digest":"sha256:c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e","SourceRepository":"x.x.x.x:5000/duyanghao/busybox"}]
```

* 6 `/data/docker/image/aufs/distribution/diffid-by-digest`目录下记录map：`digest`(**<font color="#8B0000">SHA256hex(compressed layer tar data)</font>**) -> `layer.DiffID`(**<font color="#8B0000">SHA256hex(uncompressed layer tar data)</font>**)

```sh
[root@CentOS-64-duyanghao aufs]# cat distribution/diffid-by-digest/sha256/c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e 
sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a
```

* 7 `/data/docker/image/aufs/imagedb/content/`目录下存放镜像**`configuration`**信息

* 8 `/data/docker/image/aufs/layerdb/sha256/`目录下存放镜像各**layer元数据信息**

* 9 `/data/docker/aufs/diff/`目录下存放各layer的**<font color="#8B0000">uncompressed untar data</font>**

几个问题：

* 1 `cache-id`怎么生成，有何意义？

* 2 `size`文件内容如何得来，是什么大小，size(`uncompressed layer tar data`)?

* 3 `tar-split.json.gz`如何计算生成，有何意义？

* 4 distribution/**diffid-by-digest**/sha256目录文件如何生成（在哪里进行生成）？

* 5 `repositories.json`如何生成（在哪里进行生成）？

* 6 **[Manifest Schema v1](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md)为什么不安全，相比而言，[Manifest Schema v2](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md)有什么优点？**

***下面分析[Manifest Schema v1](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md)（docker 1.10-）与[Manifest Schema v2](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md)的不同，实际上也是分析两种不同的镜像数据模型***

Manifest Schema v1对应的数据模型：

![](/public/img/docker-registry/2016-10-26-docker-registry-pull-manifest-v2/Manifest_Schema_v1.png)

>The current manifest format has two big problems which contributed to the security issues. First, it is not truly content addressable, since the digest which identifies it is only taken over a portion of the manifest. Second, it includes a “v1compatibility” string for each FS layer. This ties the format to v1 identifiers, and a one-to-one mapping between layers and configurations; both of which are problematic.

>Docker 1.10 adds new manifest format that corrects these problems. The manifest consists of a list of layers and a single configuration. The digest of the manifest is simply the hash of the serialized manifest. We add an image configuration object that completely covers both its configration and root filesystem, making it possible to use the hash of the configuration as a content addressable ID of the image.

>The new format allows end-to-end content addressability. The existing v2 manifest format puts image configurations in "v1Compatibility" strings that use the same data model and non-content-addressable ID scheme that the legacy v1 protocol uses. Supporting this format with the content addressable image/layer model in Docker 1.10 involves hacks to do things like generate fake v1 IDs. The new format carries the actual image configuration as a blob, so push/pull transfers an exact copy of the image.

>One nice side effect of this PR is that the hacks described above for assembling legacy manifests are moved out of the engine code into vendored distribution code. The distribution APIs now have a ManifestBuilder interface that abstracts away the job of creating a manifest in either the old or new format.

>Schema 1 is designed for a model where every layer of an image is actually a runnable image. That's not how Docker works anymore. Since the switch to content addressability, layers are just filesystem diffs. When the image model changed, it made sense to switch to a different manifest format that makes sense for this model.

>As mentioned above, schema 1 treats every layer like a separate image. Each layer has a v1compatibility string which includes an id key. That's the pre-1.10 image ID. But this isn't the ID that recent versions of Docker will use. Now that images are content-addressable, the ID is based on a hash of the configuration and layers. Basically Docker 1.10+ has to migrate schema1 images to the new format. That's why we replaced schema1.

**<font color="#8B0000">总结如下区分</font>**：

* 1、`Manifest Schema v2`的`image.ID`唯一标识该镜像（包括配置和layer信息），而`Manifest Schema v1`的镜像ID是随机生成的，不能标识镜像，存在伪造镜像ID的风险

* 2、`Manifest Schema v1`的每个layer都关联着一个配置configuration，它将每个layer都看作一个单独的镜像（具有随机生成的镜像ID），而这是不符合docker工作原理的，并不是每个layer都是image

* 3、`Manifest Schema v2`将layer数据与镜像`configuration`分开，结构更清晰合理，有利于后续扩展。镜像`configuration`包含镜像的配置信息以及组成镜像各layer的数据标识`layer.DiffID`；每个layer数据不再与配置关联，而是具有自己独立的内容唯一标识`layer.DIffID`和对应的树形内容（filesystem composed of a set of layers）唯一标识`layer.ChainID`

下面给出docker 1.10+[兼容Manifest Schema v1](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md#backward-compatibility)部分代码分析：

>If the manifest being requested uses the new format, and the appropriate media type is not present in an Accept header, the registry will assume that the client cannot handle the manifest as-is, and rewrite it on the fly into the old format. If the object that would otherwise be returned is a manifest list, the registry will look up the appropriate manifest for the amd64 platform and linux OS, rewrite that manifest into the old format if necessary, and return the result to the client. If no suitable manifest is found in the manifest list, the registry will return a 404 error.

>One of the challenges in rewriting manifests to the old format is that the old format involves an image configuration for each layer in the manifest, but the new format only provides one image configuration. To work around this, the registry will create synthetic image configurations for all layers except the top layer. These image configurations will not result in runnable images on their own, but only serve to fill in the parent chain in a compatible way. The IDs in these synthetic configurations will be derived from hashes of their respective blobs. The registry will create these configurations and their IDs using the same scheme as Docker 1.10 when it creates a legacy manifest to push to a registry which doesn't support the new format.

```sh
[root@CentOS-64-duyanghao docker]# curl -v  http://x.x.x.x:5000/v2/duyanghao/busybox/manifests/v0 

< HTTP/1.1 200 OK
< Content-Length: 3209
< Content-Type: application/vnd.docker.distribution.manifest.v1+prettyjws
< Docker-Content-Digest: sha256:d5ab5a18ba5a252216a930976e7a1d22ec6c4bb40d600df5dcea8714ca7973bc
< Docker-Distribution-Api-Version: registry/2.0
< Etag: "sha256:d5ab5a18ba5a252216a930976e7a1d22ec6c4bb40d600df5dcea8714ca7973bc"
< X-Content-Type-Options: nosniff
< Date: Fri, 28 Oct 2016 03:55:58 GMT
{
   "schemaVersion": 1,
   "name": "duyanghao/busybox",
   "tag": "v0",
   "architecture": "amd64",
   "fsLayers": [
      {
         "blobSum": "sha256:93eea0ce9921b81687ad054452396461f29baf653157c368cd347f9caa6e58f7"
      },
      {
         "blobSum": "sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4"
      },
      {
         "blobSum": "sha256:c0a04912aa5afc0b4fd4c34390e526d547e67431f6bc122084f1e692dcb7d34e"
      }
   ],
   "history": [
      {
         "v1Compatibility": "{\"architecture\":\"amd64\",\"config\":{\"Hostname\":\"xxxx\",\"Domainname\":\"\",\"User\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"sh\"],\"ArgsEscaped\":true,\"Image\":\"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"OnBuild\":[],\"Labels\":{}},\"container\":\"7dfa08cb9cbf2962b2362b1845b6657895685576015a8121652872fea56a7509\",\"container_config\":{\"Hostname\":\"xxxx\",\"Domainname\":\"\",\"User\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"/bin/sh\",\"-c\",\"dd if=/dev/zero of=file bs=10M count=1\"],\"ArgsEscaped\":true,\"Image\":\"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"OnBuild\":[],\"Labels\":{}},\"created\":\"2016-08-18T06:13:28.269459769Z\",\"docker_version\":\"1.11.0-dev\",\"id\":\"4b24c0d5113fac3c749381476dbe37ce56be2f20d4e75f481ef83657e4d25f65\",\"os\":\"linux\",\"parent\":\"14295db6daa4e1068fa3fabcf618fcda9a22f4a154da17e75f62bc2973c14f2c\"}"
      },
      {
         "v1Compatibility": "{\"id\":\"14295db6daa4e1068fa3fabcf618fcda9a22f4a154da17e75f62bc2973c14f2c\",\"parent\":\"77106241d10a8ed96dc42f690453f89ec95890b1494ccac18bac69065e4b67fa\",\"created\":\"2015-09-21T20:15:47.866196515Z\",\"container_config\":{\"Cmd\":[\"/bin/sh -c #(nop) CMD [\\\"sh\\\"]\"]}}"
      },
      {
         "v1Compatibility": "{\"id\":\"77106241d10a8ed96dc42f690453f89ec95890b1494ccac18bac69065e4b67fa\",\"created\":\"2015-09-21T20:15:47.433616227Z\",\"container_config\":{\"Cmd\":[\"/bin/sh -c #(nop) ADD file:6cccb5f0a3b3947116a0c0f55d071980d94427ba0d6dad17bc68ead832cc0a8f in /\"]}}"
      }
   ],
   "signatures": [
      {
         "header": {
            "jwk": {
               "crv": "P-256",
               "kid": "DS2C:VHKH:RLH3:NQLE:WXLY:OZPL:XIY3:ZJF4:PROG:7OX4:YHTD:2PWI",
               "kty": "EC",
               "x": "zbcubSTub3_q7W7b1VVT_ImvuVZpp_xmGdoN9M7DGvw",
               "y": "Us2z0GVXcM3CtfxJW76bnYC4gRRkZtB2OgV_Xkz7R9I"
            },
            "alg": "ES256"
         },
         "signature": "0wGyo5x7Xoik8ozwJbkjE6RzRF1JRYcTEABMYH-tsHbQtbP7vIPCyVQ7dWROgy-FIMGWfXXkvDirxgHpbwl5dw",
         "protected": "eyJmb3JtYXRMZW5ndGgiOjI1NjIsImZvcm1hdFRhaWwiOiJDbjAiLCJ0aW1lIjoiMjAxNi0xMC0yOFQwMzo1NTo1OFoifQ"
      }
   ]
}
```

```go
func (p *v2Puller) pullV2Tag(ctx context.Context, ref reference.Named) (tagUpdated bool, err error) {
    manSvc, err := p.repo.Manifests(ctx)
    if err != nil {
        return false, err
    }

    var (
        manifest    distribution.Manifest
        tagOrDigest string // Used for logging/progress only
    )
    if tagged, isTagged := ref.(reference.NamedTagged); isTagged {
        // NOTE: not using TagService.Get, since it uses HEAD requests
        // against the manifests endpoint, which are not supported by
        // all registry versions.
        manifest, err = manSvc.Get(ctx, "", client.WithTag(tagged.Tag()))
        if err != nil {
            return false, allowV1Fallback(err)
        }
        tagOrDigest = tagged.Tag()
    } else if digested, isDigested := ref.(reference.Canonical); isDigested {
        manifest, err = manSvc.Get(ctx, digested.Digest())
        if err != nil {
            return false, err
        }
        tagOrDigest = digested.Digest().String()
    } else {
        return false, fmt.Errorf("internal error: reference has neither a tag nor a digest: %s", ref.String())
    }

    if manifest == nil {
        return false, fmt.Errorf("image manifest does not exist for tag or digest %q", tagOrDigest)
    }

    // If manSvc.Get succeeded, we can be confident that the registry on
    // the other side speaks the v2 protocol.
    p.confirmedV2 = true

    logrus.Debugf("Pulling ref from V2 registry: %s", ref.String())
    progress.Message(p.config.ProgressOutput, tagOrDigest, "Pulling from "+p.repo.Named().Name())

    var (
        imageID        image.ID
        manifestDigest digest.Digest
    )

    switch v := manifest.(type) {
    case *schema1.SignedManifest:
        imageID, manifestDigest, err = p.pullSchema1(ctx, ref, v)
        if err != nil {
            return false, err
        }
    case *schema2.DeserializedManifest:
        imageID, manifestDigest, err = p.pullSchema2(ctx, ref, v)
        if err != nil {
            return false, err
        }
    case *manifestlist.DeserializedManifestList:
        imageID, manifestDigest, err = p.pullManifestList(ctx, ref, v)
        if err != nil {
            return false, err
        }
    default:
        return false, errors.New("unsupported manifest format")
    }

    progress.Message(p.config.ProgressOutput, "", "Digest: "+manifestDigest.String())

    oldTagImageID, err := p.config.ReferenceStore.Get(ref)
    if err == nil {
        if oldTagImageID == imageID {
            return false, nil
        }
    } else if err != reference.ErrDoesNotExist {
        return false, err
    }

    if canonical, ok := ref.(reference.Canonical); ok {
        if err = p.config.ReferenceStore.AddDigest(canonical, imageID, true); err != nil {
            return false, err
        }
    } else if err = p.config.ReferenceStore.AddTag(ref, imageID, true); err != nil {
        return false, err
    }

    return true, nil
}
func (p *v2Puller) pullSchema1(ctx context.Context, ref reference.Named, unverifiedManifest *schema1.SignedManifest) (imageID image.ID, manifestDigest digest.Digest, err error) {
    var verifiedManifest *schema1.Manifest
    verifiedManifest, err = verifySchema1Manifest(unverifiedManifest, ref)
    if err != nil {
        return "", "", err
    }

    rootFS := image.NewRootFS()

    if err := detectBaseLayer(p.config.ImageStore, verifiedManifest, rootFS); err != nil {
        return "", "", err
    }

    // remove duplicate layers and check parent chain validity
    err = fixManifestLayers(verifiedManifest)
    if err != nil {
        return "", "", err
    }

    var descriptors []xfer.DownloadDescriptor

    // Image history converted to the new format
    var history []image.History

    // Note that the order of this loop is in the direction of bottom-most
    // to top-most, so that the downloads slice gets ordered correctly.
    for i := len(verifiedManifest.FSLayers) - 1; i >= 0; i-- {
        blobSum := verifiedManifest.FSLayers[i].BlobSum

        var throwAway struct {
            ThrowAway bool `json:"throwaway,omitempty"`
        }
        if err := json.Unmarshal([]byte(verifiedManifest.History[i].V1Compatibility), &throwAway); err != nil {
            return "", "", err
        }

        h, err := v1.HistoryFromConfig([]byte(verifiedManifest.History[i].V1Compatibility), throwAway.ThrowAway)
        if err != nil {
            return "", "", err
        }
        history = append(history, h)

        if throwAway.ThrowAway {
            continue
        }

        layerDescriptor := &v2LayerDescriptor{
            digest:            blobSum,
            repoInfo:          p.repoInfo,
            repo:              p.repo,
            V2MetadataService: p.V2MetadataService,
        }

        descriptors = append(descriptors, layerDescriptor)
    }

    resultRootFS, release, err := p.config.DownloadManager.Download(ctx, *rootFS, descriptors, p.config.ProgressOutput)
    if err != nil {
        return "", "", err
    }
    defer release()

    config, err := v1.MakeConfigFromV1Config([]byte(verifiedManifest.History[0].V1Compatibility), &resultRootFS, history)
    if err != nil {
        return "", "", err
    }

    imageID, err = p.config.ImageStore.Create(config)
    if err != nil {
        return "", "", err
    }

    manifestDigest = digest.FromBytes(unverifiedManifest.Canonical)

    return imageID, manifestDigest, nil
}
```

imageID生成：

```go
...
config, err := v1.MakeConfigFromV1Config([]byte(verifiedManifest.History[0].V1Compatibility), &resultRootFS, history)
...
imageID, err = p.config.ImageStore.Create(config)
```

```go
// MakeConfigFromV1Config creates an image config from the legacy V1 config format.
func MakeConfigFromV1Config(imageJSON []byte, rootfs *image.RootFS, history []image.History) ([]byte, error) {
    var dver struct {
        DockerVersion string `json:"docker_version"`
    }

    if err := json.Unmarshal(imageJSON, &dver); err != nil {
        return nil, err
    }

    useFallback := version.Version(dver.DockerVersion).LessThan(noFallbackMinVersion)

    if useFallback {
        var v1Image image.V1Image
        err := json.Unmarshal(imageJSON, &v1Image)
        if err != nil {
            return nil, err
        }
        imageJSON, err = json.Marshal(v1Image)
        if err != nil {
            return nil, err
        }
    }

    var c map[string]*json.RawMessage
    if err := json.Unmarshal(imageJSON, &c); err != nil {
        return nil, err
    }

    delete(c, "id")
    delete(c, "parent")
    delete(c, "Size") // Size is calculated from data on disk and is inconsistent
    delete(c, "parent_id")
    delete(c, "layer_id")
    delete(c, "throwaway")

    c["rootfs"] = rawJSON(rootfs)
    c["history"] = rawJSON(history)

    return json.Marshal(c)
}

```


### 参考

* [Manifest Schema v2 design](https://gist.github.com/aaronlehmann/b42a2eaf633fc949f93b#removed-fields)

* [Manifest Schema v1](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md)

* [Manifest Schema v2](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md)

* [issues/22225](https://github.com/docker/docker/issues/22225)
