---
layout: post
title: Golang SetReadDeadline……
date: 2017-3-7 18:10:31
category: 技术
tags: Docker Go
excerpt: Golang SetReadDeadline……
---

以[Docker v1.3](https://github.com/duyanghao/docker/tree/v1.3.3)和[Go 1.3](https://github.com/golang/go/tree/release-branch.go1.3)为例分析Golang `SetReadDeadline`:

docker pull逻辑：

```go
func (s *TagStore) Install(eng *engine.Engine) error {
	for name, handler := range map[string]engine.Handler{
		"image_set":      s.CmdSet,
		"image_tag":      s.CmdTag,
		"tag":            s.CmdTagLegacy, // FIXME merge with "image_tag"
		"image_get":      s.CmdGet,
		"image_inspect":  s.CmdLookup,
		"image_tarlayer": s.CmdTarLayer,
		"image_export":   s.CmdImageExport,
		"apply_pull":     s.CmdPullAndApply,
		"apply_diff":     s.CmdDiffAndApply,
		"history":        s.CmdHistory,
		"images":         s.CmdImages,
		"viz":            s.CmdViz,
		"load":           s.CmdLoad,
		"import":         s.CmdImport,
		"pull":           s.CmdPull,
		"push":           s.CmdPush,
	} {
		if err := eng.Register(name, handler); err != nil {
			return fmt.Errorf("Could not register %q: %v", name, err)
		}
	}
	return nil
}

func (s *TagStore) CmdPull(job *engine.Job) engine.Status {
	if n := len(job.Args); n != 1 && n != 2 {
		return job.Errorf("Usage: %s IMAGE [TAG]", job.Name)
	}

	var (
		localName   = job.Args[0]
		tag         string
		sf          = utils.NewStreamFormatter(job.GetenvBool("json"))
		authConfig  = &registry.AuthConfig{}
		metaHeaders map[string][]string
		mirrors     []string
	)

	if len(job.Args) > 1 {
		tag = job.Args[1]
	}

	job.GetenvJson("authConfig", authConfig)
	job.GetenvJson("metaHeaders", &metaHeaders)

	c, err := s.poolAdd("pull", localName+":"+tag)
	if err != nil {
		if c != nil {
			// Another pull of the same repository is already taking place; just wait for it to finish
			job.Stdout.Write(sf.FormatStatus("", "Repository %s already being pulled by another client. Waiting.", localName))
			<-c
			return engine.StatusOK
		}
		return job.Error(err)
	}
	defer s.poolRemove("pull", localName+":"+tag)

	// Resolve the Repository name from fqn to endpoint + name
	hostname, remoteName, err := registry.ResolveRepositoryName(localName)
	if err != nil {
		return job.Error(err)
	}

	endpoint, err := registry.NewEndpoint(hostname, s.insecureRegistries)
	if err != nil {
		return job.Error(err)
	}

	r, err := registry.NewSession(authConfig, registry.HTTPRequestFactory(metaHeaders), endpoint, true)
	if err != nil {
		return job.Error(err)
	}

	var isOfficial bool
	if endpoint.VersionString(1) == registry.IndexServerAddress() {
		// If pull "index.docker.io/foo/bar", it's stored locally under "foo/bar"
		localName = remoteName

		isOfficial = isOfficialName(remoteName)
		if isOfficial && strings.IndexRune(remoteName, '/') == -1 {
			remoteName = "library/" + remoteName
		}
	}
	// Use provided mirrors, if any
	mirrors = s.mirrors

	if len(mirrors) == 0 && (isOfficial || endpoint.Version == registry.APIVersion2) {
		j := job.Eng.Job("trust_update_base")
		if err = j.Run(); err != nil {
			return job.Errorf("error updating trust base graph: %s", err)
		}

		if err := s.pullV2Repository(job.Eng, r, job.Stdout, localName, remoteName, tag, sf, job.GetenvBool("parallel")); err == nil {
			return engine.StatusOK
		} else if err != registry.ErrDoesNotExist {
			log.Errorf("Error from V2 registry: %s", err)
		}
	}

	if err = s.pullRepository(r, job.Stdout, localName, remoteName, tag, sf, job.GetenvBool("parallel"), mirrors); err != nil {
		return job.Error(err)
	}

	return engine.StatusOK
}

type Session struct {
	authConfig    *AuthConfig
	reqFactory    *utils.HTTPRequestFactory
	indexEndpoint *Endpoint
	jar           *cookiejar.Jar
	timeout       TimeoutType
}

func NewSession(authConfig *AuthConfig, factory *utils.HTTPRequestFactory, endpoint *Endpoint, timeout bool) (r *Session, err error) {
	r = &Session{
		authConfig:    authConfig,
		indexEndpoint: endpoint,
	}

	if timeout {
		r.timeout = ReceiveTimeout
	}

	r.jar, err = cookiejar.New(nil)
	if err != nil {
		return nil, err
	}

	// If we're working with a standalone private registry over HTTPS, send Basic Auth headers
	// alongside our requests.
	if r.indexEndpoint.VersionString(1) != IndexServerAddress() && r.indexEndpoint.URL.Scheme == "https" {
		info, err := r.indexEndpoint.Ping()
		if err != nil {
			return nil, err
		}
		if info.Standalone {
			log.Debugf("Endpoint %s is eligible for private registry registry. Enabling decorator.", r.indexEndpoint.String())
			dec := utils.NewHTTPAuthDecorator(authConfig.Username, authConfig.Password)
			factory.AddDecorator(dec)
		}
	}

	r.reqFactory = factory
	return r, nil
}

type TimeoutType uint32

const (
	NoTimeout TimeoutType = iota
	ReceiveTimeout
	ConnectTimeout
)

func (s *TagStore) pullRepository(r *registry.Session, out io.Writer, localName, remoteName, askedTag string, sf *utils.StreamFormatter, parallel bool, mirrors []string) error {
	out.Write(sf.FormatStatus("", "Pulling repository %s", localName))

	repoData, err := r.GetRepositoryData(remoteName)
	if err != nil {
		if strings.Contains(err.Error(), "HTTP code: 404") {
			return fmt.Errorf("Error: image %s not found", remoteName)
		}
		// Unexpected HTTP error
		return err
	}

	log.Debugf("Retrieving the tag list")
	tagsList, err := r.GetRemoteTags(repoData.Endpoints, remoteName, repoData.Tokens)
	if err != nil {
		log.Errorf("%v", err)
		return err
	}

	for tag, id := range tagsList {
		repoData.ImgList[id] = &registry.ImgData{
			ID:       id,
			Tag:      tag,
			Checksum: "",
		}
	}

	log.Debugf("Registering tags")
	// If no tag has been specified, pull them all
	var imageId string
	if askedTag == "" {
		for tag, id := range tagsList {
			repoData.ImgList[id].Tag = tag
		}
	} else {
		// Otherwise, check that the tag exists and use only that one
		id, exists := tagsList[askedTag]
		if !exists {
			return fmt.Errorf("Tag %s not found in repository %s", askedTag, localName)
		}
		imageId = id
		repoData.ImgList[id].Tag = askedTag
	}

	errors := make(chan error)

	layers_downloaded := false
	for _, image := range repoData.ImgList {
		downloadImage := func(img *registry.ImgData) {
			if askedTag != "" && img.Tag != askedTag {
				log.Debugf("(%s) does not match %s (id: %s), skipping", img.Tag, askedTag, img.ID)
				if parallel {
					errors <- nil
				}
				return
			}

			if img.Tag == "" {
				log.Debugf("Image (id: %s) present in this repository but untagged, skipping", img.ID)
				if parallel {
					errors <- nil
				}
				return
			}

			// ensure no two downloads of the same image happen at the same time
			if c, err := s.poolAdd("pull", "img:"+img.ID); err != nil {
				if c != nil {
					out.Write(sf.FormatProgress(utils.TruncateID(img.ID), "Layer already being pulled by another client. Waiting.", nil))
					<-c
					out.Write(sf.FormatProgress(utils.TruncateID(img.ID), "Download complete", nil))
				} else {
					log.Debugf("Image (id: %s) pull is already running, skipping: %v", img.ID, err)
				}
				if parallel {
					errors <- nil
				}
				return
			}
			defer s.poolRemove("pull", "img:"+img.ID)

			out.Write(sf.FormatProgress(utils.TruncateID(img.ID), fmt.Sprintf("Pulling image (%s) from %s", img.Tag, localName), nil))
			success := false
			var lastErr, err error
			var is_downloaded bool
			if mirrors != nil {
				for _, ep := range mirrors {
					out.Write(sf.FormatProgress(utils.TruncateID(img.ID), fmt.Sprintf("Pulling image (%s) from %s, mirror: %s", img.Tag, localName, ep), nil))
					if is_downloaded, err = s.pullImage(r, out, img.ID, ep, repoData.Tokens, sf); err != nil {
						// Don't report errors when pulling from mirrors.
						log.Debugf("Error pulling image (%s) from %s, mirror: %s, %s", img.Tag, localName, ep, err)
						continue
					}
					layers_downloaded = layers_downloaded || is_downloaded
					success = true
					break
				}
			}
			if !success {
				for _, ep := range repoData.Endpoints {
					out.Write(sf.FormatProgress(utils.TruncateID(img.ID), fmt.Sprintf("Pulling image (%s) from %s, endpoint: %s", img.Tag, localName, ep), nil))
					if is_downloaded, err = s.pullImage(r, out, img.ID, ep, repoData.Tokens, sf); err != nil {
						// It's not ideal that only the last error is returned, it would be better to concatenate the errors.
						// As the error is also given to the output stream the user will see the error.
						lastErr = err
						out.Write(sf.FormatProgress(utils.TruncateID(img.ID), fmt.Sprintf("Error pulling image (%s) from %s, endpoint: %s, %s", img.Tag, localName, ep, err), nil))
						continue
					}
					layers_downloaded = layers_downloaded || is_downloaded
					success = true
					break
				}
			}
			if !success {
				err := fmt.Errorf("Error pulling image (%s) from %s, %v", img.Tag, localName, lastErr)
				out.Write(sf.FormatProgress(utils.TruncateID(img.ID), err.Error(), nil))
				if parallel {
					errors <- err
					return
				}
			}
			out.Write(sf.FormatProgress(utils.TruncateID(img.ID), "Download complete", nil))

			if parallel {
				errors <- nil
			}
		}

		if parallel {
			go downloadImage(image)
		} else {
			downloadImage(image)
		}
	}
	if parallel {
		var lastError error
		for i := 0; i < len(repoData.ImgList); i++ {
			if err := <-errors; err != nil {
				lastError = err
			}
		}
		if lastError != nil {
			return lastError
		}

	}
	for tag, id := range tagsList {
		if askedTag != "" && id != imageId {
			continue
		}
		if err := s.Set(localName, tag, id, true); err != nil {
			return err
		}
	}

	requestedTag := localName
	if len(askedTag) > 0 {
		requestedTag = localName + ":" + askedTag
	}
	WriteStatus(requestedTag, out, sf, layers_downloaded)
	return nil
}

func (r *Session) GetRepositoryData(remote string) (*RepositoryData, error) {
	repositoryTarget := fmt.Sprintf("%srepositories/%s/images", r.indexEndpoint.VersionString(1), remote)

	log.Debugf("[registry] Calling GET %s", repositoryTarget)

	req, err := r.reqFactory.NewRequest("GET", repositoryTarget, nil)
	if err != nil {
		return nil, err
	}
	if r.authConfig != nil && len(r.authConfig.Username) > 0 {
		req.SetBasicAuth(r.authConfig.Username, r.authConfig.Password)
	}
	req.Header.Set("X-Docker-Token", "true")

	res, _, err := r.doRequest(req)
	if err != nil {
		return nil, err
	}
	defer res.Body.Close()
	if res.StatusCode == 401 {
		return nil, errLoginRequired
	}
	// TODO: Right now we're ignoring checksums in the response body.
	// In the future, we need to use them to check image validity.
	if res.StatusCode != 200 {
		return nil, utils.NewHTTPRequestError(fmt.Sprintf("HTTP code: %d", res.StatusCode), res)
	}

	var tokens []string
	if res.Header.Get("X-Docker-Token") != "" {
		tokens = res.Header["X-Docker-Token"]
	}

	var endpoints []string
	if res.Header.Get("X-Docker-Endpoints") != "" {
		endpoints, err = buildEndpointsList(res.Header["X-Docker-Endpoints"], r.indexEndpoint.VersionString(1))
		if err != nil {
			return nil, err
		}
	} else {
		// Assume the endpoint is on the same host
		endpoints = append(endpoints, fmt.Sprintf("%s://%s/v1/", r.indexEndpoint.URL.Scheme, req.URL.Host))
	}

	checksumsJSON, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return nil, err
	}
	remoteChecksums := []*ImgData{}
	if err := json.Unmarshal(checksumsJSON, &remoteChecksums); err != nil {
		return nil, err
	}

	// Forge a better object from the retrieved data
	imgsData := make(map[string]*ImgData)
	for _, elem := range remoteChecksums {
		imgsData[elem.ID] = elem
	}

	return &RepositoryData{
		ImgList:   imgsData,
		Endpoints: endpoints,
		Tokens:    tokens,
	}, nil
}

// NewRequest() creates a new *http.Request,
// applies all decorators in the HTTPRequestFactory on the request,
// then applies decorators provided by d on the request.
func (h *HTTPRequestFactory) NewRequest(method, urlStr string, body io.Reader, d ...HTTPRequestDecorator) (*http.Request, error) {
	req, err := http.NewRequest(method, urlStr, body)
	if err != nil {
		return nil, err
	}

	// By default, a nil factory should work.
	if h == nil {
		return req, nil
	}
	for _, dec := range h.decorators {
		req, err = dec.ChangeRequest(req)
		if err != nil {
			return nil, err
		}
	}
	for _, dec := range d {
		req, err = dec.ChangeRequest(req)
		if err != nil {
			return nil, err
		}
	}
	log.Debugf("%v -- HEADERS: %v", req.URL, req.Header)
	return req, err
}

func (r *Session) doRequest(req *http.Request) (*http.Response, *http.Client, error) {
	return doRequest(req, r.jar, r.timeout, r.indexEndpoint.secure)
}

func doRequest(req *http.Request, jar http.CookieJar, timeout TimeoutType, secure bool) (*http.Response, *http.Client, error) {
	var (
		pool  *x509.CertPool
		certs []*tls.Certificate
	)

	if secure && req.URL.Scheme == "https" {
		hasFile := func(files []os.FileInfo, name string) bool {
			for _, f := range files {
				if f.Name() == name {
					return true
				}
			}
			return false
		}

		hostDir := path.Join("/etc/docker/certs.d", req.URL.Host)
		log.Debugf("hostDir: %s", hostDir)
		fs, err := ioutil.ReadDir(hostDir)
		if err != nil && !os.IsNotExist(err) {
			return nil, nil, err
		}

		for _, f := range fs {
			if strings.HasSuffix(f.Name(), ".crt") {
				if pool == nil {
					pool = x509.NewCertPool()
				}
				log.Debugf("crt: %s", hostDir+"/"+f.Name())
				data, err := ioutil.ReadFile(path.Join(hostDir, f.Name()))
				if err != nil {
					return nil, nil, err
				}
				pool.AppendCertsFromPEM(data)
			}
			if strings.HasSuffix(f.Name(), ".cert") {
				certName := f.Name()
				keyName := certName[:len(certName)-5] + ".key"
				log.Debugf("cert: %s", hostDir+"/"+f.Name())
				if !hasFile(fs, keyName) {
					return nil, nil, fmt.Errorf("Missing key %s for certificate %s", keyName, certName)
				}
				cert, err := tls.LoadX509KeyPair(path.Join(hostDir, certName), path.Join(hostDir, keyName))
				if err != nil {
					return nil, nil, err
				}
				certs = append(certs, &cert)
			}
			if strings.HasSuffix(f.Name(), ".key") {
				keyName := f.Name()
				certName := keyName[:len(keyName)-4] + ".cert"
				log.Debugf("key: %s", hostDir+"/"+f.Name())
				if !hasFile(fs, certName) {
					return nil, nil, fmt.Errorf("Missing certificate %s for key %s", certName, keyName)
				}
			}
		}
	}

	if len(certs) == 0 {
		client := newClient(jar, pool, nil, timeout, secure)
		res, err := client.Do(req)
		if err != nil {
			return nil, nil, err
		}
		return res, client, nil
	}

	for i, cert := range certs {
		client := newClient(jar, pool, cert, timeout, secure)
		res, err := client.Do(req)
		// If this is the last cert, otherwise, continue to next cert if 403 or 5xx
		if i == len(certs)-1 || err == nil && res.StatusCode != 403 && res.StatusCode < 500 {
			return res, client, err
		}
	}

	return nil, nil, nil
}

func newClient(jar http.CookieJar, roots *x509.CertPool, cert *tls.Certificate, timeout TimeoutType, secure bool) *http.Client {
	tlsConfig := tls.Config{
		RootCAs: roots,
		// Avoid fallback to SSL protocols < TLS1.0
		MinVersion: tls.VersionTLS10,
	}

	if cert != nil {
		tlsConfig.Certificates = append(tlsConfig.Certificates, *cert)
	}

	if !secure {
		tlsConfig.InsecureSkipVerify = true
	}

	httpTransport := &http.Transport{
		DisableKeepAlives: true,
		Proxy:             http.ProxyFromEnvironment,
		TLSClientConfig:   &tlsConfig,
	}

	switch timeout {
	case ConnectTimeout:
		httpTransport.Dial = func(proto string, addr string) (net.Conn, error) {
			// Set the connect timeout to 5 seconds
			conn, err := net.DialTimeout(proto, addr, 5*time.Second)
			if err != nil {
				return nil, err
			}
			// Set the recv timeout to 10 seconds
			conn.SetDeadline(time.Now().Add(10 * time.Second))
			return conn, nil
		}
	case ReceiveTimeout:
		httpTransport.Dial = func(proto string, addr string) (net.Conn, error) {
			conn, err := net.Dial(proto, addr)
			if err != nil {
				return nil, err
			}
			conn = utils.NewTimeoutConn(conn, 1*time.Minute)
			return conn, nil
		}
	}

	return &http.Client{
		Transport:     httpTransport,
		CheckRedirect: AddRequiredHeadersToRedirectedRequests,
		Jar:           jar,
	}
}

func (s *TagStore) pullImage(r *registry.Session, out io.Writer, imgID, endpoint string, token []string, sf *utils.StreamFormatter) (bool, error) {
	history, err := r.GetRemoteHistory(imgID, endpoint, token)
	if err != nil {
		return false, err
	}
	out.Write(sf.FormatProgress(utils.TruncateID(imgID), "Pulling dependent layers", nil))
	// FIXME: Try to stream the images?
	// FIXME: Launch the getRemoteImage() in goroutines

	layers_downloaded := false
	for i := len(history) - 1; i >= 0; i-- {
		id := history[i]

		// ensure no two downloads of the same layer happen at the same time
		if c, err := s.poolAdd("pull", "layer:"+id); err != nil {
			log.Debugf("Image (id: %s) pull is already running, skipping: %v", id, err)
			<-c
		}
		defer s.poolRemove("pull", "layer:"+id)

		if !s.graph.Exists(id) {
			out.Write(sf.FormatProgress(utils.TruncateID(id), "Pulling metadata", nil))
			var (
				imgJSON []byte
				imgSize int
				err     error
				img     *image.Image
			)
			retries := 5
			for j := 1; j <= retries; j++ {
				imgJSON, imgSize, err = r.GetRemoteImageJSON(id, endpoint, token)
				if err != nil && j == retries {
					out.Write(sf.FormatProgress(utils.TruncateID(id), "Error pulling dependent layers", nil))
					return layers_downloaded, err
				} else if err != nil {
					time.Sleep(time.Duration(j) * 500 * time.Millisecond)
					continue
				}
				img, err = image.NewImgJSON(imgJSON)
				layers_downloaded = true
				if err != nil && j == retries {
					out.Write(sf.FormatProgress(utils.TruncateID(id), "Error pulling dependent layers", nil))
					return layers_downloaded, fmt.Errorf("Failed to parse json: %s", err)
				} else if err != nil {
					time.Sleep(time.Duration(j) * 500 * time.Millisecond)
					continue
				} else {
					break
				}
			}

			for j := 1; j <= retries; j++ {
				// Get the layer
				status := "Pulling fs layer"
				if j > 1 {
					status = fmt.Sprintf("Pulling fs layer [retries: %d]", j)
				}
				out.Write(sf.FormatProgress(utils.TruncateID(id), status, nil))
				layer, err := r.GetRemoteImageLayer(img.ID, endpoint, token, int64(imgSize))
				if uerr, ok := err.(*url.Error); ok {
					err = uerr.Err
				}
				if terr, ok := err.(net.Error); ok && terr.Timeout() && j < retries {
					time.Sleep(time.Duration(j) * 500 * time.Millisecond)
					continue
				} else if err != nil {
					out.Write(sf.FormatProgress(utils.TruncateID(id), "Error pulling dependent layers", nil))
					return layers_downloaded, err
				}
				layers_downloaded = true
				defer layer.Close()

				err = s.graph.Register(img, imgJSON,
					utils.ProgressReader(layer, imgSize, out, sf, false, utils.TruncateID(id), "Downloading"))
				if terr, ok := err.(net.Error); ok && terr.Timeout() && j < retries {
					time.Sleep(time.Duration(j) * 500 * time.Millisecond)
					continue
				} else if err != nil {
					out.Write(sf.FormatProgress(utils.TruncateID(id), "Error downloading dependent layers", nil))
					return layers_downloaded, err
				} else {
					break
				}
			}
		}
		out.Write(sf.FormatProgress(utils.TruncateID(id), "Download complete", nil))
	}
	return layers_downloaded, nil
}

// Retrieve the history of a given image from the Registry.
// Return a list of the parent's json (requested image included)
func (r *Session) GetRemoteHistory(imgID, registry string, token []string) ([]string, error) {
	req, err := r.reqFactory.NewRequest("GET", registry+"images/"+imgID+"/ancestry", nil)
	if err != nil {
		return nil, err
	}
	setTokenAuth(req, token)
	res, _, err := r.doRequest(req)
	if err != nil {
		return nil, err
	}
	defer res.Body.Close()
	if res.StatusCode != 200 {
		if res.StatusCode == 401 {
			return nil, errLoginRequired
		}
		return nil, utils.NewHTTPRequestError(fmt.Sprintf("Server error: %d trying to fetch remote history for %s", res.StatusCode, imgID), res)
	}

	jsonString, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return nil, fmt.Errorf("Error while reading the http response: %s", err)
	}

	log.Debugf("Ancestry: %s", jsonString)
	history := new([]string)
	if err := json.Unmarshal(jsonString, history); err != nil {
		return nil, err
	}
	return *history, nil
}

// Retrieve an image from the Registry.
func (r *Session) GetRemoteImageJSON(imgID, registry string, token []string) ([]byte, int, error) {
	// Get the JSON
	req, err := r.reqFactory.NewRequest("GET", registry+"images/"+imgID+"/json", nil)
	if err != nil {
		return nil, -1, fmt.Errorf("Failed to download json: %s", err)
	}
	setTokenAuth(req, token)
	res, _, err := r.doRequest(req)
	if err != nil {
		return nil, -1, fmt.Errorf("Failed to download json: %s", err)
	}
	defer res.Body.Close()
	if res.StatusCode != 200 {
		return nil, -1, utils.NewHTTPRequestError(fmt.Sprintf("HTTP code %d", res.StatusCode), res)
	}
	// if the size header is not present, then set it to '-1'
	imageSize := -1
	if hdr := res.Header.Get("X-Docker-Size"); hdr != "" {
		imageSize, err = strconv.Atoi(hdr)
		if err != nil {
			return nil, -1, err
		}
	}

	jsonString, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return nil, -1, fmt.Errorf("Failed to parse downloaded json: %s (%s)", err, jsonString)
	}
	return jsonString, imageSize, nil
}

func (r *Session) GetRemoteImageLayer(imgID, registry string, token []string, imgSize int64) (io.ReadCloser, error) {
	var (
		retries    = 5
		statusCode = 0
		client     *http.Client
		res        *http.Response
		imageURL   = fmt.Sprintf("%simages/%s/layer", registry, imgID)
	)

	req, err := r.reqFactory.NewRequest("GET", imageURL, nil)
	if err != nil {
		return nil, fmt.Errorf("Error while getting from the server: %s\n", err)
	}
	setTokenAuth(req, token)
	for i := 1; i <= retries; i++ {
		statusCode = 0
		res, client, err = r.doRequest(req)
		if err != nil {
			log.Debugf("Error contacting registry: %s", err)
			if res != nil {
				if res.Body != nil {
					res.Body.Close()
				}
				statusCode = res.StatusCode
			}
			if i == retries {
				return nil, fmt.Errorf("Server error: Status %d while fetching image layer (%s)",
					statusCode, imgID)
			}
			time.Sleep(time.Duration(i) * 5 * time.Second)
			continue
		}
		break
	}

	if res.StatusCode != 200 {
		res.Body.Close()
		return nil, fmt.Errorf("Server error: Status %d while fetching image layer (%s)",
			res.StatusCode, imgID)
	}

	if res.Header.Get("Accept-Ranges") == "bytes" && imgSize > 0 {
		log.Debugf("server supports resume")
		return httputils.ResumableRequestReaderWithInitialResponse(client, req, 5, imgSize, res), nil
	}
	log.Debugf("server doesn't support resume")
	return res.Body, nil
}
```

下面重点分析`SetReadDeadline`原理：

docker使用`SetReadDeadline`入口：

```go
...
if len(certs) == 0 {
    client := newClient(jar, pool, nil, timeout, secure)
    res, err := client.Do(req)
    if err != nil {
        return nil, nil, err
    }
    return res, client, nil
}
...

func newClient(jar http.CookieJar, roots *x509.CertPool, cert *tls.Certificate, timeout TimeoutType, secure bool) *http.Client {
	tlsConfig := tls.Config{
		RootCAs: roots,
		// Avoid fallback to SSL protocols < TLS1.0
		MinVersion: tls.VersionTLS10,
	}

	if cert != nil {
		tlsConfig.Certificates = append(tlsConfig.Certificates, *cert)
	}

	if !secure {
		tlsConfig.InsecureSkipVerify = true
	}

	httpTransport := &http.Transport{
		DisableKeepAlives: true,
		Proxy:             http.ProxyFromEnvironment,
		TLSClientConfig:   &tlsConfig,
	}

	switch timeout {
	case ConnectTimeout:
		httpTransport.Dial = func(proto string, addr string) (net.Conn, error) {
			// Set the connect timeout to 5 seconds
			conn, err := net.DialTimeout(proto, addr, 5*time.Second)
			if err != nil {
				return nil, err
			}
			// Set the recv timeout to 10 seconds
			conn.SetDeadline(time.Now().Add(10 * time.Second))
			return conn, nil
		}
	case ReceiveTimeout:
		httpTransport.Dial = func(proto string, addr string) (net.Conn, error) {
			conn, err := net.Dial(proto, addr)
			if err != nil {
				return nil, err
			}
			conn = utils.NewTimeoutConn(conn, 1*time.Minute)
			return conn, nil
		}
	}

	return &http.Client{
		Transport:     httpTransport,
		CheckRedirect: AddRequiredHeadersToRedirectedRequests,
		Jar:           jar,
	}
}

func NewTimeoutConn(conn net.Conn, timeout time.Duration) net.Conn {
	return &TimeoutConn{conn, timeout}
}

// A net.Conn that sets a deadline for every Read or Write operation
type TimeoutConn struct {
	net.Conn
	timeout time.Duration
}

func (c *TimeoutConn) Read(b []byte) (int, error) {
	if c.timeout > 0 {
		err := c.Conn.SetReadDeadline(time.Now().Add(c.timeout))
		if err != nil {
			return 0, err
		}
	}
	return c.Conn.Read(b)
}
```

分析原理：

```go
// Conn is a generic stream-oriented network connection.
//
// Multiple goroutines may invoke methods on a Conn simultaneously.
type Conn interface {
	// Read reads data from the connection.
	// Read can be made to time out and return a Error with Timeout() == true
	// after a fixed time limit; see SetDeadline and SetReadDeadline.
	Read(b []byte) (n int, err error)

	// Write writes data to the connection.
	// Write can be made to time out and return a Error with Timeout() == true
	// after a fixed time limit; see SetDeadline and SetWriteDeadline.
	Write(b []byte) (n int, err error)

	// Close closes the connection.
	// Any blocked Read or Write operations will be unblocked and return errors.
	Close() error

	// LocalAddr returns the local network address.
	LocalAddr() Addr

	// RemoteAddr returns the remote network address.
	RemoteAddr() Addr

	// SetDeadline sets the read and write deadlines associated
	// with the connection. It is equivalent to calling both
	// SetReadDeadline and SetWriteDeadline.
	//
	// A deadline is an absolute time after which I/O operations
	// fail with a timeout (see type Error) instead of
	// blocking. The deadline applies to all future I/O, not just
	// the immediately following call to Read or Write.
	//
	// An idle timeout can be implemented by repeatedly extending
	// the deadline after successful Read or Write calls.
	//
	// A zero value for t means I/O operations will not time out.
	SetDeadline(t time.Time) error

	// SetReadDeadline sets the deadline for future Read calls.
	// A zero value for t means Read will not time out.
	SetReadDeadline(t time.Time) error

	// SetWriteDeadline sets the deadline for future Write calls.
	// Even if write times out, it may return n > 0, indicating that
	// some of the data was successfully written.
	// A zero value for t means Write will not time out.
	SetWriteDeadline(t time.Time) error
}


```