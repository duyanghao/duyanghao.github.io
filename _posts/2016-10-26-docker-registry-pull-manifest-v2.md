---
layout: post
title: docker pull分析
date: 2016-10-26 17:36:11
category: 技术
tags: Docker-registry Docker Docker-Manifest-v2
excerpt: Image Manifest Version 2, Schema 2
---

总结`docker pull`流程中`Image Manifest Version 2, Schema 2`原理（以[docker 1.11.0](https://github.com/docker/docker/tree/v1.11.0)为分析版本）

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

