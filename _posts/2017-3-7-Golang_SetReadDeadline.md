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

func (r *Session) GetRemoteTags(registries []string, repository string, token []string) (map[string]string, error) {
	if strings.Count(repository, "/") == 0 {
		// This will be removed once the Registry supports auto-resolution on
		// the "library" namespace
		repository = "library/" + repository
	}
	for _, host := range registries {
		endpoint := fmt.Sprintf("%srepositories/%s/tags", host, repository)
		req, err := r.reqFactory.NewRequest("GET", endpoint, nil)

		if err != nil {
			return nil, err
		}
		setTokenAuth(req, token)
		res, _, err := r.doRequest(req)
		if err != nil {
			return nil, err
		}

		log.Debugf("Got status code %d from %s", res.StatusCode, endpoint)
		defer res.Body.Close()

		if res.StatusCode != 200 && res.StatusCode != 404 {
			continue
		} else if res.StatusCode == 404 {
			return nil, fmt.Errorf("Repository not found")
		}

		result := make(map[string]string)
		rawJSON, err := ioutil.ReadAll(res.Body)
		if err != nil {
			return nil, err
		}
		if err := json.Unmarshal(rawJSON, &result); err != nil {
			return nil, err
		}
		return result, nil
	}
	return nil, fmt.Errorf("Could not reach any registry endpoint")
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
// Register imports a pre-existing image into the graph.
func (graph *Graph) Register(img *image.Image, jsonData []byte, layerData archive.ArchiveReader) (err error) {
	defer func() {
		// If any error occurs, remove the new dir from the driver.
		// Don't check for errors since the dir might not have been created.
		// FIXME: this leaves a possible race condition.
		if err != nil {
			graph.driver.Remove(img.ID)
		}
	}()
	if err := utils.ValidateID(img.ID); err != nil {
		return err
	}
	// (This is a convenience to save time. Race conditions are taken care of by os.Rename)
	if graph.Exists(img.ID) {
		return fmt.Errorf("Image %s already exists", img.ID)
	}

	// Ensure that the image root does not exist on the filesystem
	// when it is not registered in the graph.
	// This is common when you switch from one graph driver to another
	if err := os.RemoveAll(graph.ImageRoot(img.ID)); err != nil && !os.IsNotExist(err) {
		return err
	}

	// If the driver has this ID but the graph doesn't, remove it from the driver to start fresh.
	// (the graph is the source of truth).
	// Ignore errors, since we don't know if the driver correctly returns ErrNotExist.
	// (FIXME: make that mandatory for drivers).
	graph.driver.Remove(img.ID)

	tmp, err := graph.Mktemp("")
	defer os.RemoveAll(tmp)
	if err != nil {
		return fmt.Errorf("Mktemp failed: %s", err)
	}

	// Create root filesystem in the driver
	if err := graph.driver.Create(img.ID, img.Parent); err != nil {
		return fmt.Errorf("Driver %s failed to create image rootfs %s: %s", graph.driver, img.ID, err)
	}
	// Apply the diff/layer
	img.SetGraph(graph)
	if err := image.StoreImage(img, jsonData, layerData, tmp); err != nil {
		return err
	}
	// Commit
	if err := os.Rename(tmp, graph.ImageRoot(img.ID)); err != nil {
		return err
	}
	graph.idIndex.Add(img.ID)
	return nil
}
func WriteStatus(requestedTag string, out io.Writer, sf *utils.StreamFormatter, layers_downloaded bool) {
	if layers_downloaded {
		out.Write(sf.FormatStatus("", "Status: Downloaded newer image for %s", requestedTag))
	} else {
		out.Write(sf.FormatStatus("", "Status: Image is up to date for %s", requestedTag))
	}
}
```

如下是docker pull流程图：

![](/public/img/golang_client_flow/docker_pull_flow.png)

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

`res, err := client.Do(req)`请求分析：

```go
// A Client is an HTTP client. Its zero value (DefaultClient) is a
// usable client that uses DefaultTransport.
//
// The Client's Transport typically has internal state (cached TCP
// connections), so Clients should be reused instead of created as
// needed. Clients are safe for concurrent use by multiple goroutines.
//
// A Client is higher-level than a RoundTripper (such as Transport)
// and additionally handles HTTP details such as cookies and
// redirects.
type Client struct {
	// Transport specifies the mechanism by which individual
	// HTTP requests are made.
	// If nil, DefaultTransport is used.
	Transport RoundTripper

	// CheckRedirect specifies the policy for handling redirects.
	// If CheckRedirect is not nil, the client calls it before
	// following an HTTP redirect. The arguments req and via are
	// the upcoming request and the requests made already, oldest
	// first. If CheckRedirect returns an error, the Client's Get
	// method returns both the previous Response (with its Body
	// closed) and CheckRedirect's error (wrapped in a url.Error)
	// instead of issuing the Request req.
	// As a special case, if CheckRedirect returns ErrUseLastResponse,
	// then the most recent response is returned with its body
	// unclosed, along with a nil error.
	//
	// If CheckRedirect is nil, the Client uses its default policy,
	// which is to stop after 10 consecutive requests.
	CheckRedirect func(req *Request, via []*Request) error

	// Jar specifies the cookie jar.
	// If Jar is nil, cookies are not sent in requests and ignored
	// in responses.
	Jar CookieJar

	// Timeout specifies a time limit for requests made by this
	// Client. The timeout includes connection time, any
	// redirects, and reading the response body. The timer remains
	// running after Get, Head, Post, or Do return and will
	// interrupt reading of the Response.Body.
	//
	// A Timeout of zero means no timeout.
	//
	// The Client cancels requests to the underlying Transport
	// using the Request.Cancel mechanism. Requests passed
	// to Client.Do may still set Request.Cancel; both will
	// cancel the request.
	//
	// For compatibility, the Client will also use the deprecated
	// CancelRequest method on Transport if found. New
	// RoundTripper implementations should use Request.Cancel
	// instead of implementing CancelRequest.
	Timeout time.Duration
}

// DefaultMaxIdleConnsPerHost is the default value of Transport's
// MaxIdleConnsPerHost.
const DefaultMaxIdleConnsPerHost = 2

// Transport is an implementation of RoundTripper that supports HTTP,
// HTTPS, and HTTP proxies (for either HTTP or HTTPS with CONNECT).
//
// By default, Transport caches connections for future re-use.
// This may leave many open connections when accessing many hosts.
// This behavior can be managed using Transport's CloseIdleConnections method
// and the MaxIdleConnsPerHost and DisableKeepAlives fields.
//
// Transports should be reused instead of created as needed.
// Transports are safe for concurrent use by multiple goroutines.
//
// A Transport is a low-level primitive for making HTTP and HTTPS requests.
// For high-level functionality, such as cookies and redirects, see Client.
//
// Transport uses HTTP/1.1 for HTTP URLs and either HTTP/1.1 or HTTP/2
// for HTTPS URLs, depending on whether the server supports HTTP/2.
// See the package docs for more about HTTP/2.
type Transport struct {
	idleMu     sync.Mutex
	wantIdle   bool                                // user has requested to close all idle conns
	idleConn   map[connectMethodKey][]*persistConn // most recently used at end
	idleConnCh map[connectMethodKey]chan *persistConn
	idleLRU    connLRU

	reqMu       sync.Mutex
	reqCanceler map[*Request]func()

	altMu    sync.RWMutex
	altProto map[string]RoundTripper // nil or map of URI scheme => RoundTripper

	// Proxy specifies a function to return a proxy for a given
	// Request. If the function returns a non-nil error, the
	// request is aborted with the provided error.
	// If Proxy is nil or returns a nil *URL, no proxy is used.
	Proxy func(*Request) (*url.URL, error)

	// DialContext specifies the dial function for creating unencrypted TCP connections.
	// If DialContext is nil (and the deprecated Dial below is also nil),
	// then the transport dials using package net.
	DialContext func(ctx context.Context, network, addr string) (net.Conn, error)

	// Dial specifies the dial function for creating unencrypted TCP connections.
	//
	// Deprecated: Use DialContext instead, which allows the transport
	// to cancel dials as soon as they are no longer needed.
	// If both are set, DialContext takes priority.
	Dial func(network, addr string) (net.Conn, error)

	// DialTLS specifies an optional dial function for creating
	// TLS connections for non-proxied HTTPS requests.
	//
	// If DialTLS is nil, Dial and TLSClientConfig are used.
	//
	// If DialTLS is set, the Dial hook is not used for HTTPS
	// requests and the TLSClientConfig and TLSHandshakeTimeout
	// are ignored. The returned net.Conn is assumed to already be
	// past the TLS handshake.
	DialTLS func(network, addr string) (net.Conn, error)

	// TLSClientConfig specifies the TLS configuration to use with
	// tls.Client. If nil, the default configuration is used.
	TLSClientConfig *tls.Config

	// TLSHandshakeTimeout specifies the maximum amount of time waiting to
	// wait for a TLS handshake. Zero means no timeout.
	TLSHandshakeTimeout time.Duration

	// DisableKeepAlives, if true, prevents re-use of TCP connections
	// between different HTTP requests.
	DisableKeepAlives bool

	// DisableCompression, if true, prevents the Transport from
	// requesting compression with an "Accept-Encoding: gzip"
	// request header when the Request contains no existing
	// Accept-Encoding value. If the Transport requests gzip on
	// its own and gets a gzipped response, it's transparently
	// decoded in the Response.Body. However, if the user
	// explicitly requested gzip it is not automatically
	// uncompressed.
	DisableCompression bool

	// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.
	MaxIdleConns int

	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
	// DefaultMaxIdleConnsPerHost is used.
	MaxIdleConnsPerHost int

	// IdleConnTimeout is the maximum amount of time an idle
	// (keep-alive) connection will remain idle before closing
	// itself.
	// Zero means no limit.
	IdleConnTimeout time.Duration

	// ResponseHeaderTimeout, if non-zero, specifies the amount of
	// time to wait for a server's response headers after fully
	// writing the request (including its body, if any). This
	// time does not include the time to read the response body.
	ResponseHeaderTimeout time.Duration

	// ExpectContinueTimeout, if non-zero, specifies the amount of
	// time to wait for a server's first response headers after fully
	// writing the request headers if the request has an
	// "Expect: 100-continue" header. Zero means no timeout.
	// This time does not include the time to send the request header.
	ExpectContinueTimeout time.Duration

	// TLSNextProto specifies how the Transport switches to an
	// alternate protocol (such as HTTP/2) after a TLS NPN/ALPN
	// protocol negotiation. If Transport dials an TLS connection
	// with a non-empty protocol name and TLSNextProto contains a
	// map entry for that key (such as "h2"), then the func is
	// called with the request's authority (such as "example.com"
	// or "example.com:1234") and the TLS connection. The function
	// must return a RoundTripper that then handles the request.
	// If TLSNextProto is nil, HTTP/2 support is enabled automatically.
	TLSNextProto map[string]func(authority string, c *tls.Conn) RoundTripper

	// MaxResponseHeaderBytes specifies a limit on how many
	// response bytes are allowed in the server's response
	// header.
	//
	// Zero means to use a default limit.
	MaxResponseHeaderBytes int64

	// nextProtoOnce guards initialization of TLSNextProto and
	// h2transport (via onceSetNextProtoDefaults)
	nextProtoOnce sync.Once
	h2transport   *http2Transport // non-nil if http2 wired up

	// TODO: tunable on max per-host TCP dials in flight (Issue 13957)
}

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

// Do sends an HTTP request and returns an HTTP response, following
// policy (e.g. redirects, cookies, auth) as configured on the client.
//
// An error is returned if caused by client policy (such as
// CheckRedirect), or if there was an HTTP protocol error.
// A non-2xx response doesn't cause an error.
//
// When err is nil, resp always contains a non-nil resp.Body.
//
// Callers should close resp.Body when done reading from it. If
// resp.Body is not closed, the Client's underlying RoundTripper
// (typically Transport) may not be able to re-use a persistent TCP
// connection to the server for a subsequent "keep-alive" request.
//
// The request Body, if non-nil, will be closed by the underlying
// Transport, even on errors.
//
// Generally Get, Post, or PostForm will be used instead of Do.
func (c *Client) Do(req *Request) (resp *Response, err error) {
	if req.Method == "GET" || req.Method == "HEAD" {
		return c.doFollowingRedirects(req, shouldRedirectGet)
	}
	if req.Method == "POST" || req.Method == "PUT" {
		return c.doFollowingRedirects(req, shouldRedirectPost)
	}
	return c.send(req)
}

func (c *Client) doFollowingRedirects(ireq *Request, shouldRedirect func(int) bool) (resp *Response, err error) {
	var base *url.URL
	redirectChecker := c.CheckRedirect
	if redirectChecker == nil {
		redirectChecker = defaultCheckRedirect
	}
	var via []*Request

	if ireq.URL == nil {
		ireq.closeBody()
		return nil, errors.New("http: nil Request.URL")
	}

	var reqmu sync.Mutex // guards req
	req := ireq

	var timer *time.Timer
	if c.Timeout > 0 {
		type canceler interface {
			CancelRequest(*Request)
		}
		tr, ok := c.transport().(canceler)
		if !ok {
			return nil, fmt.Errorf("net/http: Client Transport of type %T doesn't support CancelRequest; Timeout not supported", c.transport())
		}
		timer = time.AfterFunc(c.Timeout, func() {
			reqmu.Lock()
			defer reqmu.Unlock()
			tr.CancelRequest(req)
		})
	}

	urlStr := "" // next relative or absolute URL to fetch (after first request)
	redirectFailed := false
	for redirect := 0; ; redirect++ {
		if redirect != 0 {
			nreq := new(Request)
			nreq.Method = ireq.Method
			if ireq.Method == "POST" || ireq.Method == "PUT" {
				nreq.Method = "GET"
			}
			nreq.Header = make(Header)
			nreq.URL, err = base.Parse(urlStr)
			if err != nil {
				break
			}
			if len(via) > 0 {
				// Add the Referer header.
				lastReq := via[len(via)-1]
				if lastReq.URL.Scheme != "https" {
					nreq.Header.Set("Referer", lastReq.URL.String())
				}

				err = redirectChecker(nreq, via)
				if err != nil {
					redirectFailed = true
					break
				}
			}
			reqmu.Lock()
			req = nreq
			reqmu.Unlock()
		}

		urlStr = req.URL.String()
		if resp, err = c.send(req); err != nil {
			break
		}

		if shouldRedirect(resp.StatusCode) {
			// Read the body if small so underlying TCP connection will be re-used.
			// No need to check for errors: if it fails, Transport won't reuse it anyway.
			const maxBodySlurpSize = 2 << 10
			if resp.ContentLength == -1 || resp.ContentLength <= maxBodySlurpSize {
				io.CopyN(ioutil.Discard, resp.Body, maxBodySlurpSize)
			}
			resp.Body.Close()
			if urlStr = resp.Header.Get("Location"); urlStr == "" {
				err = errors.New(fmt.Sprintf("%d response missing Location header", resp.StatusCode))
				break
			}
			base = req.URL
			via = append(via, req)
			continue
		}
		if timer != nil {
			resp.Body = &cancelTimerBody{timer, resp.Body}
		}
		return resp, nil
	}

	method := ireq.Method
	urlErr := &url.Error{
		Op:  method[0:1] + strings.ToLower(method[1:]),
		URL: urlStr,
		Err: err,
	}

	if redirectFailed {
		// Special case for Go 1 compatibility: return both the response
		// and an error if the CheckRedirect function failed.
		// See http://golang.org/issue/3795
		return resp, urlErr
	}

	if resp != nil {
		resp.Body.Close()
	}
	return nil, urlErr
}

// True if the specified HTTP status code is one for which the Get utility should
// automatically redirect.
func shouldRedirectGet(statusCode int) bool {
	switch statusCode {
	case StatusMovedPermanently, StatusFound, StatusSeeOther, StatusTemporaryRedirect:
		return true
	}
	return false
}

func (c *Client) send(req *Request) (*Response, error) {
	if c.Jar != nil {
		for _, cookie := range c.Jar.Cookies(req.URL) {
			req.AddCookie(cookie)
		}
	}
	resp, err := send(req, c.transport())
	if err != nil {
		return nil, err
	}
	if c.Jar != nil {
		if rc := resp.Cookies(); len(rc) > 0 {
			c.Jar.SetCookies(req.URL, rc)
		}
	}
	return resp, err
}

func (c *Client) transport() RoundTripper {
	if c.Transport != nil {
		return c.Transport
	}
	return DefaultTransport
}

// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	Dial: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}).Dial,
	TLSHandshakeTimeout: 10 * time.Second,
}

// send issues an HTTP request.
// Caller should close resp.Body when done reading from it.
func send(req *Request, t RoundTripper) (resp *Response, err error) {
	if t == nil {
		req.closeBody()
		return nil, errors.New("http: no Client.Transport or DefaultTransport")
	}

	if req.URL == nil {
		req.closeBody()
		return nil, errors.New("http: nil Request.URL")
	}

	if req.RequestURI != "" {
		req.closeBody()
		return nil, errors.New("http: Request.RequestURI can't be set in client requests.")
	}

	// Most the callers of send (Get, Post, et al) don't need
	// Headers, leaving it uninitialized.  We guarantee to the
	// Transport that this has been initialized, though.
	if req.Header == nil {
		req.Header = make(Header)
	}

	if u := req.URL.User; u != nil {
		username := u.Username()
		password, _ := u.Password()
		req.Header.Set("Authorization", "Basic "+basicAuth(username, password))
	}
	resp, err = t.RoundTrip(req)
	if err != nil {
		if resp != nil {
			log.Printf("RoundTripper returned a response & error; ignoring response")
		}
		return nil, err
	}
	return resp, nil
}

// RoundTrip implements the RoundTripper interface.
//
// For higher-level HTTP client support (such as handling of cookies
// and redirects), see Get, Post, and the Client type.
func (t *Transport) RoundTrip(req *Request) (resp *Response, err error) {
	if req.URL == nil {
		req.closeBody()
		return nil, errors.New("http: nil Request.URL")
	}
	if req.Header == nil {
		req.closeBody()
		return nil, errors.New("http: nil Request.Header")
	}
	if req.URL.Scheme != "http" && req.URL.Scheme != "https" {
		t.altMu.RLock()
		var rt RoundTripper
		if t.altProto != nil {
			rt = t.altProto[req.URL.Scheme]
		}
		t.altMu.RUnlock()
		if rt == nil {
			req.closeBody()
			return nil, &badStringError{"unsupported protocol scheme", req.URL.Scheme}
		}
		return rt.RoundTrip(req)
	}
	if req.URL.Host == "" {
		req.closeBody()
		return nil, errors.New("http: no Host in request URL")
	}
	treq := &transportRequest{Request: req}
	cm, err := t.connectMethodForRequest(treq)
	if err != nil {
		req.closeBody()
		return nil, err
	}

	// Get the cached or newly-created connection to either the
	// host (for http or https), the http proxy, or the http proxy
	// pre-CONNECTed to https server.  In any case, we'll be ready
	// to send it requests.
	pconn, err := t.getConn(req, cm)
	if err != nil {
		t.setReqCanceler(req, nil)
		req.closeBody()
		return nil, err
	}

	return pconn.roundTrip(treq)
}

// getConn dials and creates a new persistConn to the target as
// specified in the connectMethod.  This includes doing a proxy CONNECT
// and/or setting up TLS.  If this doesn't return an error, the persistConn
// is ready to write requests to.
func (t *Transport) getConn(req *Request, cm connectMethod) (*persistConn, error) {
	if pc := t.getIdleConn(cm); pc != nil {
		return pc, nil
	}

	type dialRes struct {
		pc  *persistConn
		err error
	}
	dialc := make(chan dialRes)

	handlePendingDial := func() {
		if v := <-dialc; v.err == nil {
			t.putIdleConn(v.pc)
		}
	}

	cancelc := make(chan struct{})
	t.setReqCanceler(req, func() { close(cancelc) })

	go func() {
		pc, err := t.dialConn(cm)
		dialc <- dialRes{pc, err}
	}()

	idleConnCh := t.getIdleConnCh(cm)
	select {
	case v := <-dialc:
		// Our dial finished.
		return v.pc, v.err
	case pc := <-idleConnCh:
		// Another request finished first and its net.Conn
		// became available before our dial. Or somebody
		// else's dial that they didn't use.
		// But our dial is still going, so give it away
		// when it finishes:
		go handlePendingDial()
		return pc, nil
	case <-cancelc:
		go handlePendingDial()
		return nil, errors.New("net/http: request canceled while waiting for connection")
	}
}

// persistConn wraps a connection, usually a persistent one
// (but may be used for non-keep-alive requests as well)
type persistConn struct {
	t        *Transport
	cacheKey connectMethodKey
	conn     net.Conn
	tlsState *tls.ConnectionState
	br       *bufio.Reader       // from conn
	sawEOF   bool                // whether we've seen EOF from conn; owned by readLoop
	bw       *bufio.Writer       // to conn
	reqch    chan requestAndChan // written by roundTrip; read by readLoop
	writech  chan writeRequest   // written by roundTrip; read by writeLoop
	closech  chan struct{}       // closed when conn closed
	isProxy  bool
	// writeErrCh passes the request write error (usually nil)
	// from the writeLoop goroutine to the readLoop which passes
	// it off to the res.Body reader, which then uses it to decide
	// whether or not a connection can be reused. Issue 7569.
	writeErrCh chan error

	lk                   sync.Mutex // guards following fields
	numExpectedResponses int
	closed               bool // whether conn has been closed
	broken               bool // an error has happened on this connection; marked broken so it's not reused.
	// mutateHeaderFunc is an optional func to modify extra
	// headers on each outbound request before it's written. (the
	// original Request given to RoundTrip is not modified)
	mutateHeaderFunc func(Header)
}

func (t *Transport) getIdleConn(cm connectMethod) (pconn *persistConn) {
	key := cm.key()
	t.idleMu.Lock()
	defer t.idleMu.Unlock()
	if t.idleConn == nil {
		return nil
	}
	for {
		pconns, ok := t.idleConn[key]
		if !ok {
			return nil
		}
		if len(pconns) == 1 {
			pconn = pconns[0]
			delete(t.idleConn, key)
		} else {
			// 2 or more cached connections; pop last
			// TODO: queue?
			pconn = pconns[len(pconns)-1]
			t.idleConn[key] = pconns[:len(pconns)-1]
		}
		if !pconn.isBroken() {
			return
		}
	}
}

func (t *Transport) dialConn(cm connectMethod) (*persistConn, error) {
	conn, err := t.dial("tcp", cm.addr())
	if err != nil {
		if cm.proxyURL != nil {
			err = fmt.Errorf("http: error connecting to proxy %s: %v", cm.proxyURL, err)
		}
		return nil, err
	}

	pa := cm.proxyAuth()

	pconn := &persistConn{
		t:          t,
		cacheKey:   cm.key(),
		conn:       conn,
		reqch:      make(chan requestAndChan, 1),
		writech:    make(chan writeRequest, 1),
		closech:    make(chan struct{}),
		writeErrCh: make(chan error, 1),
	}

	switch {
	case cm.proxyURL == nil:
		// Do nothing.
	case cm.targetScheme == "http":
		pconn.isProxy = true
		if pa != "" {
			pconn.mutateHeaderFunc = func(h Header) {
				h.Set("Proxy-Authorization", pa)
			}
		}
	case cm.targetScheme == "https":
		connectReq := &Request{
			Method: "CONNECT",
			URL:    &url.URL{Opaque: cm.targetAddr},
			Host:   cm.targetAddr,
			Header: make(Header),
		}
		if pa != "" {
			connectReq.Header.Set("Proxy-Authorization", pa)
		}
		connectReq.Write(conn)

		// Read response.
		// Okay to use and discard buffered reader here, because
		// TLS server will not speak until spoken to.
		br := bufio.NewReader(conn)
		resp, err := ReadResponse(br, connectReq)
		if err != nil {
			conn.Close()
			return nil, err
		}
		if resp.StatusCode != 200 {
			f := strings.SplitN(resp.Status, " ", 2)
			conn.Close()
			return nil, errors.New(f[1])
		}
	}

	if cm.targetScheme == "https" {
		// Initiate TLS and check remote host name against certificate.
		cfg := t.TLSClientConfig
		if cfg == nil || cfg.ServerName == "" {
			host := cm.tlsHost()
			if cfg == nil {
				cfg = &tls.Config{ServerName: host}
			} else {
				clone := *cfg // shallow clone
				clone.ServerName = host
				cfg = &clone
			}
		}
		plainConn := conn
		tlsConn := tls.Client(plainConn, cfg)
		errc := make(chan error, 2)
		var timer *time.Timer // for canceling TLS handshake
		if d := t.TLSHandshakeTimeout; d != 0 {
			timer = time.AfterFunc(d, func() {
				errc <- tlsHandshakeTimeoutError{}
			})
		}
		go func() {
			err := tlsConn.Handshake()
			if timer != nil {
				timer.Stop()
			}
			errc <- err
		}()
		if err := <-errc; err != nil {
			plainConn.Close()
			return nil, err
		}
		if !cfg.InsecureSkipVerify {
			if err := tlsConn.VerifyHostname(cfg.ServerName); err != nil {
				plainConn.Close()
				return nil, err
			}
		}
		cs := tlsConn.ConnectionState()
		pconn.tlsState = &cs
		pconn.conn = tlsConn
	}

	pconn.br = bufio.NewReader(noteEOFReader{pconn.conn, &pconn.sawEOF})
	pconn.bw = bufio.NewWriter(pconn.conn)
	go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}

func (t *Transport) dial(network, addr string) (c net.Conn, err error) {
	if t.Dial != nil {
		return t.Dial(network, addr)
	}
	return net.Dial(network, addr)
}
```

这里用到了docker中设置的`httpTransport.Dial`，如下：

```go
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
```

继续分析`func (c *Client) Do(req *Request) (resp *Response, err error)`，如下：

```go
...
case ReceiveTimeout:
    httpTransport.Dial = func(proto string, addr string) (net.Conn, error) {
        conn, err := net.Dial(proto, addr)
        if err != nil {
            return nil, err
        }
        conn = utils.NewTimeoutConn(conn, 1*time.Minute)
        return conn, nil
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
...

// Dial connects to the address on the named network.
//
// Known networks are "tcp", "tcp4" (IPv4-only), "tcp6" (IPv6-only),
// "udp", "udp4" (IPv4-only), "udp6" (IPv6-only), "ip", "ip4"
// (IPv4-only), "ip6" (IPv6-only), "unix", "unixgram" and
// "unixpacket".
//
// For TCP and UDP networks, addresses have the form host:port.
// If host is a literal IPv6 address or host name, it must be enclosed
// in square brackets as in "[::1]:80", "[ipv6-host]:http" or
// "[ipv6-host%zone]:80".
// The functions JoinHostPort and SplitHostPort manipulate addresses
// in this form.
//
// Examples:
//	Dial("tcp", "12.34.56.78:80")
//	Dial("tcp", "google.com:http")
//	Dial("tcp", "[2001:db8::1]:http")
//	Dial("tcp", "[fe80::1%lo0]:80")
//
// For IP networks, the network must be "ip", "ip4" or "ip6" followed
// by a colon and a protocol number or name and the addr must be a
// literal IP address.
//
// Examples:
//	Dial("ip4:1", "127.0.0.1")
//	Dial("ip6:ospf", "::1")
//
// For Unix networks, the address must be a file system path.
func Dial(network, address string) (Conn, error) {
	var d Dialer
	return d.Dial(network, address)
}

// Dial connects to the address on the named network.
//
// See func Dial for a description of the network and address
// parameters.
func (d *Dialer) Dial(network, address string) (Conn, error) {
	ra, err := resolveAddr("dial", network, address, d.deadline())
	if err != nil {
		return nil, &OpError{Op: "dial", Net: network, Addr: nil, Err: err}
	}
	dialer := func(deadline time.Time) (Conn, error) {
		return dialSingle(network, address, d.LocalAddr, ra.toAddr(), deadline)
	}
	if ras, ok := ra.(addrList); ok && d.DualStack && network == "tcp" {
		dialer = func(deadline time.Time) (Conn, error) {
			return dialMulti(network, address, d.LocalAddr, ras, deadline)
		}
	}
	c, err := dial(network, ra.toAddr(), dialer, d.deadline())
	if d.KeepAlive > 0 && err == nil {
		if tc, ok := c.(*TCPConn); ok {
			tc.SetKeepAlive(true)
			tc.SetKeepAlivePeriod(d.KeepAlive)
			testHookSetKeepAlive()
		}
	}
	return c, err
}

// persistConn wraps a connection, usually a persistent one
// (but may be used for non-keep-alive requests as well)
type persistConn struct {
	t        *Transport
	cacheKey connectMethodKey
	conn     net.Conn
	tlsState *tls.ConnectionState
	br       *bufio.Reader       // from conn
	sawEOF   bool                // whether we've seen EOF from conn; owned by readLoop
	bw       *bufio.Writer       // to conn
	reqch    chan requestAndChan // written by roundTrip; read by readLoop
	writech  chan writeRequest   // written by roundTrip; read by writeLoop
	closech  chan struct{}       // closed when conn closed
	isProxy  bool
	// writeErrCh passes the request write error (usually nil)
	// from the writeLoop goroutine to the readLoop which passes
	// it off to the res.Body reader, which then uses it to decide
	// whether or not a connection can be reused. Issue 7569.
	writeErrCh chan error

	lk                   sync.Mutex // guards following fields
	numExpectedResponses int
	closed               bool // whether conn has been closed
	broken               bool // an error has happened on this connection; marked broken so it's not reused.
	// mutateHeaderFunc is an optional func to modify extra
	// headers on each outbound request before it's written. (the
	// original Request given to RoundTrip is not modified)
	mutateHeaderFunc func(Header)
}

func (t *Transport) dialConn(cm connectMethod) (*persistConn, error) {
	conn, err := t.dial("tcp", cm.addr())
	if err != nil {
		if cm.proxyURL != nil {
			err = fmt.Errorf("http: error connecting to proxy %s: %v", cm.proxyURL, err)
		}
		return nil, err
	}

	pa := cm.proxyAuth()

	pconn := &persistConn{
		t:          t,
		cacheKey:   cm.key(),
		conn:       conn,
		reqch:      make(chan requestAndChan, 1),
		writech:    make(chan writeRequest, 1),
		closech:    make(chan struct{}),
		writeErrCh: make(chan error, 1),
	}

	switch {
	case cm.proxyURL == nil:
		// Do nothing.
	case cm.targetScheme == "http":
		pconn.isProxy = true
		if pa != "" {
			pconn.mutateHeaderFunc = func(h Header) {
				h.Set("Proxy-Authorization", pa)
			}
		}
	case cm.targetScheme == "https":
		connectReq := &Request{
			Method: "CONNECT",
			URL:    &url.URL{Opaque: cm.targetAddr},
			Host:   cm.targetAddr,
			Header: make(Header),
		}
		if pa != "" {
			connectReq.Header.Set("Proxy-Authorization", pa)
		}
		connectReq.Write(conn)

		// Read response.
		// Okay to use and discard buffered reader here, because
		// TLS server will not speak until spoken to.
		br := bufio.NewReader(conn)
		resp, err := ReadResponse(br, connectReq)
		if err != nil {
			conn.Close()
			return nil, err
		}
		if resp.StatusCode != 200 {
			f := strings.SplitN(resp.Status, " ", 2)
			conn.Close()
			return nil, errors.New(f[1])
		}
	}

	if cm.targetScheme == "https" {
		// Initiate TLS and check remote host name against certificate.
		cfg := t.TLSClientConfig
		if cfg == nil || cfg.ServerName == "" {
			host := cm.tlsHost()
			if cfg == nil {
				cfg = &tls.Config{ServerName: host}
			} else {
				clone := *cfg // shallow clone
				clone.ServerName = host
				cfg = &clone
			}
		}
		plainConn := conn
		tlsConn := tls.Client(plainConn, cfg)
		errc := make(chan error, 2)
		var timer *time.Timer // for canceling TLS handshake
		if d := t.TLSHandshakeTimeout; d != 0 {
			timer = time.AfterFunc(d, func() {
				errc <- tlsHandshakeTimeoutError{}
			})
		}
		go func() {
			err := tlsConn.Handshake()
			if timer != nil {
				timer.Stop()
			}
			errc <- err
		}()
		if err := <-errc; err != nil {
			plainConn.Close()
			return nil, err
		}
		if !cfg.InsecureSkipVerify {
			if err := tlsConn.VerifyHostname(cfg.ServerName); err != nil {
				plainConn.Close()
				return nil, err
			}
		}
		cs := tlsConn.ConnectionState()
		pconn.tlsState = &cs
		pconn.conn = tlsConn
	}

	pconn.br = bufio.NewReader(noteEOFReader{pconn.conn, &pconn.sawEOF})
	pconn.bw = bufio.NewWriter(pconn.conn)
	go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}

// Writer implements buffering for an io.Writer object.
// If an error occurs writing to a Writer, no more data will be
// accepted and all subsequent writes will return the error.
// After all data has been written, the client should call the
// Flush method to guarantee all data has been forwarded to
// the underlying io.Writer.
type Writer struct {
	err error
	buf []byte
	n   int
	wr  io.Writer
}

// NewWriterSize returns a new Writer whose buffer has at least the specified
// size. If the argument io.Writer is already a Writer with large enough
// size, it returns the underlying Writer.
func NewWriterSize(w io.Writer, size int) *Writer {
	// Is it already a Writer?
	b, ok := w.(*Writer)
	if ok && len(b.buf) >= size {
		return b
	}
	if size <= 0 {
		size = defaultBufSize
	}
	return &Writer{
		buf: make([]byte, size),
		wr:  w,
	}
}

// NewWriter returns a new Writer whose buffer has the default size.
func NewWriter(w io.Writer) *Writer {
	return NewWriterSize(w, defaultBufSize)
}

// Reader implements buffering for an io.Reader object.
type Reader struct {
	buf          []byte
	rd           io.Reader
	r, w         int
	err          error
	lastByte     int
	lastRuneSize int
}

const minReadBufferSize = 16
const maxConsecutiveEmptyReads = 100

// NewReaderSize returns a new Reader whose buffer has at least the specified
// size. If the argument io.Reader is already a Reader with large enough
// size, it returns the underlying Reader.
func NewReaderSize(rd io.Reader, size int) *Reader {
	// Is it already a Reader?
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
	}
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	r.reset(make([]byte, size), rd)
	return r
}

// NewReader returns a new Reader whose buffer has the default size.
func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}

// Reset discards any buffered data, resets all state, and switches
// the buffered reader to read from r.
func (b *Reader) Reset(r io.Reader) {
	b.reset(b.buf, r)
}

func (b *Reader) reset(buf []byte, r io.Reader) {
	*b = Reader{
		buf:          buf,
		rd:           r,
		lastByte:     -1,
		lastRuneSize: -1,
	}
}

func (pc *persistConn) writeLoop() {
	for {
		select {
		case wr := <-pc.writech:
			if pc.isBroken() {
				wr.ch <- errors.New("http: can't write HTTP request on broken connection")
				continue
			}
			err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra)
			if err == nil {
				err = pc.bw.Flush()
			}
			if err != nil {
				pc.markBroken()
				wr.req.Request.closeBody()
			}
			pc.writeErrCh <- err // to the body reader, which might recycle us
			wr.ch <- err         // to the roundTrip function
		case <-pc.closech:
			return
		}
	}
}

func (pc *persistConn)  readLoop() {
	alive := true

	for alive {
		pb, err := pc.br.Peek(1)

		pc.lk.Lock()
		if pc.numExpectedResponses == 0 {
			if !pc.closed {
				pc.closeLocked()
				if len(pb) > 0 {
					log.Printf("Unsolicited response received on idle HTTP channel starting with %q; err=%v",
						string(pb), err)
				}
			}
			pc.lk.Unlock()
			return
		}
		pc.lk.Unlock()

		rc := <-pc.reqch

		var resp *Response
		if err == nil {
			resp, err = ReadResponse(pc.br, rc.req)
			if err == nil && resp.StatusCode == 100 {
				// Skip any 100-continue for now.
				// TODO(bradfitz): if rc.req had "Expect: 100-continue",
				// actually block the request body write and signal the
				// writeLoop now to begin sending it. (Issue 2184) For now we
				// eat it, since we're never expecting one.
				resp, err = ReadResponse(pc.br, rc.req)
			}
		}

		if resp != nil {
			resp.TLS = pc.tlsState
		}

		hasBody := resp != nil && rc.req.Method != "HEAD" && resp.ContentLength != 0

		if err != nil {
			pc.close()
		} else {
			if rc.addedGzip && hasBody && resp.Header.Get("Content-Encoding") == "gzip" {
				resp.Header.Del("Content-Encoding")
				resp.Header.Del("Content-Length")
				resp.ContentLength = -1
				resp.Body = &gzipReader{body: resp.Body}
			}
			resp.Body = &bodyEOFSignal{body: resp.Body}
		}

		if err != nil || resp.Close || rc.req.Close || resp.StatusCode <= 199 {
			// Don't do keep-alive on error if either party requested a close
			// or we get an unexpected informational (1xx) response.
			// StatusCode 100 is already handled above.
			alive = false
		}

		var waitForBodyRead chan bool
		if hasBody {
			waitForBodyRead = make(chan bool, 2)
			resp.Body.(*bodyEOFSignal).earlyCloseFn = func() error {
				// Sending false here sets alive to
				// false and closes the connection
				// below.
				waitForBodyRead <- false
				return nil
			}
			resp.Body.(*bodyEOFSignal).fn = func(err error) {
				waitForBodyRead <- alive &&
					err == nil &&
					!pc.sawEOF &&
					pc.wroteRequest() &&
					pc.t.putIdleConn(pc)
			}
		}

		if alive && !hasBody {
			alive = !pc.sawEOF &&
				pc.wroteRequest() &&
				pc.t.putIdleConn(pc)
		}

		rc.ch <- responseAndError{resp, err}

		// Wait for the just-returned response body to be fully consumed
		// before we race and peek on the underlying bufio reader.
		if waitForBodyRead != nil {
			select {
			case alive = <-waitForBodyRead:
			case <-pc.closech:
				alive = false
			}
		}

		pc.t.setReqCanceler(rc.req, nil)

		if !alive {
			pc.close()
		}
	}
}

// ReadResponse reads and returns an HTTP response from r.
// The req parameter optionally specifies the Request that corresponds
// to this Response. If nil, a GET request is assumed.
// Clients must call resp.Body.Close when finished reading resp.Body.
// After that call, clients can inspect resp.Trailer to find key/value
// pairs included in the response trailer.
func ReadResponse(r *bufio.Reader, req *Request) (*Response, error) {
	tp := textproto.NewReader(r)
	resp := &Response{
		Request: req,
	}

	// Parse the first line of the response.
	line, err := tp.ReadLine()
	if err != nil {
		if err == io.EOF {
			err = io.ErrUnexpectedEOF
		}
		return nil, err
	}
	f := strings.SplitN(line, " ", 3)
	if len(f) < 2 {
		return nil, &badStringError{"malformed HTTP response", line}
	}
	reasonPhrase := ""
	if len(f) > 2 {
		reasonPhrase = f[2]
	}
	resp.Status = f[1] + " " + reasonPhrase
	resp.StatusCode, err = strconv.Atoi(f[1])
	if err != nil {
		return nil, &badStringError{"malformed HTTP status code", f[1]}
	}

	resp.Proto = f[0]
	var ok bool
	if resp.ProtoMajor, resp.ProtoMinor, ok = ParseHTTPVersion(resp.Proto); !ok {
		return nil, &badStringError{"malformed HTTP version", resp.Proto}
	}

	// Parse the response headers.
	mimeHeader, err := tp.ReadMIMEHeader()
	if err != nil {
		if err == io.EOF {
			err = io.ErrUnexpectedEOF
		}
		return nil, err
	}
	resp.Header = Header(mimeHeader)

	fixPragmaCacheControl(resp.Header)

	err = readTransfer(resp, r)
	if err != nil {
		return nil, err
	}

	return resp, nil
}

// Response represents the response from an HTTP request.
//
type Response struct {
	Status     string // e.g. "200 OK"
	StatusCode int    // e.g. 200
	Proto      string // e.g. "HTTP/1.0"
	ProtoMajor int    // e.g. 1
	ProtoMinor int    // e.g. 0

	// Header maps header keys to values.  If the response had multiple
	// headers with the same key, they may be concatenated, with comma
	// delimiters.  (Section 4.2 of RFC 2616 requires that multiple headers
	// be semantically equivalent to a comma-delimited sequence.) Values
	// duplicated by other fields in this struct (e.g., ContentLength) are
	// omitted from Header.
	//
	// Keys in the map are canonicalized (see CanonicalHeaderKey).
	Header Header

	// Body represents the response body.
	//
	// The http Client and Transport guarantee that Body is always
	// non-nil, even on responses without a body or responses with
	// a zero-length body. It is the caller's responsibility to
	// close Body.
	//
	// The Body is automatically dechunked if the server replied
	// with a "chunked" Transfer-Encoding.
	Body io.ReadCloser

	// ContentLength records the length of the associated content.  The
	// value -1 indicates that the length is unknown.  Unless Request.Method
	// is "HEAD", values >= 0 indicate that the given number of bytes may
	// be read from Body.
	ContentLength int64

	// Contains transfer encodings from outer-most to inner-most. Value is
	// nil, means that "identity" encoding is used.
	TransferEncoding []string

	// Close records whether the header directed that the connection be
	// closed after reading Body.  The value is advice for clients: neither
	// ReadResponse nor Response.Write ever closes a connection.
	Close bool

	// Trailer maps trailer keys to values, in the same
	// format as the header.
	Trailer Header

	// The Request that was sent to obtain this Response.
	// Request's Body is nil (having already been consumed).
	// This is only populated for Client requests.
	Request *Request

	// TLS contains information about the TLS connection on which the
	// response was received. It is nil for unencrypted responses.
	// The pointer is shared between responses and should not be
	// modified.
	TLS *tls.ConnectionState
}

// msg is *Request or *Response.
func readTransfer(msg interface{}, r *bufio.Reader) (err error) {
	t := &transferReader{RequestMethod: "GET"}

	// Unify input
	isResponse := false
	switch rr := msg.(type) {
	case *Response:
		t.Header = rr.Header
		t.StatusCode = rr.StatusCode
		t.ProtoMajor = rr.ProtoMajor
		t.ProtoMinor = rr.ProtoMinor
		t.Close = shouldClose(t.ProtoMajor, t.ProtoMinor, t.Header)
		isResponse = true
		if rr.Request != nil {
			t.RequestMethod = rr.Request.Method
		}
	case *Request:
		t.Header = rr.Header
		t.ProtoMajor = rr.ProtoMajor
		t.ProtoMinor = rr.ProtoMinor
		// Transfer semantics for Requests are exactly like those for
		// Responses with status code 200, responding to a GET method
		t.StatusCode = 200
	default:
		panic("unexpected type")
	}

	// Default to HTTP/1.1
	if t.ProtoMajor == 0 && t.ProtoMinor == 0 {
		t.ProtoMajor, t.ProtoMinor = 1, 1
	}

	// Transfer encoding, content length
	t.TransferEncoding, err = fixTransferEncoding(t.RequestMethod, t.Header)
	if err != nil {
		return err
	}

	realLength, err := fixLength(isResponse, t.StatusCode, t.RequestMethod, t.Header, t.TransferEncoding)
	if err != nil {
		return err
	}
	if isResponse && t.RequestMethod == "HEAD" {
		if n, err := parseContentLength(t.Header.get("Content-Length")); err != nil {
			return err
		} else {
			t.ContentLength = n
		}
	} else {
		t.ContentLength = realLength
	}

	// Trailer
	t.Trailer, err = fixTrailer(t.Header, t.TransferEncoding)
	if err != nil {
		return err
	}

	// If there is no Content-Length or chunked Transfer-Encoding on a *Response
	// and the status is not 1xx, 204 or 304, then the body is unbounded.
	// See RFC2616, section 4.4.
	switch msg.(type) {
	case *Response:
		if realLength == -1 &&
			!chunked(t.TransferEncoding) &&
			bodyAllowedForStatus(t.StatusCode) {
			// Unbounded body.
			t.Close = true
		}
	}

	// Prepare body reader.  ContentLength < 0 means chunked encoding
	// or close connection when finished, since multipart is not supported yet
	switch {
	case chunked(t.TransferEncoding):
		if noBodyExpected(t.RequestMethod) {
			t.Body = eofReader
		} else {
			t.Body = &body{src: newChunkedReader(r), hdr: msg, r: r, closing: t.Close}
		}
	case realLength == 0:
		t.Body = eofReader
	case realLength > 0:
		t.Body = &body{src: io.LimitReader(r, realLength), closing: t.Close}
	default:
		// realLength < 0, i.e. "Content-Length" not mentioned in header
		if t.Close {
			// Close semantics (i.e. HTTP/1.0)
			t.Body = &body{src: r, closing: t.Close}
		} else {
			// Persistent connection (i.e. HTTP/1.1)
			t.Body = eofReader
		}
	}

	// Unify output
	switch rr := msg.(type) {
	case *Request:
		rr.Body = t.Body
		rr.ContentLength = t.ContentLength
		rr.TransferEncoding = t.TransferEncoding
		rr.Close = t.Close
		rr.Trailer = t.Trailer
	case *Response:
		rr.Body = t.Body
		rr.ContentLength = t.ContentLength
		rr.TransferEncoding = t.TransferEncoding
		rr.Close = t.Close
		rr.Trailer = t.Trailer
	}

	return nil
}

type transferReader struct {
	// Input
	Header        Header
	StatusCode    int
	RequestMethod string
	ProtoMajor    int
	ProtoMinor    int
	// Output
	Body             io.ReadCloser
	ContentLength    int64
	TransferEncoding []string
	Close            bool
	Trailer          Header
}

func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
	pc.t.setReqCanceler(req.Request, pc.cancelRequest)
	pc.lk.Lock()
	pc.numExpectedResponses++
	headerFn := pc.mutateHeaderFunc
	pc.lk.Unlock()

	if headerFn != nil {
		headerFn(req.extraHeaders())
	}

	// Ask for a compressed version if the caller didn't set their
	// own value for Accept-Encoding. We only attempted to
	// uncompress the gzip stream if we were the layer that
	// requested it.
	requestedGzip := false
	if !pc.t.DisableCompression && req.Header.Get("Accept-Encoding") == "" && req.Method != "HEAD" {
		// Request gzip only, not deflate. Deflate is ambiguous and
		// not as universally supported anyway.
		// See: http://www.gzip.org/zlib/zlib_faq.html#faq38
		//
		// Note that we don't request this for HEAD requests,
		// due to a bug in nginx:
		//   http://trac.nginx.org/nginx/ticket/358
		//   http://golang.org/issue/5522
		requestedGzip = true
		req.extraHeaders().Set("Accept-Encoding", "gzip")
	}

	// Write the request concurrently with waiting for a response,
	// in case the server decides to reply before reading our full
	// request body.
	writeErrCh := make(chan error, 1)
	pc.writech <- writeRequest{req, writeErrCh}

	resc := make(chan responseAndError, 1)
	pc.reqch <- requestAndChan{req.Request, resc, requestedGzip}

	var re responseAndError
	var pconnDeadCh = pc.closech
	var failTicker <-chan time.Time
	var respHeaderTimer <-chan time.Time
WaitResponse:
	for {
		select {
		case err := <-writeErrCh:
			if err != nil {
				re = responseAndError{nil, err}
				pc.close()
				break WaitResponse
			}
			if d := pc.t.ResponseHeaderTimeout; d > 0 {
				respHeaderTimer = time.After(d)
			}
		case <-pconnDeadCh:
			// The persist connection is dead. This shouldn't
			// usually happen (only with Connection: close responses
			// with no response bodies), but if it does happen it
			// means either a) the remote server hung up on us
			// prematurely, or b) the readLoop sent us a response &
			// closed its closech at roughly the same time, and we
			// selected this case first, in which case a response
			// might still be coming soon.
			//
			// We can't avoid the select race in b) by using a unbuffered
			// resc channel instead, because then goroutines can
			// leak if we exit due to other errors.
			pconnDeadCh = nil                               // avoid spinning
			failTicker = time.After(100 * time.Millisecond) // arbitrary time to wait for resc
		case <-failTicker:
			re = responseAndError{err: errClosed}
			break WaitResponse
		case <-respHeaderTimer:
			pc.close()
			re = responseAndError{err: errTimeout}
			break WaitResponse
		case re = <-resc:
			break WaitResponse
		}
	}

	pc.lk.Lock()
	pc.numExpectedResponses--
	pc.lk.Unlock()

	if re.err != nil {
		pc.t.setReqCanceler(req.Request, nil)
	}
	return re.res, re.err
}

// A writeRequest is sent by the readLoop's goroutine to the
// writeLoop's goroutine to write a request while the read loop
// concurrently waits on both the write response and the server's
// reply.
type writeRequest struct {
	req *transportRequest
	ch  chan<- error
}
// extraHeaders may be nil
func (req *Request) write(w io.Writer, usingProxy bool, extraHeaders Header) error {
	host := req.Host
	if host == "" {
		if req.URL == nil {
			return errors.New("http: Request.Write on Request with no Host or URL set")
		}
		host = req.URL.Host
	}

	ruri := req.URL.RequestURI()
	if usingProxy && req.URL.Scheme != "" && req.URL.Opaque == "" {
		ruri = req.URL.Scheme + "://" + host + ruri
	} else if req.Method == "CONNECT" && req.URL.Path == "" {
		// CONNECT requests normally give just the host and port, not a full URL.
		ruri = host
	}
	// TODO(bradfitz): escape at least newlines in ruri?

	// Wrap the writer in a bufio Writer if it's not already buffered.
	// Don't always call NewWriter, as that forces a bytes.Buffer
	// and other small bufio Writers to have a minimum 4k buffer
	// size.
	var bw *bufio.Writer
	if _, ok := w.(io.ByteWriter); !ok {
		bw = bufio.NewWriter(w)
		w = bw
	}

	fmt.Fprintf(w, "%s %s HTTP/1.1\r\n", valueOrDefault(req.Method, "GET"), ruri)

	// Header lines
	fmt.Fprintf(w, "Host: %s\r\n", host)

	// Use the defaultUserAgent unless the Header contains one, which
	// may be blank to not send the header.
	userAgent := defaultUserAgent
	if req.Header != nil {
		if ua := req.Header["User-Agent"]; len(ua) > 0 {
			userAgent = ua[0]
		}
	}
	if userAgent != "" {
		fmt.Fprintf(w, "User-Agent: %s\r\n", userAgent)
	}

	// Process Body,ContentLength,Close,Trailer
	tw, err := newTransferWriter(req)
	if err != nil {
		return err
	}
	err = tw.WriteHeader(w)
	if err != nil {
		return err
	}

	err = req.Header.WriteSubset(w, reqWriteExcludeHeader)
	if err != nil {
		return err
	}

	if extraHeaders != nil {
		err = extraHeaders.Write(w)
		if err != nil {
			return err
		}
	}

	io.WriteString(w, "\r\n")

	// Write body and trailer
	err = tw.WriteBody(w)
	if err != nil {
		return err
	}

	if bw != nil {
		return bw.Flush()
	}
	return nil
}
```

使用`net/http client`的一般流程如下:

```go
tr := &http.Transport{
	TLSClientConfig:    &tls.Config{RootCAs: pool},
	DisableCompression: true,
}

tr.Dial = func(proto string, addr string) (net.Conn, error) {
    conn, err := net.Dial(proto, addr)
    if err != nil {
        return nil, err
    }
    conn = utils.NewTimeoutConn(conn, 1*time.Minute)
    return conn, nil
}
    
client := &http.Client{
	Transport: tr,
	CheckRedirect: redirectPolicyFunc,
}

req, err := http.NewRequest("GET", "http://example.com", nil)
// ...
req.Header.Add("If-None-Match", `W/"wyzzy"`)
resp, err := client.Do(req)
// ...
```

如下是流程图：

![](/public/img/golang_client_flow/client_flow.png)

数据结构：

* Client

`http.Client`是客户端请求数据结构，它包装了`RoundTripper`，客户端使用时应该创建一个`Client`实例，并重复使用(多goroutines安全)，如下：

```go
// A Client is an HTTP client. Its zero value (DefaultClient) is a
// usable client that uses DefaultTransport.
//
// The Client's Transport typically has internal state (cached TCP
// connections), so Clients should be reused instead of created as
// needed. Clients are safe for concurrent use by multiple goroutines.
//
// A Client is higher-level than a RoundTripper (such as Transport)
// and additionally handles HTTP details such as cookies and
// redirects.
type Client struct {
	// Transport specifies the mechanism by which individual
	// HTTP requests are made.
	// If nil, DefaultTransport is used.
	Transport RoundTripper

	// CheckRedirect specifies the policy for handling redirects.
	// If CheckRedirect is not nil, the client calls it before
	// following an HTTP redirect. The arguments req and via are
	// the upcoming request and the requests made already, oldest
	// first. If CheckRedirect returns an error, the Client's Get
	// method returns both the previous Response and
	// CheckRedirect's error (wrapped in a url.Error) instead of
	// issuing the Request req.
	//
	// If CheckRedirect is nil, the Client uses its default policy,
	// which is to stop after 10 consecutive requests.
	CheckRedirect func(req *Request, via []*Request) error

	// Jar specifies the cookie jar.
	// If Jar is nil, cookies are not sent in requests and ignored
	// in responses.
	Jar CookieJar

	// Timeout specifies a time limit for requests made by this
	// Client. The timeout includes connection time, any
	// redirects, and reading the response body. The timer remains
	// running after Get, Head, Post, or Do return and will
	// interrupt reading of the Response.Body.
	//
	// A Timeout of zero means no timeout.
	//
	// The Client's Transport must support the CancelRequest
	// method or Client will return errors when attempting to make
	// a request with Get, Head, Post, or Do. Client's default
	// Transport (DefaultTransport) supports CancelRequest.
	Timeout time.Duration
}
```

* Transport

http.Transport代表client与server之间的传输管道，它比net.Conn更加高层，它基于net.Conn(实际上是http.persistConn)进行数据传输，并管理空闲的http.persistConn，如下：

```go
// Transport is an implementation of RoundTripper that supports http,
// https, and http proxies (for either http or https with CONNECT).
// Transport can also cache connections for future re-use.
type Transport struct {
	idleMu      sync.Mutex
	idleConn    map[connectMethodKey][]*persistConn
	idleConnCh  map[connectMethodKey]chan *persistConn
	reqMu       sync.Mutex
	reqCanceler map[*Request]func()
	altMu       sync.RWMutex
	altProto    map[string]RoundTripper // nil or map of URI scheme => RoundTripper

	// Proxy specifies a function to return a proxy for a given
	// Request. If the function returns a non-nil error, the
	// request is aborted with the provided error.
	// If Proxy is nil or returns a nil *URL, no proxy is used.
	Proxy func(*Request) (*url.URL, error)

	// Dial specifies the dial function for creating TCP
	// connections.
	// If Dial is nil, net.Dial is used.
	Dial func(network, addr string) (net.Conn, error)

	// TLSClientConfig specifies the TLS configuration to use with
	// tls.Client. If nil, the default configuration is used.
	TLSClientConfig *tls.Config

	// TLSHandshakeTimeout specifies the maximum amount of time waiting to
	// wait for a TLS handshake. Zero means no timeout.
	TLSHandshakeTimeout time.Duration

	// DisableKeepAlives, if true, prevents re-use of TCP connections
	// between different HTTP requests.
	DisableKeepAlives bool

	// DisableCompression, if true, prevents the Transport from
	// requesting compression with an "Accept-Encoding: gzip"
	// request header when the Request contains no existing
	// Accept-Encoding value. If the Transport requests gzip on
	// its own and gets a gzipped response, it's transparently
	// decoded in the Response.Body. However, if the user
	// explicitly requested gzip it is not automatically
	// uncompressed.
	DisableCompression bool

	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) to keep per-host.  If zero,
	// DefaultMaxIdleConnsPerHost is used.
	MaxIdleConnsPerHost int

	// ResponseHeaderTimeout, if non-zero, specifies the amount of
	// time to wait for a server's response headers after fully
	// writing the request (including its body, if any). This
	// time does not include the time to read the response body.
	ResponseHeaderTimeout time.Duration

	// TODO: tunable on global max cached connections
	// TODO: tunable on timeout on cached connections
}
```

* persistConn

http.persistConn对net.Conn进行了包装，以实现持久的connection（keep-alive），如下：

```go
// persistConn wraps a connection, usually a persistent one
// (but may be used for non-keep-alive requests as well)
type persistConn struct {
	t        *Transport
	cacheKey connectMethodKey
	conn     net.Conn
	tlsState *tls.ConnectionState
	br       *bufio.Reader       // from conn
	sawEOF   bool                // whether we've seen EOF from conn; owned by readLoop
	bw       *bufio.Writer       // to conn
	reqch    chan requestAndChan // written by roundTrip; read by readLoop
	writech  chan writeRequest   // written by roundTrip; read by writeLoop
	closech  chan struct{}       // closed when conn closed
	isProxy  bool
	// writeErrCh passes the request write error (usually nil)
	// from the writeLoop goroutine to the readLoop which passes
	// it off to the res.Body reader, which then uses it to decide
	// whether or not a connection can be reused. Issue 7569.
	writeErrCh chan error

	lk                   sync.Mutex // guards following fields
	numExpectedResponses int
	closed               bool // whether conn has been closed
	broken               bool // an error has happened on this connection; marked broken so it's not reused.
	// mutateHeaderFunc is an optional func to modify extra
	// headers on each outbound request before it's written. (the
	// original Request given to RoundTrip is not modified)
	mutateHeaderFunc func(Header)
}
```

执行流程：

* 1、`roundTrip`向`pc.writech`管道发消息，`writeLoop`接收该消息，之后向tcp socket写http request，如下：

```go
// Write the request concurrently with waiting for a response,
// in case the server decides to reply before reading our full
// request body.
writeErrCh := make(chan error, 1)
pc.writech <- writeRequest{req, writeErrCh}

// persistConn wraps a connection, usually a persistent one
// (but may be used for non-keep-alive requests as well)
type persistConn struct {
	t        *Transport
	cacheKey connectMethodKey
	conn     net.Conn
	tlsState *tls.ConnectionState
	br       *bufio.Reader       // from conn
	sawEOF   bool                // whether we've seen EOF from conn; owned by readLoop
	bw       *bufio.Writer       // to conn
	reqch    chan requestAndChan // written by roundTrip; read by readLoop
	writech  chan writeRequest   // written by roundTrip; read by writeLoop
	closech  chan struct{}       // closed when conn closed
	isProxy  bool
	// writeErrCh passes the request write error (usually nil)
	// from the writeLoop goroutine to the readLoop which passes
	// it off to the res.Body reader, which then uses it to decide
	// whether or not a connection can be reused. Issue 7569.
	writeErrCh chan error

	lk                   sync.Mutex // guards following fields
	numExpectedResponses int
	closed               bool // whether conn has been closed
	broken               bool // an error has happened on this connection; marked broken so it's not reused.
	// mutateHeaderFunc is an optional func to modify extra
	// headers on each outbound request before it's written. (the
	// original Request given to RoundTrip is not modified)
	mutateHeaderFunc func(Header)
}

func (t *Transport) dialConn(cm connectMethod) (*persistConn, error) {
	conn, err := t.dial("tcp", cm.addr())
	if err != nil {
		if cm.proxyURL != nil {
			err = fmt.Errorf("http: error connecting to proxy %s: %v", cm.proxyURL, err)
		}
		return nil, err
	}

	pa := cm.proxyAuth()

	pconn := &persistConn{
		t:          t,
		cacheKey:   cm.key(),
		conn:       conn,
		reqch:      make(chan requestAndChan, 1),
		writech:    make(chan writeRequest, 1),
		closech:    make(chan struct{}),
		writeErrCh: make(chan error, 1),
	}

	switch {
	case cm.proxyURL == nil:
		// Do nothing.
	case cm.targetScheme == "http":
		pconn.isProxy = true
		if pa != "" {
			pconn.mutateHeaderFunc = func(h Header) {
				h.Set("Proxy-Authorization", pa)
			}
		}
	case cm.targetScheme == "https":
		connectReq := &Request{
			Method: "CONNECT",
			URL:    &url.URL{Opaque: cm.targetAddr},
			Host:   cm.targetAddr,
			Header: make(Header),
		}
		if pa != "" {
			connectReq.Header.Set("Proxy-Authorization", pa)
		}
		connectReq.Write(conn)

		// Read response.
		// Okay to use and discard buffered reader here, because
		// TLS server will not speak until spoken to.
		br := bufio.NewReader(conn)
		resp, err := ReadResponse(br, connectReq)
		if err != nil {
			conn.Close()
			return nil, err
		}
		if resp.StatusCode != 200 {
			f := strings.SplitN(resp.Status, " ", 2)
			conn.Close()
			return nil, errors.New(f[1])
		}
	}

	if cm.targetScheme == "https" {
		// Initiate TLS and check remote host name against certificate.
		cfg := t.TLSClientConfig
		if cfg == nil || cfg.ServerName == "" {
			host := cm.tlsHost()
			if cfg == nil {
				cfg = &tls.Config{ServerName: host}
			} else {
				clone := *cfg // shallow clone
				clone.ServerName = host
				cfg = &clone
			}
		}
		plainConn := conn
		tlsConn := tls.Client(plainConn, cfg)
		errc := make(chan error, 2)
		var timer *time.Timer // for canceling TLS handshake
		if d := t.TLSHandshakeTimeout; d != 0 {
			timer = time.AfterFunc(d, func() {
				errc <- tlsHandshakeTimeoutError{}
			})
		}
		go func() {
			err := tlsConn.Handshake()
			if timer != nil {
				timer.Stop()
			}
			errc <- err
		}()
		if err := <-errc; err != nil {
			plainConn.Close()
			return nil, err
		}
		if !cfg.InsecureSkipVerify {
			if err := tlsConn.VerifyHostname(cfg.ServerName); err != nil {
				plainConn.Close()
				return nil, err
			}
		}
		cs := tlsConn.ConnectionState()
		pconn.tlsState = &cs
		pconn.conn = tlsConn
	}

	pconn.br = bufio.NewReader(noteEOFReader{pconn.conn, &pconn.sawEOF})
	pconn.bw = bufio.NewWriter(pconn.conn)
	go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}

func (pc *persistConn) writeLoop() {
	for {
		select {
		case wr := <-pc.writech:
			if pc.isBroken() {
				wr.ch <- errors.New("http: can't write HTTP request on broken connection")
				continue
			}
			err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra)
			if err == nil {
				err = pc.bw.Flush()
			}
			if err != nil {
				pc.markBroken()
				wr.req.Request.closeBody()
			}
			pc.writeErrCh <- err // to the body reader, which might recycle us
			wr.ch <- err         // to the roundTrip function
		case <-pc.closech:
			return
		}
	}
}
```

* 2、`roundTrip`向`pc.reqch`发消息，`readLoop`接收该消息，并从tcp socket读取http response，如下:

```go
...
resc := make(chan responseAndError, 1)
pc.reqch <- requestAndChan{req.Request, resc, requestedGzip}
...

func (pc *persistConn)  readLoop() {
	alive := true

	for alive {
		pb, err := pc.br.Peek(1)

		pc.lk.Lock()
		if pc.numExpectedResponses == 0 {
			if !pc.closed {
				pc.closeLocked()
				if len(pb) > 0 {
					log.Printf("Unsolicited response received on idle HTTP channel starting with %q; err=%v",
						string(pb), err)
				}
			}
			pc.lk.Unlock()
			return
		}
		pc.lk.Unlock()

		rc := <-pc.reqch

		var resp *Response
		if err == nil {
			resp, err = ReadResponse(pc.br, rc.req)
			if err == nil && resp.StatusCode == 100 {
				// Skip any 100-continue for now.
				// TODO(bradfitz): if rc.req had "Expect: 100-continue",
				// actually block the request body write and signal the
				// writeLoop now to begin sending it. (Issue 2184) For now we
				// eat it, since we're never expecting one.
				resp, err = ReadResponse(pc.br, rc.req)
			}
		}

		if resp != nil {
			resp.TLS = pc.tlsState
		}

		hasBody := resp != nil && rc.req.Method != "HEAD" && resp.ContentLength != 0

		if err != nil {
			pc.close()
		} else {
			if rc.addedGzip && hasBody && resp.Header.Get("Content-Encoding") == "gzip" {
				resp.Header.Del("Content-Encoding")
				resp.Header.Del("Content-Length")
				resp.ContentLength = -1
				resp.Body = &gzipReader{body: resp.Body}
			}
			resp.Body = &bodyEOFSignal{body: resp.Body}
		}

		if err != nil || resp.Close || rc.req.Close || resp.StatusCode <= 199 {
			// Don't do keep-alive on error if either party requested a close
			// or we get an unexpected informational (1xx) response.
			// StatusCode 100 is already handled above.
			alive = false
		}

		var waitForBodyRead chan bool
		if hasBody {
			waitForBodyRead = make(chan bool, 2)
			resp.Body.(*bodyEOFSignal).earlyCloseFn = func() error {
				// Sending false here sets alive to
				// false and closes the connection
				// below.
				waitForBodyRead <- false
				return nil
			}
			resp.Body.(*bodyEOFSignal).fn = func(err error) {
				waitForBodyRead <- alive &&
					err == nil &&
					!pc.sawEOF &&
					pc.wroteRequest() &&
					pc.t.putIdleConn(pc)
			}
		}

		if alive && !hasBody {
			alive = !pc.sawEOF &&
				pc.wroteRequest() &&
				pc.t.putIdleConn(pc)
		}

		rc.ch <- responseAndError{resp, err}

		// Wait for the just-returned response body to be fully consumed
		// before we race and peek on the underlying bufio reader.
		if waitForBodyRead != nil {
			select {
			case alive = <-waitForBodyRead:
			case <-pc.closech:
				alive = false
			}
		}

		pc.t.setReqCanceler(rc.req, nil)

		if !alive {
			pc.close()
		}
	}
}
```

* 3、`readLoop`读取http response后，将response写入到`rc.ch`管道中，roundTrip读取该消息，取消请求`pc.t.setReqCanceler(req.Request, nil)`,并返回http response，如下：

```go
...
rc.ch <- responseAndError{resp, err}
...

func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
	pc.t.setReqCanceler(req.Request, pc.cancelRequest)
	pc.lk.Lock()
	pc.numExpectedResponses++
	headerFn := pc.mutateHeaderFunc
	pc.lk.Unlock()

	if headerFn != nil {
		headerFn(req.extraHeaders())
	}

	// Ask for a compressed version if the caller didn't set their
	// own value for Accept-Encoding. We only attempted to
	// uncompress the gzip stream if we were the layer that
	// requested it.
	requestedGzip := false
	if !pc.t.DisableCompression && req.Header.Get("Accept-Encoding") == "" && req.Method != "HEAD" {
		// Request gzip only, not deflate. Deflate is ambiguous and
		// not as universally supported anyway.
		// See: http://www.gzip.org/zlib/zlib_faq.html#faq38
		//
		// Note that we don't request this for HEAD requests,
		// due to a bug in nginx:
		//   http://trac.nginx.org/nginx/ticket/358
		//   http://golang.org/issue/5522
		requestedGzip = true
		req.extraHeaders().Set("Accept-Encoding", "gzip")
	}

	// Write the request concurrently with waiting for a response,
	// in case the server decides to reply before reading our full
	// request body.
	writeErrCh := make(chan error, 1)
	pc.writech <- writeRequest{req, writeErrCh}

	resc := make(chan responseAndError, 1)
	pc.reqch <- requestAndChan{req.Request, resc, requestedGzip}

	var re responseAndError
	var pconnDeadCh = pc.closech
	var failTicker <-chan time.Time
	var respHeaderTimer <-chan time.Time
WaitResponse:
	for {
		select {
		case err := <-writeErrCh:
			if err != nil {
				re = responseAndError{nil, err}
				pc.close()
				break WaitResponse
			}
			if d := pc.t.ResponseHeaderTimeout; d > 0 {
				respHeaderTimer = time.After(d)
			}
		case <-pconnDeadCh:
			// The persist connection is dead. This shouldn't
			// usually happen (only with Connection: close responses
			// with no response bodies), but if it does happen it
			// means either a) the remote server hung up on us
			// prematurely, or b) the readLoop sent us a response &
			// closed its closech at roughly the same time, and we
			// selected this case first, in which case a response
			// might still be coming soon.
			//
			// We can't avoid the select race in b) by using a unbuffered
			// resc channel instead, because then goroutines can
			// leak if we exit due to other errors.
			pconnDeadCh = nil                               // avoid spinning
			failTicker = time.After(100 * time.Millisecond) // arbitrary time to wait for resc
		case <-failTicker:
			re = responseAndError{err: errClosed}
			break WaitResponse
		case <-respHeaderTimer:
			pc.close()
			re = responseAndError{err: errTimeout}
			break WaitResponse
		case re = <-resc:
			break WaitResponse
		}
	}

	pc.lk.Lock()
	pc.numExpectedResponses--
	pc.lk.Unlock()

	if re.err != nil {
		pc.t.setReqCanceler(req.Request, nil)
	}
	return re.res, re.err
}
```

Docker设置`httpTransport.Dial`，如下：

```go
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
```

`dialConn`使用`httpTransport.Dial`如下：

```go
func (t *Transport) dialConn(cm connectMethod) (*persistConn, error) {
	conn, err := t.dial("tcp", cm.addr())
	if err != nil {
		if cm.proxyURL != nil {
			err = fmt.Errorf("http: error connecting to proxy %s: %v", cm.proxyURL, err)
		}
		return nil, err
	}

	pa := cm.proxyAuth()

	pconn := &persistConn{
		t:          t,
		cacheKey:   cm.key(),
		conn:       conn,
		reqch:      make(chan requestAndChan, 1),
		writech:    make(chan writeRequest, 1),
		closech:    make(chan struct{}),
		writeErrCh: make(chan error, 1),
	}

	switch {
	case cm.proxyURL == nil:
		// Do nothing.
	case cm.targetScheme == "http":
		pconn.isProxy = true
		if pa != "" {
			pconn.mutateHeaderFunc = func(h Header) {
				h.Set("Proxy-Authorization", pa)
			}
		}
	case cm.targetScheme == "https":
		connectReq := &Request{
			Method: "CONNECT",
			URL:    &url.URL{Opaque: cm.targetAddr},
			Host:   cm.targetAddr,
			Header: make(Header),
		}
		if pa != "" {
			connectReq.Header.Set("Proxy-Authorization", pa)
		}
		connectReq.Write(conn)

		// Read response.
		// Okay to use and discard buffered reader here, because
		// TLS server will not speak until spoken to.
		br := bufio.NewReader(conn)
		resp, err := ReadResponse(br, connectReq)
		if err != nil {
			conn.Close()
			return nil, err
		}
		if resp.StatusCode != 200 {
			f := strings.SplitN(resp.Status, " ", 2)
			conn.Close()
			return nil, errors.New(f[1])
		}
	}

	if cm.targetScheme == "https" {
		// Initiate TLS and check remote host name against certificate.
		cfg := t.TLSClientConfig
		if cfg == nil || cfg.ServerName == "" {
			host := cm.tlsHost()
			if cfg == nil {
				cfg = &tls.Config{ServerName: host}
			} else {
				clone := *cfg // shallow clone
				clone.ServerName = host
				cfg = &clone
			}
		}
		plainConn := conn
		tlsConn := tls.Client(plainConn, cfg)
		errc := make(chan error, 2)
		var timer *time.Timer // for canceling TLS handshake
		if d := t.TLSHandshakeTimeout; d != 0 {
			timer = time.AfterFunc(d, func() {
				errc <- tlsHandshakeTimeoutError{}
			})
		}
		go func() {
			err := tlsConn.Handshake()
			if timer != nil {
				timer.Stop()
			}
			errc <- err
		}()
		if err := <-errc; err != nil {
			plainConn.Close()
			return nil, err
		}
		if !cfg.InsecureSkipVerify {
			if err := tlsConn.VerifyHostname(cfg.ServerName); err != nil {
				plainConn.Close()
				return nil, err
			}
		}
		cs := tlsConn.ConnectionState()
		pconn.tlsState = &cs
		pconn.conn = tlsConn
	}

	pconn.br = bufio.NewReader(noteEOFReader{pconn.conn, &pconn.sawEOF})
	pconn.bw = bufio.NewWriter(pconn.conn)
	go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}

func (t *Transport) dial(network, addr string) (c net.Conn, err error) {
	if t.Dial != nil {
		return t.Dial(network, addr)
	}
	return net.Dial(network, addr)
}

// Dial connects to the address on the named network.
//
// Known networks are "tcp", "tcp4" (IPv4-only), "tcp6" (IPv6-only),
// "udp", "udp4" (IPv4-only), "udp6" (IPv6-only), "ip", "ip4"
// (IPv4-only), "ip6" (IPv6-only), "unix", "unixgram" and
// "unixpacket".
//
// For TCP and UDP networks, addresses have the form host:port.
// If host is a literal IPv6 address or host name, it must be enclosed
// in square brackets as in "[::1]:80", "[ipv6-host]:http" or
// "[ipv6-host%zone]:80".
// The functions JoinHostPort and SplitHostPort manipulate addresses
// in this form.
//
// Examples:
//	Dial("tcp", "12.34.56.78:80")
//	Dial("tcp", "google.com:http")
//	Dial("tcp", "[2001:db8::1]:http")
//	Dial("tcp", "[fe80::1%lo0]:80")
//
// For IP networks, the network must be "ip", "ip4" or "ip6" followed
// by a colon and a protocol number or name and the addr must be a
// literal IP address.
//
// Examples:
//	Dial("ip4:1", "127.0.0.1")
//	Dial("ip6:ospf", "::1")
//
// For Unix networks, the address must be a file system path.
func Dial(network, address string) (Conn, error) {
	var d Dialer
	return d.Dial(network, address)
}

// Dial connects to the address on the named network.
//
// See func Dial for a description of the network and address
// parameters.
func (d *Dialer) Dial(network, address string) (Conn, error) {
	ra, err := resolveAddr("dial", network, address, d.deadline())
	if err != nil {
		return nil, &OpError{Op: "dial", Net: network, Addr: nil, Err: err}
	}
	dialer := func(deadline time.Time) (Conn, error) {
		return dialSingle(network, address, d.LocalAddr, ra.toAddr(), deadline)
	}
	if ras, ok := ra.(addrList); ok && d.DualStack && network == "tcp" {
		dialer = func(deadline time.Time) (Conn, error) {
			return dialMulti(network, address, d.LocalAddr, ras, deadline)
		}
	}
	c, err := dial(network, ra.toAddr(), dialer, d.deadline())
	if d.KeepAlive > 0 && err == nil {
		if tc, ok := c.(*TCPConn); ok {
			tc.SetKeepAlive(true)
			tc.SetKeepAlivePeriod(d.KeepAlive)
			testHookSetKeepAlive()
		}
	}
	return c, err
}

// A Dialer contains options for connecting to an address.
//
// The zero value for each field is equivalent to dialing
// without that option. Dialing with the zero value of Dialer
// is therefore equivalent to just calling the Dial function.
type Dialer struct {
	// Timeout is the maximum amount of time a dial will wait for
	// a connect to complete. If Deadline is also set, it may fail
	// earlier.
	//
	// The default is no timeout.
	//
	// With or without a timeout, the operating system may impose
	// its own earlier timeout. For instance, TCP timeouts are
	// often around 3 minutes.
	Timeout time.Duration

	// Deadline is the absolute point in time after which dials
	// will fail. If Timeout is set, it may fail earlier.
	// Zero means no deadline, or dependent on the operating system
	// as with the Timeout option.
	Deadline time.Time

	// LocalAddr is the local address to use when dialing an
	// address. The address must be of a compatible type for the
	// network being dialed.
	// If nil, a local address is automatically chosen.
	LocalAddr Addr

	// DualStack allows a single dial to attempt to establish
	// multiple IPv4 and IPv6 connections and to return the first
	// established connection when the network is "tcp" and the
	// destination is a host name that has multiple address family
	// DNS records.
	DualStack bool

	// KeepAlive specifies the keep-alive period for an active
	// network connection.
	// If zero, keep-alives are not enabled. Network protocols
	// that do not support keep-alives ignore this field.
	KeepAlive time.Duration
}

// dialSingle attempts to establish and returns a single connection to
// the destination address.
func dialSingle(net, addr string, la, ra Addr, deadline time.Time) (c Conn, err error) {
	if la != nil && la.Network() != ra.Network() {
		return nil, &OpError{Op: "dial", Net: net, Addr: ra, Err: errors.New("mismatched local address type " + la.Network())}
	}
	switch ra := ra.(type) {
	case *TCPAddr:
		la, _ := la.(*TCPAddr)
		c, err = dialTCP(net, la, ra, deadline)
	case *UDPAddr:
		la, _ := la.(*UDPAddr)
		c, err = dialUDP(net, la, ra, deadline)
	case *IPAddr:
		la, _ := la.(*IPAddr)
		c, err = dialIP(net, la, ra, deadline)
	case *UnixAddr:
		la, _ := la.(*UnixAddr)
		c, err = dialUnix(net, la, ra, deadline)
	default:
		return nil, &OpError{Op: "dial", Net: net, Addr: ra, Err: &AddrError{Err: "unexpected address type", Addr: addr}}
	}
	if err != nil {
		return nil, err // c is non-nil interface containing nil pointer
	}
	return c, nil
}

func dialTCP(net string, laddr, raddr *TCPAddr, deadline time.Time) (*TCPConn, error) {
	fd, err := internetSocket(net, laddr, raddr, deadline, syscall.SOCK_STREAM, 0, "dial", sockaddrToTCP)

	// TCP has a rarely used mechanism called a 'simultaneous connection' in
	// which Dial("tcp", addr1, addr2) run on the machine at addr1 can
	// connect to a simultaneous Dial("tcp", addr2, addr1) run on the machine
	// at addr2, without either machine executing Listen.  If laddr == nil,
	// it means we want the kernel to pick an appropriate originating local
	// address.  Some Linux kernels cycle blindly through a fixed range of
	// local ports, regardless of destination port.  If a kernel happens to
	// pick local port 50001 as the source for a Dial("tcp", "", "localhost:50001"),
	// then the Dial will succeed, having simultaneously connected to itself.
	// This can only happen when we are letting the kernel pick a port (laddr == nil)
	// and when there is no listener for the destination address.
	// It's hard to argue this is anything other than a kernel bug.  If we
	// see this happen, rather than expose the buggy effect to users, we
	// close the fd and try again.  If it happens twice more, we relent and
	// use the result.  See also:
	//	http://golang.org/issue/2690
	//	http://stackoverflow.com/questions/4949858/
	//
	// The opposite can also happen: if we ask the kernel to pick an appropriate
	// originating local address, sometimes it picks one that is already in use.
	// So if the error is EADDRNOTAVAIL, we have to try again too, just for
	// a different reason.
	//
	// The kernel socket code is no doubt enjoying watching us squirm.
	for i := 0; i < 2 && (laddr == nil || laddr.Port == 0) && (selfConnect(fd, err) || spuriousENOTAVAIL(err)); i++ {
		if err == nil {
			fd.Close()
		}
		fd, err = internetSocket(net, laddr, raddr, deadline, syscall.SOCK_STREAM, 0, "dial", sockaddrToTCP)
	}

	if err != nil {
		return nil, &OpError{Op: "dial", Net: net, Addr: raddr, Err: err}
	}
	return newTCPConn(fd), nil
}

// Internet sockets (TCP, UDP, IP)

func internetSocket(net string, laddr, raddr sockaddr, deadline time.Time, sotype, proto int, mode string, toAddr func(syscall.Sockaddr) Addr) (fd *netFD, err error) {
	family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
	return socket(net, family, sotype, proto, ipv6only, laddr, raddr, deadline, toAddr)
}

// socket returns a network file descriptor that is ready for
// asynchronous I/O using the network poller.
func socket(net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, deadline time.Time, toAddr func(syscall.Sockaddr) Addr) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		closesocket(s)
		return nil, err
	}
	if fd, err = newFD(s, family, sotype, net); err != nil {
		closesocket(s)
		return nil, err
	}

	// This function makes a network file descriptor for the
	// following applications:
	//
	// - An endpoint holder that opens a passive stream
	//   connenction, known as a stream listener
	//
	// - An endpoint holder that opens a destination-unspecific
	//   datagram connection, known as a datagram listener
	//
	// - An endpoint holder that opens an active stream or a
	//   destination-specific datagram connection, known as a
	//   dialer
	//
	// - An endpoint holder that opens the other connection, such
	//   as talking to the protocol stack inside the kernel
	//
	// For stream and datagram listeners, they will only require
	// named sockets, so we can assume that it's just a request
	// from stream or datagram listeners when laddr is not nil but
	// raddr is nil. Otherwise we assume it's just for dialers or
	// the other connection holders.

	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(laddr, listenerBacklog, toAddr); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr, toAddr); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}
	if err := fd.dial(laddr, raddr, deadline, toAddr); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}

func newFD(sysfd, family, sotype int, net string) (*netFD, error) {
	return &netFD{sysfd: sysfd, family: family, sotype: sotype, net: net}, nil
}

// Network file descriptor.
type netFD struct {
	// locking/lifetime of sysfd + serialize access to Read and Write methods
	fdmu fdMutex

	// immutable until Close
	sysfd       int
	family      int
	sotype      int
	isConnected bool
	net         string
	laddr       Addr
	raddr       Addr

	// wait server
	pd pollDesc
}

struct PollDesc
{
	PollDesc* link;	// in pollcache, protected by pollcache.Lock

	// The lock protects pollOpen, pollSetDeadline, pollUnblock and deadlineimpl operations.
	// This fully covers seq, rt and wt variables. fd is constant throughout the PollDesc lifetime.
	// pollReset, pollWait, pollWaitCanceled and runtime·netpollready (IO rediness notification)
	// proceed w/o taking the lock. So closing, rg, rd, wg and wd are manipulated
	// in a lock-free way by all operations.
	Lock;		// protectes the following fields
	uintptr	fd;
	bool	closing;
	uintptr	seq;	// protects from stale timers and ready notifications
	G*	rg;	// READY, WAIT, G waiting for read or nil
	Timer	rt;	// read deadline timer (set if rt.fv != nil)
	int64	rd;	// read deadline
	G*	wg;	// READY, WAIT, G waiting for write or nil
	Timer	wt;	// write deadline timer
	int64	wd;	// write deadline
	void*	user;	// user settable cookie
};

func (fd *netFD) init() error {
	if err := fd.pd.Init(fd); err != nil {
		return err
	}
	return nil
}

func (pd *pollDesc) Init(fd *netFD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.sysfd))
	if errno != 0 {
		return syscall.Errno(errno)
	}
	pd.runtimeCtx = ctx
	return nil
}

func (fd *netFD) dial(laddr, raddr sockaddr, deadline time.Time, toAddr func(syscall.Sockaddr) Addr) error {
	var err error
	var lsa syscall.Sockaddr
	if laddr != nil {
		if lsa, err = laddr.sockaddr(fd.family); err != nil {
			return err
		} else if lsa != nil {
			if err := syscall.Bind(fd.sysfd, lsa); err != nil {
				return os.NewSyscallError("bind", err)
			}
		}
	}
	var rsa syscall.Sockaddr
	if raddr != nil {
		if rsa, err = raddr.sockaddr(fd.family); err != nil {
			return err
		}
		if err := fd.connect(lsa, rsa, deadline); err != nil {
			return err
		}
		fd.isConnected = true
	} else {
		if err := fd.init(); err != nil {
			return err
		}
	}
	lsa, _ = syscall.Getsockname(fd.sysfd)
	if rsa, _ = syscall.Getpeername(fd.sysfd); rsa != nil {
		fd.setAddr(toAddr(lsa), toAddr(rsa))
	} else {
		fd.setAddr(toAddr(lsa), raddr)
	}
	return nil
}

func newTCPConn(fd *netFD) *TCPConn {
	c := &TCPConn{conn{fd}}
	c.SetNoDelay(true)
	return c
}

type conn struct {
	fd *netFD
}

// TCPConn is an implementation of the Conn interface for TCP network
// connections.
type TCPConn struct {
	conn
}

func dial(network string, ra Addr, dialer func(time.Time) (Conn, error), deadline time.Time) (Conn, error) {
	return dialer(deadline)
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

Docker tcp connection结构关系如下：

![](/public/img/golang_client_flow/conn_struct.png)

`socket`创建tcp socket，设置socket属性，创建netFD，并执行connect动作，最终返回一个可正常使用的tcp socket（正常完成tcp三次握手），如下：

```go
// socket returns a network file descriptor that is ready for
// asynchronous I/O using the network poller.
func socket(net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, deadline time.Time, toAddr func(syscall.Sockaddr) Addr) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		closesocket(s)
		return nil, err
	}
	if fd, err = newFD(s, family, sotype, net); err != nil {
		closesocket(s)
		return nil, err
	}

	// This function makes a network file descriptor for the
	// following applications:
	//
	// - An endpoint holder that opens a passive stream
	//   connenction, known as a stream listener
	//
	// - An endpoint holder that opens a destination-unspecific
	//   datagram connection, known as a datagram listener
	//
	// - An endpoint holder that opens an active stream or a
	//   destination-specific datagram connection, known as a
	//   dialer
	//
	// - An endpoint holder that opens the other connection, such
	//   as talking to the protocol stack inside the kernel
	//
	// For stream and datagram listeners, they will only require
	// named sockets, so we can assume that it's just a request
	// from stream or datagram listeners when laddr is not nil but
	// raddr is nil. Otherwise we assume it's just for dialers or
	// the other connection holders.

	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(laddr, listenerBacklog, toAddr); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr, toAddr); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}
	if err := fd.dial(laddr, raddr, deadline, toAddr); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}
```

`sysSocket`创建tcp socket，如下：

```go
// Wrapper around the socket system call that marks the returned file
// descriptor as nonblocking and close-on-exec.
func sysSocket(family, sotype, proto int) (int, error) {
	s, err := syscall.Socket(family, sotype|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC, proto)
	// On Linux the SOCK_NONBLOCK and SOCK_CLOEXEC flags were
	// introduced in 2.6.27 kernel and on FreeBSD both flags were
	// introduced in 10 kernel. If we get an EINVAL error on Linux
	// or EPROTONOSUPPORT error on FreeBSD, fall back to using
	// socket without them.
	if err == nil || (err != syscall.EPROTONOSUPPORT && err != syscall.EINVAL) {
		return s, err
	}

	// See ../syscall/exec_unix.go for description of ForkLock.
	syscall.ForkLock.RLock()
	s, err = syscall.Socket(family, sotype, proto)
	if err == nil {
		syscall.CloseOnExec(s)
	}
	syscall.ForkLock.RUnlock()
	if err != nil {
		return -1, err
	}
	if err = syscall.SetNonblock(s, true); err != nil {
		syscall.Close(s)
		return -1, err
	}
	return s, nil
}
```

`setDefaultSockopts`设置socket属性，如下：

```go
func setDefaultSockopts(s, family, sotype int, ipv6only bool) error {
	if family == syscall.AF_INET6 && sotype != syscall.SOCK_RAW {
		// Allow both IP versions even if the OS default
		// is otherwise.  Note that some operating systems
		// never admit this option.
		syscall.SetsockoptInt(s, syscall.IPPROTO_IPV6, syscall.IPV6_V6ONLY, boolint(ipv6only))
	}
	// Allow broadcast.
	return os.NewSyscallError("setsockopt", syscall.SetsockoptInt(s, syscall.SOL_SOCKET, syscall.SO_BROADCAST, 1))
}
```

`newFD`根据`network file descriptor`创建netFD，如下:

```go
func newFD(sysfd, family, sotype int, net string) (*netFD, error) {
	return &netFD{sysfd: sysfd, family: family, sotype: sotype, net: net}, nil
}
```

`dial`执行connect动作，如下：

```go
func (fd *netFD) dial(laddr, raddr sockaddr, deadline time.Time, toAddr func(syscall.Sockaddr) Addr) error {
	var err error
	var lsa syscall.Sockaddr
	if laddr != nil {
		if lsa, err = laddr.sockaddr(fd.family); err != nil {
			return err
		} else if lsa != nil {
			if err := syscall.Bind(fd.sysfd, lsa); err != nil {
				return os.NewSyscallError("bind", err)
			}
		}
	}
	var rsa syscall.Sockaddr
	if raddr != nil {
		if rsa, err = raddr.sockaddr(fd.family); err != nil {
			return err
		}
		if err := fd.connect(lsa, rsa, deadline); err != nil {
			return err
		}
		fd.isConnected = true
	} else {
		if err := fd.init(); err != nil {
			return err
		}
	}
	lsa, _ = syscall.Getsockname(fd.sysfd)
	if rsa, _ = syscall.Getpeername(fd.sysfd); rsa != nil {
		fd.setAddr(toAddr(lsa), toAddr(rsa))
	} else {
		fd.setAddr(toAddr(lsa), raddr)
	}
	return nil
}

func (fd *netFD) connect(la, ra syscall.Sockaddr, deadline time.Time) error {
	// Do not need to call fd.writeLock here,
	// because fd is not yet accessible to user,
	// so no concurrent operations are possible.
	switch err := syscall.Connect(fd.sysfd, ra); err {
	case syscall.EINPROGRESS, syscall.EALREADY, syscall.EINTR:
	case nil, syscall.EISCONN:
		if !deadline.IsZero() && deadline.Before(time.Now()) {
			return errTimeout
		}
		if err := fd.init(); err != nil {
			return err
		}
		return nil
	case syscall.EINVAL:
		// On Solaris we can see EINVAL if the socket has
		// already been accepted and closed by the server.
		// Treat this as a successful connection--writes to
		// the socket will see EOF.  For details and a test
		// case in C see http://golang.org/issue/6828.
		if runtime.GOOS == "solaris" {
			return nil
		}
		fallthrough
	default:
		return err
	}
	if err := fd.init(); err != nil {
		return err
	}
	if !deadline.IsZero() {
		fd.setWriteDeadline(deadline)
		defer fd.setWriteDeadline(noDeadline)
	}
	for {
		// Performing multiple connect system calls on a
		// non-blocking socket under Unix variants does not
		// necessarily result in earlier errors being
		// returned. Instead, once runtime-integrated network
		// poller tells us that the socket is ready, get the
		// SO_ERROR socket option to see if the connection
		// succeeded or failed. See issue 7474 for further
		// details.
		if err := fd.pd.WaitWrite(); err != nil {
			return err
		}
		nerr, err := syscall.GetsockoptInt(fd.sysfd, syscall.SOL_SOCKET, syscall.SO_ERROR)
		if err != nil {
			return err
		}
		switch err := syscall.Errno(nerr); err {
		case syscall.EINPROGRESS, syscall.EALREADY, syscall.EINTR:
		case syscall.Errno(0), syscall.EISCONN:
			return nil
		default:
			return err
		}
	}
}
```

在`connect`执行成功后，执行`fd.init()`初始化操作，将`pd pollDesc`结构初始化，如下：

```go
// Network file descriptor.
type netFD struct {
	// locking/lifetime of sysfd + serialize access to Read and Write methods
	fdmu fdMutex

	// immutable until Close
	sysfd       int
	family      int
	sotype      int
	isConnected bool
	net         string
	laddr       Addr
	raddr       Addr

	// wait server
	pd pollDesc
}

func (fd *netFD) init() error {
	if err := fd.pd.Init(fd); err != nil {
		return err
	}
	return nil
}

func (pd *pollDesc) Init(fd *netFD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.sysfd))
	if errno != 0 {
		return syscall.Errno(errno)
	}
	pd.runtimeCtx = ctx
	return nil
}

// Once is an object that will perform exactly one action.
type Once struct {
	m    Mutex
	done uint32
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once.  In other words, given
// 	var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation.  A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once.  Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
// 	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 1 {
		return
	}
	// Slow-path.
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		f()
		atomic.StoreUint32(&o.done, 1)
	}
}

func runtime_pollServerInit() {//只执行一次
	runtime·netpollinit();
}

void
runtime·netpollinit(void) //创建epoll
{
	epfd = runtime·epollcreate1(EPOLL_CLOEXEC);
	if(epfd >= 0)
		return;
	epfd = runtime·epollcreate(1024);
	if(epfd >= 0) {
		runtime·closeonexec(epfd);
		return;
	}
	runtime·printf("netpollinit: failed to create descriptor (%d)\n", -epfd);
	runtime·throw("netpollinit: failed to create descriptor");
}

func runtime_pollOpen(fd uintptr) (pd *PollDesc, errno int) {
	pd = allocPollDesc();
	runtime·lock(pd);
	if(pd->wg != nil && pd->wg != READY)
		runtime·throw("runtime_pollOpen: blocked write on free descriptor");
	if(pd->rg != nil && pd->rg != READY)
		runtime·throw("runtime_pollOpen: blocked read on free descriptor");
	pd->fd = fd;
	pd->closing = false;
	pd->seq++;
	pd->rg = nil;
	pd->rd = 0;
	pd->wg = nil;
	pd->wd = 0;
	runtime·unlock(pd);

	errno = runtime·netpollopen(fd, pd);
}

static PollDesc*
allocPollDesc(void)
{
	PollDesc *pd;
	uint32 i, n;

	runtime·lock(&pollcache);
	if(pollcache.first == nil) {
		n = PollBlockSize/sizeof(*pd);
		if(n == 0)
			n = 1;
		// Must be in non-GC memory because can be referenced
		// only from epoll/kqueue internals.
		pd = runtime·persistentalloc(n*sizeof(*pd), 0, &mstats.other_sys);
		for(i = 0; i < n; i++) {
			pd[i].link = pollcache.first;
			pollcache.first = &pd[i];
		}
	}
	pd = pollcache.first;
	pollcache.first = pd->link;
	runtime·unlock(&pollcache);
	return pd;
}

struct PollDesc
{
	PollDesc* link;	// in pollcache, protected by pollcache.Lock

	// The lock protects pollOpen, pollSetDeadline, pollUnblock and deadlineimpl operations.
	// This fully covers seq, rt and wt variables. fd is constant throughout the PollDesc lifetime.
	// pollReset, pollWait, pollWaitCanceled and runtime·netpollready (IO rediness notification)
	// proceed w/o taking the lock. So closing, rg, rd, wg and wd are manipulated
	// in a lock-free way by all operations.
	Lock;		// protectes the following fields
	uintptr	fd;
	bool	closing;
	uintptr	seq;	// protects from stale timers and ready notifications
	G*	rg;	// READY, WAIT, G waiting for read or nil
	Timer	rt;	// read deadline timer (set if rt.fv != nil)
	int64	rd;	// read deadline
	G*	wg;	// READY, WAIT, G waiting for write or nil
	Timer	wt;	// write deadline timer
	int64	wd;	// write deadline
	void*	user;	// user settable cookie
};

//src/pkg/runtime/netpoll_epoll.c
int32
runtime·netpollopen(uintptr fd, PollDesc *pd)
{
	EpollEvent ev;
	int32 res;

	ev.events = EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET;
	ev.data = (uint64)pd;
	res = runtime·epollctl(epfd, EPOLL_CTL_ADD, (int32)fd, &ev); //添加到epoll中
	return -res;
}
```

也即创建epoll，初始化`pd pollDesc`结构，并将`network file descriptor`添加到epoll中

`sysmon main goroutine`会定期检查epoll事件，并将READY的G加入到全局队列中，如下：

```go
//src/pkg/runtime/proc.c
// The main goroutine.
// Note: C frames in general are not copyable during stack growth, for two reasons:
//   1) We don't know where in a frame to find pointers to other stack locations.
//   2) There's no guarantee that globals or heap values do not point into the frame.
//
// The C frame for runtime.main is copyable, because:
//   1) There are no pointers to other stack locations in the frame
//      (d.fn points at a global, d.link is nil, d.argp is -1).
//   2) The only pointer into this frame is from the defer chain,
//      which is explicitly handled during stack copying.
void
runtime·main(void)
{
	Defer d;
	
	// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
	// Using decimal instead of binary GB and MB because
	// they look nicer in the stack overflow failure message.
	if(sizeof(void*) == 8)
		runtime·maxstacksize = 1000000000;
	else
		runtime·maxstacksize = 250000000;

	newm(sysmon, nil);

	// Lock the main goroutine onto this, the main OS thread,
	// during initialization.  Most programs won't care, but a few
	// do require certain calls to be made by the main thread.
	// Those can arrange for main.main to run in the main thread
	// by calling runtime.LockOSThread during initialization
	// to preserve the lock.
	runtime·lockOSThread();
	
	// Defer unlock so that runtime.Goexit during init does the unlock too.
	d.fn = &initDone;
	d.siz = 0;
	d.link = g->defer;
	d.argp = NoArgs;
	d.special = true;
	g->defer = &d;

	if(m != &runtime·m0)
		runtime·throw("runtime·main not on m0");
	runtime·newproc1(&scavenger, nil, 0, 0, runtime·main);
	main·init();

	if(g->defer != &d || d.fn != &initDone)
		runtime·throw("runtime: bad defer entry after init");
	g->defer = d.link;
	runtime·unlockOSThread();

	main·main();
	if(raceenabled)
		runtime·racefini();

	// Make racy client program work: if panicking on
	// another goroutine at the same time as main returns,
	// let the other goroutine finish printing the panic trace.
	// Once it does, it will exit. See issue 3934.
	if(runtime·panicking)
		runtime·park(nil, nil, "panicwait");

	runtime·exit(0);
	for(;;)
		*(int32*)runtime·main = 0;
}

static void
sysmon(void)
{
	uint32 idle, delay;
	int64 now, lastpoll, lasttrace;
	G *gp;

	lasttrace = 0;
	idle = 0;  // how many cycles in succession we had not wokeup somebody
	delay = 0;
	for(;;) {
		if(idle == 0)  // start with 20us sleep...
			delay = 20;
		else if(idle > 50)  // start doubling the sleep after 1ms...
			delay *= 2;
		if(delay > 10*1000)  // up to 10ms
			delay = 10*1000;
		runtime·usleep(delay);
		if(runtime·debug.schedtrace <= 0 &&
			(runtime·sched.gcwaiting || runtime·atomicload(&runtime·sched.npidle) == runtime·gomaxprocs)) {  // TODO: fast atomic
			runtime·lock(&runtime·sched);
			if(runtime·atomicload(&runtime·sched.gcwaiting) || runtime·atomicload(&runtime·sched.npidle) == runtime·gomaxprocs) {
				runtime·atomicstore(&runtime·sched.sysmonwait, 1);
				runtime·unlock(&runtime·sched);
				runtime·notesleep(&runtime·sched.sysmonnote);
				runtime·noteclear(&runtime·sched.sysmonnote);
				idle = 0;
				delay = 20;
			} else
				runtime·unlock(&runtime·sched);
		}
		// poll network if not polled for more than 10ms
		lastpoll = runtime·atomicload64(&runtime·sched.lastpoll);
		now = runtime·nanotime();
		if(lastpoll != 0 && lastpoll + 10*1000*1000 < now) {
			runtime·cas64(&runtime·sched.lastpoll, lastpoll, now);
			gp = runtime·netpoll(false);  // non-blocking
			if(gp) {
				// Need to decrement number of idle locked M's
				// (pretending that one more is running) before injectglist.
				// Otherwise it can lead to the following situation:
				// injectglist grabs all P's but before it starts M's to run the P's,
				// another M returns from syscall, finishes running its G,
				// observes that there is no work to do and no other running M's
				// and reports deadlock.
				incidlelocked(-1);
				injectglist(gp); //Put gp on the global runnable queue
				incidlelocked(1);
			}
		}
		// retake P's blocked in syscalls
		// and preempt long running G's
		if(retake(now))
			idle = 0;
		else
			idle++;

		if(runtime·debug.schedtrace > 0 && lasttrace + runtime·debug.schedtrace*1000000ll <= now) {
			lasttrace = now;
			runtime·schedtrace(runtime·debug.scheddetail);
		}
	}
}

static int32 epfd = -1;  // epoll descriptor

void
runtime·netpollinit(void)
{
	epfd = runtime·epollcreate1(EPOLL_CLOEXEC);
	if(epfd >= 0)
		return;
	epfd = runtime·epollcreate(1024);
	if(epfd >= 0) {
		runtime·closeonexec(epfd);
		return;
	}
	runtime·printf("netpollinit: failed to create descriptor (%d)\n", -epfd);
	runtime·throw("netpollinit: failed to create descriptor");
}

// polls for ready network connections
// returns list of goroutines that become runnable
G*
runtime·netpoll(bool block)
{
	static int32 lasterr;
	EpollEvent events[128], *ev;
	int32 n, i, waitms, mode;
	G *gp;

	if(epfd == -1)
		return nil;
	waitms = -1;
	if(!block)
		waitms = 0;
retry:
	n = runtime·epollwait(epfd, events, nelem(events), waitms);
	if(n < 0) {
		if(n != -EINTR && n != lasterr) {
			lasterr = n;
			runtime·printf("runtime: epollwait on fd %d failed with %d\n", epfd, -n);
		}
		goto retry;
	}
	gp = nil;
	for(i = 0; i < n; i++) {
		ev = &events[i];
		if(ev->events == 0)
			continue;
		mode = 0;
		if(ev->events & (EPOLLIN|EPOLLRDHUP|EPOLLHUP|EPOLLERR))
			mode += 'r';
		if(ev->events & (EPOLLOUT|EPOLLHUP|EPOLLERR))
			mode += 'w';
		if(mode)
			runtime·netpollready(&gp, (void*)ev->data, mode);
	}
	if(block && gp == nil)
		goto retry;
	return gp;
}

// make pd ready, newly runnable goroutines (if any) are enqueued info gpp list
void
runtime·netpollready(G **gpp, PollDesc *pd, int32 mode)
{
	G *rg, *wg;

	rg = wg = nil;
	if(mode == 'r' || mode == 'r'+'w')
		rg = netpollunblock(pd, 'r', true);
	if(mode == 'w' || mode == 'r'+'w')
		wg = netpollunblock(pd, 'w', true);
	if(rg) {
		rg->schedlink = *gpp;
		*gpp = rg;
	}
	if(wg) {
		wg->schedlink = *gpp;
		*gpp = wg;
	}
}

static G*
netpollunblock(PollDesc *pd, int32 mode, bool ioready)
{
	G **gpp, *old, *new;

	gpp = &pd->rg;
	if(mode == 'w')
		gpp = &pd->wg;

	for(;;) {
		old = *gpp;
		if(old == READY)
			return nil;
		if(old == nil && !ioready) {
			// Only set READY for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil;
		}
		new = nil;
		if(ioready)
			new = READY;
		if(runtime·casp(gpp, old, new))
			break;
	}
	if(old > WAIT)
		return old;  // must be G*
	return nil;
}

// Injects the list of runnable G's into the scheduler.
// Can run concurrently with GC.
static void
injectglist(G *glist)
{
	int32 n;
	G *gp;

	if(glist == nil)
		return;
	runtime·lock(&runtime·sched);
	for(n = 0; glist; n++) {
		gp = glist;
		glist = gp->schedlink;
		gp->status = Grunnable;
		globrunqput(gp);
	}
	runtime·unlock(&runtime·sched);

	for(; n && runtime·sched.npidle; n--)
		startm(nil, false);
}

// Put gp on the global runnable queue.
// Sched must be locked.
static void
globrunqput(G *gp)
{
	gp->schedlink = nil;
	if(runtime·sched.runqtail)
		runtime·sched.runqtail->schedlink = gp;
	else
		runtime·sched.runqhead = gp;
	runtime·sched.runqtail = gp;
	runtime·sched.runqsize++;
}
```

`SetReadDeadline`用于实现Go的网络IO超时原语，它会给netFD创建对应的IO定时器，当定时器超时(从`netFD.setReadDeadline`开始计时，也即从`TimeoutConn.Read`开始计时)，如果对应的goroutine处于等待的状态(默认情况下deadline为0，不会创建定时器)，runtime会调用runtime·ready唤醒对应进行Read/Write的goroutine，并产生`i/o timeout` error

Go并没有使用`epoll_wait`实现IO的超时，而是通过`Set[Read|Write]Deadline(time.Time)`对每个netFD设置超时

当`SetDeadline`设置的定时器超时后，在超时处理函数中，会删除该定时器；而且，每次收到或者发送数据时，也不会reset该定时器。所以，每次Read/Write操作之前，都需要调用该函数：

```go
// A net.Conn that sets a deadline for every Read or Write operation
type TimeoutConn struct {
	net.Conn
	timeout time.Duration
}
type conn struct {
	fd *netFD
}

// TCPConn is an implementation of the Conn interface for TCP network
// connections.
type TCPConn struct {
	conn
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

// SetReadDeadline implements the Conn SetReadDeadline method.
func (c *conn) SetReadDeadline(t time.Time) error {
	if !c.ok() {
		return syscall.EINVAL
	}
	return c.fd.setReadDeadline(t)
}

// file:/src/pkg/net/fd_poll_runtime.go
func (fd *netFD) setReadDeadline(t time.Time) error {
	return setDeadlineImpl(fd, t, 'r')
}

// file:/src/pkg/net/fd_poll_runtime.go
func setDeadlineImpl(fd *netFD, t time.Time, mode int) error {
	d := runtimeNano() + int64(t.Sub(time.Now()))
	if t.IsZero() {
		d = 0
	}
	if err := fd.incref(); err != nil {
		return err
	}
	runtime_pollSetDeadline(fd.pd.runtimeCtx, d, mode)
	fd.decref()
	return nil
}

func runtime_pollSetDeadline(pd *PollDesc, d int64, mode int) {
	G *rg, *wg;

	runtime·lock(pd);
	if(pd->closing) {
		runtime·unlock(pd);
		return;
	}
	pd->seq++;  // invalidate current timers
	// Reset current timers.
	if(pd->rt.fv) {
		runtime·deltimer(&pd->rt);
		pd->rt.fv = nil;
	}
	if(pd->wt.fv) {
		runtime·deltimer(&pd->wt);
		pd->wt.fv = nil;
	}
	// Setup new timers.
	if(d != 0 && d <= runtime·nanotime())
		d = -1;
	if(mode == 'r' || mode == 'r'+'w')
		pd->rd = d;
	if(mode == 'w' || mode == 'r'+'w')
		pd->wd = d;
	if(pd->rd > 0 && pd->rd == pd->wd) {
		pd->rt.fv = &deadlineFn;
		pd->rt.when = pd->rd;
		// Copy current seq into the timer arg.
		// Timer func will check the seq against current descriptor seq,
		// if they differ the descriptor was reused or timers were reset.
		pd->rt.arg.type = (Type*)pd->seq;
		pd->rt.arg.data = pd;
		runtime·addtimer(&pd->rt);
	} else {
		if(pd->rd > 0) {
			pd->rt.fv = &readDeadlineFn;
			pd->rt.when = pd->rd;
			pd->rt.arg.type = (Type*)pd->seq;
			pd->rt.arg.data = pd;
			runtime·addtimer(&pd->rt);
		}
		if(pd->wd > 0) {
			pd->wt.fv = &writeDeadlineFn;
			pd->wt.when = pd->wd;
			pd->wt.arg.type = (Type*)pd->seq;
			pd->wt.arg.data = pd;
			runtime·addtimer(&pd->wt);
		}
	}
	// If we set the new deadline in the past, unblock currently pending IO if any.
	rg = nil;
	runtime·atomicstorep(&wg, nil);  // full memory barrier between stores to rd/wd and load of rg/wg in netpollunblock
	if(pd->rd < 0)
		rg = netpollunblock(pd, 'r', false);
	if(pd->wd < 0)
		wg = netpollunblock(pd, 'w', false);
	runtime·unlock(pd);
	if(rg)
		runtime·ready(rg);
	if(wg)
		runtime·ready(wg);
}

// Package time knows the layout of this structure.
// If this struct changes, adjust ../time/sleep.go:/runtimeTimer.
// For GOOS=nacl, package syscall knows the layout of this structure.
// If this struct changes, adjust ../syscall/net_nacl.go:/runtimeTimer.
struct	Timer
{
	int32	i;	// heap index

	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(now, arg) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	int64	when;
	int64	period;
	FuncVal	*fv;
	Eface	arg;
};

static void
readDeadline(int64 now, Eface arg)
{
	deadlineimpl(now, arg, true, false);
}

static void
deadlineimpl(int64 now, Eface arg, bool read, bool write)
{
	PollDesc *pd;
	uint32 seq;
	G *rg, *wg;

	USED(now);
	pd = (PollDesc*)arg.data;
	// This is the seq when the timer was set.
	// If it's stale, ignore the timer event.
	seq = (uintptr)arg.type;
	rg = wg = nil;
	runtime·lock(pd);
	if(seq != pd->seq) {
		// The descriptor was reused or timers were reset.
		runtime·unlock(pd);
		return;
	}
	if(read) {
		if(pd->rd <= 0 || pd->rt.fv == nil)
			runtime·throw("deadlineimpl: inconsistent read deadline");
		pd->rd = -1;
		runtime·atomicstorep(&pd->rt.fv, nil);  // full memory barrier between store to rd and load of rg in netpollunblock
		rg = netpollunblock(pd, 'r', false);
	}
	if(write) {
		if(pd->wd <= 0 || (pd->wt.fv == nil && !read))
			runtime·throw("deadlineimpl: inconsistent write deadline");
		pd->wd = -1;
		runtime·atomicstorep(&pd->wt.fv, nil);  // full memory barrier between store to wd and load of wg in netpollunblock
		wg = netpollunblock(pd, 'w', false);
	}
	runtime·unlock(pd);
	if(rg)
		runtime·ready(rg);
	if(wg)
		runtime·ready(wg);
}

static G*
netpollunblock(PollDesc *pd, int32 mode, bool ioready)
{
	G **gpp, *old, *new;

	gpp = &pd->rg;
	if(mode == 'w')
		gpp = &pd->wg;

	for(;;) {
		old = *gpp;
		if(old == READY)
			return nil;
		if(old == nil && !ioready) {
			// Only set READY for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil;
		}
		new = nil;
		if(ioready)
			new = READY;
		if(runtime·casp(gpp, old, new))
			break;
	}
	if(old > WAIT)
		return old;  // must be G*
	return nil;
}

// Mark gp ready to run.
void
runtime·ready(G *gp)
{
	// Mark runnable.
	m->locks++;  // disable preemption because it can be holding p in a local var
	if(gp->status != Gwaiting) {
		runtime·printf("goroutine %D has status %d\n", gp->goid, gp->status);
		runtime·throw("bad g->status in ready");
	}
	gp->status = Grunnable;
	runqput(m->p, gp);
	if(runtime·atomicload(&runtime·sched.npidle) != 0 && runtime·atomicload(&runtime·sched.nmspinning) == 0)  // TODO: fast atomic
		wakep();
	m->locks--;
	if(m->locks == 0 && g->preempt)  // restore the preemption request in case we've cleared it in newstack
		g->stackguard0 = StackPreempt;
}

// Tries to add one more P to execute G's.
// Called when a G is made runnable (newproc, ready).
static void
wakep(void)
{
	// be conservative about spinning threads
	if(!runtime·cas(&runtime·sched.nmspinning, 0, 1))
		return;
	startm(nil, true);
}

// Read implements the Conn Read method.
func (c *conn) Read(b []byte) (int, error) {
	if !c.ok() {
		return 0, syscall.EINVAL
	}
	return c.fd.Read(b)
}

func (fd *netFD) Read(p []byte) (n int, err error) {
	if err := fd.readLock(); err != nil {
		return 0, err
	}
	defer fd.readUnlock()
	if err := fd.pd.PrepareRead(); err != nil {
		return 0, &OpError{"read", fd.net, fd.raddr, err}
	}
	for {
		n, err = syscall.Read(int(fd.sysfd), p)
		if err != nil {
			n = 0
			if err == syscall.EAGAIN {
				if err = fd.pd.WaitRead(); err == nil {
					continue
				}
			}
		}
		err = chkReadErr(n, err, fd)
		break
	}
	if err != nil && err != io.EOF {
		err = &OpError{"read", fd.net, fd.raddr, err}
	}
	return
}

// Add a reference to this fd and lock for reading.
// Returns an error if the fd cannot be used.
func (fd *netFD) readLock() error {
	if !fd.fdmu.RWLock(true) {
		return errClosing
	}
	return nil
}

func (pd *pollDesc) PrepareRead() error {
	return pd.Prepare('r')
}

func (pd *pollDesc) Prepare(mode int) error {
	res := runtime_pollReset(pd.runtimeCtx, mode)
	return convertErr(res)
}

func runtime_pollReset(pd *PollDesc, mode int) (err int) {
	err = checkerr(pd, mode);
	if(err)
		goto ret;
	if(mode == 'r')
		pd->rg = nil;
	else if(mode == 'w')
		pd->wg = nil;
ret:
}

static intgo
checkerr(PollDesc *pd, int32 mode)
{
	if(pd->closing)
		return 1;  // errClosing
	if((mode == 'r' && pd->rd < 0) || (mode == 'w' && pd->wd < 0))
		return 2;  // errTimeout
	return 0;
}

// Various errors contained in OpError.
var (
	// For connection setup and write operations.
	errMissingAddress = errors.New("missing address")

	// For both read and write operations.
	errTimeout          error = &timeoutError{}
	errClosing                = errors.New("use of closed network connection")
	ErrWriteToConnected       = errors.New("use of WriteTo with pre-connected connection")
)

func convertErr(res int) error {
	switch res {
	case 0:
		return nil
	case 1:
		return errClosing
	case 2:
		return errTimeout
	}
	println("unreachable: ", res)
	panic("unreachable")
}

func Read(fd int, p []byte) (n int, err error) {
	n, err = read(fd, p)
	if raceenabled {
		if n > 0 {
			raceWriteRange(unsafe.Pointer(&p[0]), n)
		}
		if err == nil {
			raceAcquire(unsafe.Pointer(&ioSync))
		}
	}
	return
}

func (pd *pollDesc) WaitRead() error {
	return pd.Wait('r')
}

func (pd *pollDesc) Wait(mode int) error {
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res)
}

func runtime_pollWait(pd *PollDesc, mode int) (err int) {
	err = checkerr(pd, mode);
	if(err == 0) {
		// As for now only Solaris uses level-triggered IO.
		if(Solaris)
			runtime·netpollarm(pd, mode);
		while(!netpollblock(pd, mode, false)) {
			err = checkerr(pd, mode);
			if(err != 0)
				break;
			// Can happen if timeout has fired and unblocked us,
			// but before we had a chance to run, timeout has been reset.
			// Pretend it has not happened and retry.
		}
	}
}

// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
static bool
netpollblock(PollDesc *pd, int32 mode, bool waitio)
{
	G **gpp, *old;

	gpp = &pd->rg;
	if(mode == 'w')
		gpp = &pd->wg;

	// set the gpp semaphore to WAIT
	for(;;) {
		old = *gpp;
		if(old == READY) {
			*gpp = nil;
			return true;
		}
		if(old != nil)
			runtime·throw("netpollblock: double wait");
		if(runtime·casp(gpp, nil, WAIT))
			break;
	}

	// need to recheck error states after setting gpp to WAIT
	// this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
	// do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
	if(waitio || checkerr(pd, mode) == 0)
		runtime·park((bool(*)(G*, void*))blockcommit, gpp, "IO wait"); //Puts the current goroutine into a waiting state
	// be careful to not lose concurrent READY notification
	old = runtime·xchgp(gpp, nil);
	if(old > WAIT)
		runtime·throw("netpollblock: corrupted state");
	return old == READY;
}

// Puts the current goroutine into a waiting state and calls unlockf.
// If unlockf returns false, the goroutine is resumed.
void
runtime·park(bool(*unlockf)(G*, void*), void *lock, int8 *reason)
{
	if(g->status != Grunning)
		runtime·throw("bad g status");
	m->waitlock = lock;
	m->waitunlockf = unlockf;
	g->waitreason = reason;
	runtime·mcall(park0);
}

// runtime·park continuation on g0.
static void
park0(G *gp)
{
	bool ok;

	gp->status = Gwaiting;
	gp->m = nil;
	m->curg = nil;
	if(m->waitunlockf) {
		ok = m->waitunlockf(gp, m->waitlock);
		m->waitunlockf = nil;
		m->waitlock = nil;
		if(!ok) {
			gp->status = Grunnable;
			execute(gp);  // Schedule it back, never returns.
		}
	}
	if(m->lockedg) {
		stoplockedm();
		execute(gp);  // Never returns.
	}
	schedule();
}

// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
static void
schedule(void)
{
	G *gp;
	uint32 tick;

	if(m->locks)
		runtime·throw("schedule: holding locks");

top:
	if(runtime·sched.gcwaiting) {
		gcstopm();
		goto top;
	}

	gp = nil;
	// Check the global runnable queue once in a while to ensure fairness.
	// Otherwise two goroutines can completely occupy the local runqueue
	// by constantly respawning each other.
	tick = m->p->schedtick;
	// This is a fancy way to say tick%61==0,
	// it uses 2 MUL instructions instead of a single DIV and so is faster on modern processors.
	if(tick - (((uint64)tick*0x4325c53fu)>>36)*61 == 0 && runtime·sched.runqsize > 0) {
		runtime·lock(&runtime·sched);
		gp = globrunqget(m->p, 1);
		runtime·unlock(&runtime·sched);
		if(gp)
			resetspinning();
	}
	if(gp == nil) {
		gp = runqget(m->p);
		if(gp && m->spinning)
			runtime·throw("schedule: spinning with local work");
	}
	if(gp == nil) {
		gp = findrunnable();  // blocks until work is available
		resetspinning();
	}

	if(gp->lockedm) {
		// Hands off own p to the locked m,
		// then blocks waiting for a new p.
		startlockedm(gp);
		goto top;
	}

	execute(gp);
}

func chkReadErr(n int, err error, fd *netFD) error {
	if n == 0 && err == nil && fd.sotype != syscall.SOCK_DGRAM && fd.sotype != syscall.SOCK_RAW {
		return io.EOF
	}
	return err
}

// OpError is the error type usually returned by functions in the net
// package. It describes the operation, network type, and address of
// an error.
type OpError struct {
	// Op is the operation which caused the error, such as
	// "read" or "write".
	Op string

	// Net is the network type on which this error occurred,
	// such as "tcp" or "udp6".
	Net string

	// Addr is the network address on which this error occurred.
	Addr Addr

	// Err is the error that occurred during the operation.
	Err error
}

func (e *OpError) Error() string {
	if e == nil {
		return "<nil>"
	}
	s := e.Op
	if e.Net != "" {
		s += " " + e.Net
	}
	if e.Addr != nil {
		s += " " + e.Addr.String()
	}
	s += ": " + e.Err.Error()
	return s
}

func (e *timeoutError) Error() string   { return "i/o timeout" }
```

测试代码：

server端代码：

```go
package main

import(
    "time"
    "fmt"
    "net/http"
)

type Server struct {
}

var server Server

func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Println(r.RequestURI)
    time.Sleep(time.Minute*3)
    fmt.Println("here")
    http.Error(w, fmt.Sprintf("Done for request!"), http.StatusOK)
}

func main() {
    // http server
    err := http.ListenAndServe("xxx", &server)
    fmt.Printf("err:%s", err)
}
```

client端代码：

```go
package main

import (
        "net"
        "time"
        "net/http"
        "fmt"
)

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

func main(){
        req, err := http.NewRequest("GET", "http://xxx/example.com", nil)
        httpTransport := &http.Transport{
                DisableKeepAlives: true,
                Proxy:             http.ProxyFromEnvironment,
        }
        httpTransport.Dial = func(proto string, addr string) (net.Conn, error) {
                conn, err := net.Dial(proto, addr)
                if err != nil {
                        return nil, err
                }
                conn = NewTimeoutConn(conn, 1*time.Minute)
                return conn, nil
        }
        client := &http.Client{
                Transport: httpTransport,
        }
        res, err := client.Do(req)
        if err != nil {
                fmt.Printf("error:%s", err)
                return
        }
        defer res.Body.Close()
        fmt.Printf("res:%s", res)
}
```

运行结果：

```bash
error:Get http://xxx/example.com: read tcp xxx:5002->xxx:5403: i/o timeout
```

## Refs

* [hustcat.github.io](http://hustcat.github.io/go-netpoller-and-timeout/)
* [The Go netpoller](https://morsmachine.dk/netpoller)
* [The complete guide to Go net/http timeouts](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)