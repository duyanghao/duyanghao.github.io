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
```

如下：

```
> Accept: application/vnd.docker.distribution.manifest.v2+json
```

```
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
```

镜像configuration文件存储位置：


>/var/lib/docker/image/aufs/imagedb/content/sha256/2b519bd204483370e81176d98fd0c9bc4632e156da7b2cc752fa383b96e7c042

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
```




### 参考

[Manifest Schema v2 design](https://gist.github.com/aaronlehmann/b42a2eaf633fc949f93b#removed-fields)
[Manifest Schema v1](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md)
[Manifest Schema v2](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md)
[issues/22225](https://github.com/docker/docker/issues/22225)
