---
layout: post
title: docker build configuration分析
date: 2016-10-28 21:41:55
category: 技术
tags: Docker-registry Docker Docker-build
excerpt: 分析docker高版本build中镜像configuration文件如何生成
---

分析`docker build`中，镜像configuration文件如何生成，重点几个信息：`created`、`config`、`author`、`config.Image`等

configuration文件`example`：

```sh
{"architecture":"amd64","config":{"Hostname":"xxxx","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["sh"],"ArgsEscaped":true,"Image":"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"container":"7dfa08cb9cbf2962b2362b1845b6657895685576015a8121652872fea56a7509","container_config":{"Hostname":"xxxx","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sh","-c","dd if=/dev/zero of=file bs=10M count=1"],"ArgsEscaped":true,"Image":"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"created":"2016-08-18T06:13:28.269459769Z","docker_version":"1.11.0-dev","history":[{"created":"2015-09-21T20:15:47.433616227Z","created_by":"/bin/sh -c #(nop) ADD file:6cccb5f0a3b3947116a0c0f55d071980d94427ba0d6dad17bc68ead832cc0a8f in /"},{"created":"2015-09-21T20:15:47.866196515Z","created_by":"/bin/sh -c #(nop) CMD [\"sh\"]"},{"created":"2016-08-18T06:13:28.269459769Z","created_by":"/bin/sh -c dd if=/dev/zero of=file bs=10M count=1"}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a","sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef","sha256:d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06"]}}
```

### docker build

```sh
$docker build -t image_name Dockerfile_path
```

```sh
$ docker build -t svendowideit/ambassador .
Sending build context to Docker daemon 15.36 kB
Step 1 : FROM alpine:3.2
 ---> 31f630c65071
Step 2 : MAINTAINER SvenDowideit@home.org.au
 ---> Using cache
 ---> 2a1c91448f5f
Step 3 : RUN apk update &&      apk add socat &&        rm -r /var/cache/
 ---> Using cache
 ---> 21ed6e7fbb73
Step 4 : CMD env | grep _TCP= | (sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat -t 100000000 TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3 \&/' && echo wait) | sh
 ---> Using cache
 ---> 7ea8aef582cc
Successfully built 7ea8aef582cc
```

docker client

```go
// CmdBuild builds a new image from the source code at a given path.
//
// If '-' is provided instead of a path or URL, Docker will build an image from either a Dockerfile or tar archive read from STDIN.
//
// Usage: docker build [OPTIONS] PATH | URL | -
func (cli *DockerCli) CmdBuild(args ...string) error {
    cmd := Cli.Subcmd("build", []string{"PATH | URL | -"}, Cli.DockerCommands["build"].Description, true)
    flTags := opts.NewListOpts(validateTag)
    cmd.Var(&flTags, []string{"t", "-tag"}, "Name and optionally a tag in the 'name:tag' format")
    suppressOutput := cmd.Bool([]string{"q", "-quiet"}, false, "Suppress the build output and print image ID on success")
    noCache := cmd.Bool([]string{"-no-cache"}, false, "Do not use cache when building the image")
    rm := cmd.Bool([]string{"-rm"}, true, "Remove intermediate containers after a successful build")
    forceRm := cmd.Bool([]string{"-force-rm"}, false, "Always remove intermediate containers")
    pull := cmd.Bool([]string{"-pull"}, false, "Always attempt to pull a newer version of the image")
    dockerfileName := cmd.String([]string{"f", "-file"}, "", "Name of the Dockerfile (Default is 'PATH/Dockerfile')")
    flMemoryString := cmd.String([]string{"m", "-memory"}, "", "Memory limit")
    flMemorySwap := cmd.String([]string{"-memory-swap"}, "", "Swap limit equal to memory plus swap: '-1' to enable unlimited swap")
    flShmSize := cmd.String([]string{"-shm-size"}, "", "Size of /dev/shm, default value is 64MB")
    flCPUShares := cmd.Int64([]string{"#c", "-cpu-shares"}, 0, "CPU shares (relative weight)")
    flCPUPeriod := cmd.Int64([]string{"-cpu-period"}, 0, "Limit the CPU CFS (Completely Fair Scheduler) period")
    flCPUQuota := cmd.Int64([]string{"-cpu-quota"}, 0, "Limit the CPU CFS (Completely Fair Scheduler) quota")
    flCPUSetCpus := cmd.String([]string{"-cpuset-cpus"}, "", "CPUs in which to allow execution (0-3, 0,1)")
    flCPUSetMems := cmd.String([]string{"-cpuset-mems"}, "", "MEMs in which to allow execution (0-3, 0,1)")
    flCgroupParent := cmd.String([]string{"-cgroup-parent"}, "", "Optional parent cgroup for the container")
    flBuildArg := opts.NewListOpts(runconfigopts.ValidateEnv)
    cmd.Var(&flBuildArg, []string{"-build-arg"}, "Set build-time variables")
    isolation := cmd.String([]string{"-isolation"}, "", "Container isolation technology")

    flLabels := opts.NewListOpts(nil)
    cmd.Var(&flLabels, []string{"-label"}, "Set metadata for an image")

    ulimits := make(map[string]*units.Ulimit)
    flUlimits := runconfigopts.NewUlimitOpt(&ulimits)
    cmd.Var(flUlimits, []string{"-ulimit"}, "Ulimit options")

    cmd.Require(flag.Exact, 1)

    // For trusted pull on "FROM <image>" instruction.
    addTrustedFlags(cmd, true)

    cmd.ParseFlags(args, true)

    var (
        ctx io.ReadCloser
        err error
    )

    specifiedContext := cmd.Arg(0)

    var (
        contextDir    string
        tempDir       string
        relDockerfile string
        progBuff      io.Writer
        buildBuff     io.Writer
    )

    progBuff = cli.out
    buildBuff = cli.out
    if *suppressOutput {
        progBuff = bytes.NewBuffer(nil)
        buildBuff = bytes.NewBuffer(nil)
    }

    switch {
    case specifiedContext == "-":
        ctx, relDockerfile, err = builder.GetContextFromReader(cli.in, *dockerfileName)
    case urlutil.IsGitURL(specifiedContext):
        tempDir, relDockerfile, err = builder.GetContextFromGitURL(specifiedContext, *dockerfileName)
    case urlutil.IsURL(specifiedContext):
        ctx, relDockerfile, err = builder.GetContextFromURL(progBuff, specifiedContext, *dockerfileName)
    default:
        contextDir, relDockerfile, err = builder.GetContextFromLocalDir(specifiedContext, *dockerfileName)
    }

    if err != nil {
        if *suppressOutput && urlutil.IsURL(specifiedContext) {
            fmt.Fprintln(cli.err, progBuff)
        }
        return fmt.Errorf("unable to prepare context: %s", err)
    }

    if tempDir != "" {
        defer os.RemoveAll(tempDir)
        contextDir = tempDir
    }

    if ctx == nil {
        // And canonicalize dockerfile name to a platform-independent one
        relDockerfile, err = archive.CanonicalTarNameForPath(relDockerfile)
        if err != nil {
            return fmt.Errorf("cannot canonicalize dockerfile path %s: %v", relDockerfile, err)
        }

        f, err := os.Open(filepath.Join(contextDir, ".dockerignore"))
        if err != nil && !os.IsNotExist(err) {
            return err
        }

        var excludes []string
        if err == nil {
            excludes, err = dockerignore.ReadAll(f)
            if err != nil {
                return err
            }
        }

        if err := builder.ValidateContextDirectory(contextDir, excludes); err != nil {
            return fmt.Errorf("Error checking context: '%s'.", err)
        }

        // If .dockerignore mentions .dockerignore or the Dockerfile
        // then make sure we send both files over to the daemon
        // because Dockerfile is, obviously, needed no matter what, and
        // .dockerignore is needed to know if either one needs to be
        // removed. The daemon will remove them for us, if needed, after it
        // parses the Dockerfile. Ignore errors here, as they will have been
        // caught by validateContextDirectory above.
        var includes = []string{"."}
        keepThem1, _ := fileutils.Matches(".dockerignore", excludes)
        keepThem2, _ := fileutils.Matches(relDockerfile, excludes)
        if keepThem1 || keepThem2 {
            includes = append(includes, ".dockerignore", relDockerfile)
        }

        ctx, err = archive.TarWithOptions(contextDir, &archive.TarOptions{
            Compression:     archive.Uncompressed,
            ExcludePatterns: excludes,
            IncludeFiles:    includes,
        })
        if err != nil {
            return err
        }
    }

    var resolvedTags []*resolvedTag
    if isTrusted() {
        // Wrap the tar archive to replace the Dockerfile entry with the rewritten
        // Dockerfile which uses trusted pulls.
        ctx = replaceDockerfileTarWrapper(ctx, relDockerfile, cli.trustedReference, &resolvedTags)
    }

    // Setup an upload progress bar
    progressOutput := streamformatter.NewStreamFormatter().NewProgressOutput(progBuff, true)

    var body io.Reader = progress.NewProgressReader(ctx, progressOutput, 0, "", "Sending build context to Docker daemon")

    var memory int64
    if *flMemoryString != "" {
        parsedMemory, err := units.RAMInBytes(*flMemoryString)
        if err != nil {
            return err
        }
        memory = parsedMemory
    }

    var memorySwap int64
    if *flMemorySwap != "" {
        if *flMemorySwap == "-1" {
            memorySwap = -1
        } else {
            parsedMemorySwap, err := units.RAMInBytes(*flMemorySwap)
            if err != nil {
                return err
            }
            memorySwap = parsedMemorySwap
        }
    }

    var shmSize int64
    if *flShmSize != "" {
        shmSize, err = units.RAMInBytes(*flShmSize)
        if err != nil {
            return err
        }
    }

    options := types.ImageBuildOptions{
        Context:        body,
        Memory:         memory,
        MemorySwap:     memorySwap,
        Tags:           flTags.GetAll(),
        SuppressOutput: *suppressOutput,
        NoCache:        *noCache,
        Remove:         *rm,
        ForceRemove:    *forceRm,
        PullParent:     *pull,
        Isolation:      container.Isolation(*isolation),
        CPUSetCPUs:     *flCPUSetCpus,
        CPUSetMems:     *flCPUSetMems,
        CPUShares:      *flCPUShares,
        CPUQuota:       *flCPUQuota,
        CPUPeriod:      *flCPUPeriod,
        CgroupParent:   *flCgroupParent,
        Dockerfile:     relDockerfile,
        ShmSize:        shmSize,
        Ulimits:        flUlimits.GetList(),
        BuildArgs:      runconfigopts.ConvertKVStringsToMap(flBuildArg.GetAll()),
        AuthConfigs:    cli.retrieveAuthConfigs(),
        Labels:         runconfigopts.ConvertKVStringsToMap(flLabels.GetAll()),
    }

    response, err := cli.client.ImageBuild(context.Background(), options)
    if err != nil {
        return err
    }
    defer response.Body.Close()

    err = jsonmessage.DisplayJSONMessagesStream(response.Body, buildBuff, cli.outFd, cli.isTerminalOut, nil)
    if err != nil {
        if jerr, ok := err.(*jsonmessage.JSONError); ok {
            // If no error code is set, default to 1
            if jerr.Code == 0 {
                jerr.Code = 1
            }
            if *suppressOutput {
                fmt.Fprintf(cli.err, "%s%s", progBuff, buildBuff)
            }
            return Cli.StatusError{Status: jerr.Message, StatusCode: jerr.Code}
        }
    }

    // Windows: show error message about modified file permissions if the
    // daemon isn't running Windows.
    if response.OSType != "windows" && runtime.GOOS == "windows" {
        fmt.Fprintln(cli.err, `SECURITY WARNING: You are building a Docker image from Windows against a non-Windows Docker host. All files and directories added to build context will have '-rwxr-xr-x' permissions. It is recommended to double check and reset permissions for sensitive files and directories.`)
    }

    // Everything worked so if -q was provided the output from the daemon
    // should be just the image ID and we'll print that to stdout.
    if *suppressOutput {
        fmt.Fprintf(cli.out, "%s", buildBuff)
    }

    if isTrusted() {
        // Since the build was successful, now we must tag any of the resolved
        // images from the above Dockerfile rewrite.
        for _, resolved := range resolvedTags {
            if err := cli.tagTrusted(resolved.digestRef, resolved.tagRef); err != nil {
                return err
            }
        }
    }

    return nil
}
// DockerCli represents the docker command line client.
// Instances of the client can be returned from NewDockerCli.
type DockerCli struct {
    // initializing closure
    init func() error

    // configFile has the client configuration file
    configFile *cliconfig.ConfigFile
    // in holds the input stream and closer (io.ReadCloser) for the client.
    in io.ReadCloser
    // out holds the output stream (io.Writer) for the client.
    out io.Writer
    // err holds the error stream (io.Writer) for the client.
    err io.Writer
    // keyFile holds the key file as a string.
    keyFile string
    // inFd holds the file descriptor of the client's STDIN (if valid).
    inFd uintptr
    // outFd holds file descriptor of the client's STDOUT (if valid).
    outFd uintptr
    // isTerminalIn indicates whether the client's STDIN is a TTY
    isTerminalIn bool
    // isTerminalOut indicates whether the client's STDOUT is a TTY
    isTerminalOut bool
    // client is the http client that performs all API operations
    client client.APIClient
    // state holds the terminal state
    state *term.State
}
// ImageBuild sends request to the daemon to build images.
// The Body in the response implement an io.ReadCloser and it's up to the caller to
// close it.
func (cli *Client) ImageBuild(ctx context.Context, options types.ImageBuildOptions) (types.ImageBuildResponse, error) {
    query, err := imageBuildOptionsToQuery(options)
    if err != nil {
        return types.ImageBuildResponse{}, err
    }

    headers := http.Header(make(map[string][]string))
    buf, err := json.Marshal(options.AuthConfigs)
    if err != nil {
        return types.ImageBuildResponse{}, err
    }
    headers.Add("X-Registry-Config", base64.URLEncoding.EncodeToString(buf))
    headers.Set("Content-Type", "application/tar")

    serverResp, err := cli.postRaw(ctx, "/build", query, options.Context, headers)
    if err != nil {
        return types.ImageBuildResponse{}, err
    }

    osType := getDockerOS(serverResp.header.Get("Server"))

    return types.ImageBuildResponse{
        Body:   serverResp.body,
        OSType: osType,
    }, nil
}
func (cli *Client) postRaw(ctx context.Context, path string, query url.Values, body io.Reader, headers map[string][]string) (*serverResponse, error) {
    return cli.sendClientRequest(ctx, "POST", path, query, body, headers)
}
func (cli *Client) sendClientRequest(ctx context.Context, method, path string, query url.Values, body io.Reader, headers map[string][]string) (*serverResponse, error) {
    serverResp := &serverResponse{
        body:       nil,
        statusCode: -1,
    }

    expectedPayload := (method == "POST" || method == "PUT")
    if expectedPayload && body == nil {
        body = bytes.NewReader([]byte{})
    }

    req, err := cli.newRequest(method, path, query, body, headers)
    req.URL.Host = cli.addr
    req.URL.Scheme = cli.transport.Scheme()

    if expectedPayload && req.Header.Get("Content-Type") == "" {
        req.Header.Set("Content-Type", "text/plain")
    }

    resp, err := cancellable.Do(ctx, cli.transport, req)
    if resp != nil {
        serverResp.statusCode = resp.StatusCode
    }

    if err != nil {
        if isTimeout(err) || strings.Contains(err.Error(), "connection refused") || strings.Contains(err.Error(), "dial unix") {
            return serverResp, ErrConnectionFailed
        }

        if !cli.transport.Secure() && strings.Contains(err.Error(), "malformed HTTP response") {
            return serverResp, fmt.Errorf("%v.\n* Are you trying to connect to a TLS-enabled daemon without TLS?", err)
        }
        if cli.transport.Secure() && strings.Contains(err.Error(), "remote error: bad certificate") {
            return serverResp, fmt.Errorf("The server probably has client authentication (--tlsverify) enabled. Please check your TLS client certification settings: %v", err)
        }

        return serverResp, fmt.Errorf("An error occurred trying to connect: %v", err)
    }

    if serverResp.statusCode < 200 || serverResp.statusCode >= 400 {
        body, err := ioutil.ReadAll(resp.Body)
        if err != nil {
            return serverResp, err
        }
        if len(body) == 0 {
            return serverResp, fmt.Errorf("Error: request returned %s for API route and version %s, check if the server supports the requested API version", http.StatusText(serverResp.statusCode), req.URL)
        }
        return serverResp, fmt.Errorf("Error response from daemon: %s", bytes.TrimSpace(body))
    }

    serverResp.body = resp.Body
    serverResp.header = resp.Header
    return serverResp, nil
}
```

docker daemon

```go
// CmdDaemon is the daemon command, called the raw arguments after `docker daemon`.
func (cli *DaemonCli) CmdDaemon(args ...string) error {
    // warn from uuid package when running the daemon
    uuid.Loggerf = logrus.Warnf

    if !commonFlags.FlagSet.IsEmpty() || !clientFlags.FlagSet.IsEmpty() {
        // deny `docker -D daemon`
        illegalFlag := getGlobalFlag()
        fmt.Fprintf(os.Stderr, "invalid flag '-%s'.\nSee 'docker daemon --help'.\n", illegalFlag.Names[0])
        os.Exit(1)
    } else {
        // allow new form `docker daemon -D`
        flag.Merge(cli.flags, commonFlags.FlagSet)
    }

    configFile := cli.flags.String([]string{daemonConfigFileFlag}, defaultDaemonConfigFile, "Daemon configuration file")

    cli.flags.ParseFlags(args, true)
    commonFlags.PostParse()

    if commonFlags.TrustKey == "" {
        commonFlags.TrustKey = filepath.Join(getDaemonConfDir(), defaultTrustKeyFile)
    }
    cliConfig, err := loadDaemonCliConfig(cli.Config, cli.flags, commonFlags, *configFile)
    if err != nil {
        fmt.Fprint(os.Stderr, err)
        os.Exit(1)
    }
    cli.Config = cliConfig

    if cli.Config.Debug {
        utils.EnableDebug()
    }

    if utils.ExperimentalBuild() {
        logrus.Warn("Running experimental build")
    }

    logrus.SetFormatter(&logrus.TextFormatter{
        TimestampFormat: jsonlog.RFC3339NanoFixed,
        DisableColors:   cli.Config.RawLogs,
    })

    if err := setDefaultUmask(); err != nil {
        logrus.Fatalf("Failed to set umask: %v", err)
    }

    if len(cli.LogConfig.Config) > 0 {
        if err := logger.ValidateLogOpts(cli.LogConfig.Type, cli.LogConfig.Config); err != nil {
            logrus.Fatalf("Failed to set log opts: %v", err)
        }
    }

    var pfile *pidfile.PIDFile
    if cli.Pidfile != "" {
        pf, err := pidfile.New(cli.Pidfile)
        if err != nil {
            logrus.Fatalf("Error starting daemon: %v", err)
        }
        pfile = pf
        defer func() {
            if err := pfile.Remove(); err != nil {
                logrus.Error(err)
            }
        }()
    }

    serverConfig := &apiserver.Config{
        AuthorizationPluginNames: cli.Config.AuthorizationPlugins,
        Logging:                  true,
        SocketGroup:              cli.Config.SocketGroup,
        Version:                  dockerversion.Version,
    }
    serverConfig = setPlatformServerConfig(serverConfig, cli.Config)

    if cli.Config.TLS {
        tlsOptions := tlsconfig.Options{
            CAFile:   cli.Config.CommonTLSOptions.CAFile,
            CertFile: cli.Config.CommonTLSOptions.CertFile,
            KeyFile:  cli.Config.CommonTLSOptions.KeyFile,
        }

        if cli.Config.TLSVerify {
            // server requires and verifies client's certificate
            tlsOptions.ClientAuth = tls.RequireAndVerifyClientCert
        }
        tlsConfig, err := tlsconfig.Server(tlsOptions)
        if err != nil {
            logrus.Fatal(err)
        }
        serverConfig.TLSConfig = tlsConfig
    }

    if len(cli.Config.Hosts) == 0 {
        cli.Config.Hosts = make([]string, 1)
    }

    api := apiserver.New(serverConfig)

    for i := 0; i < len(cli.Config.Hosts); i++ {
        var err error
        if cli.Config.Hosts[i], err = opts.ParseHost(cli.Config.TLS, cli.Config.Hosts[i]); err != nil {
            logrus.Fatalf("error parsing -H %s : %v", cli.Config.Hosts[i], err)
        }

        protoAddr := cli.Config.Hosts[i]
        protoAddrParts := strings.SplitN(protoAddr, "://", 2)
        if len(protoAddrParts) != 2 {
            logrus.Fatalf("bad format %s, expected PROTO://ADDR", protoAddr)
        }
        l, err := listeners.Init(protoAddrParts[0], protoAddrParts[1], serverConfig.SocketGroup, serverConfig.TLSConfig)
        if err != nil {
            logrus.Fatal(err)
        }

        logrus.Debugf("Listener created for HTTP on %s (%s)", protoAddrParts[0], protoAddrParts[1])
        api.Accept(protoAddrParts[1], l...)
    }

    if err := migrateKey(); err != nil {
        logrus.Fatal(err)
    }
    cli.TrustKeyPath = commonFlags.TrustKey

    registryService := registry.NewService(cli.Config.ServiceOptions)

    containerdRemote, err := libcontainerd.New(filepath.Join(cli.Config.ExecRoot, "libcontainerd"), cli.getPlatformRemoteOptions()...)
    if err != nil {
        logrus.Fatal(err)
    }

    d, err := daemon.NewDaemon(cli.Config, registryService, containerdRemote)
    if err != nil {
        if pfile != nil {
            if err := pfile.Remove(); err != nil {
                logrus.Error(err)
            }
        }
        logrus.Fatalf("Error starting daemon: %v", err)
    }

    logrus.Info("Daemon has completed initialization")

    logrus.WithFields(logrus.Fields{
        "version":     dockerversion.Version,
        "commit":      dockerversion.GitCommit,
        "graphdriver": d.GraphDriverName(),
    }).Info("Docker daemon")

    initRouter(api, d)

    reload := func(config *daemon.Config) {
        if err := d.Reload(config); err != nil {
            logrus.Errorf("Error reconfiguring the daemon: %v", err)
            return
        }
        if config.IsValueSet("debug") {
            debugEnabled := utils.IsDebugEnabled()
            switch {
            case debugEnabled && !config.Debug: // disable debug
                utils.DisableDebug()
                api.DisableProfiler()
            case config.Debug && !debugEnabled: // enable debug
                utils.EnableDebug()
                api.EnableProfiler()
            }

        }
    }

    setupConfigReloadTrap(*configFile, cli.flags, reload)

    // The serve API routine never exits unless an error occurs
    // We need to start it as a goroutine and wait on it so
    // daemon doesn't exit
    serveAPIWait := make(chan error)
    go api.Wait(serveAPIWait)

    signal.Trap(func() {
        api.Close()
        <-serveAPIWait
        shutdownDaemon(d, 15)
        if pfile != nil {
            if err := pfile.Remove(); err != nil {
                logrus.Error(err)
            }
        }
    })

    // after the daemon is done setting up we can notify systemd api
    notifySystem()

    // Daemon is fully initialized and handling API traffic
    // Wait for serve API to complete
    errAPI := <-serveAPIWait
    shutdownDaemon(d, 15)
    containerdRemote.Cleanup()
    if errAPI != nil {
        if pfile != nil {
            if err := pfile.Remove(); err != nil {
                logrus.Error(err)
            }
        }
        logrus.Fatalf("Shutting down due to ServeAPI error: %v", errAPI)
    }
    return nil
}
func initRouter(s *apiserver.Server, d *daemon.Daemon) {
    routers := []router.Router{
        container.NewRouter(d),
        image.NewRouter(d),
        systemrouter.NewRouter(d),
        volume.NewRouter(d),
        build.NewRouter(dockerfile.NewBuildManager(d)),
    }
    if d.NetworkControllerEnabled() {
        routers = append(routers, network.NewRouter(d))
    }

    s.InitRouter(utils.IsDebugEnabled(), routers...)
}
// NewBuildManager creates a BuildManager.
func NewBuildManager(b builder.Backend) (bm *BuildManager) {
    return &BuildManager{backend: b}
}
// BuildManager implements builder.Backend and is shared across all Builder objects.
type BuildManager struct {
    backend builder.Backend
}
// buildRouter is a router to talk with the build controller
type buildRouter struct {
    backend Backend
    routes  []router.Route
}
// NewRouter initializes a new build router
func NewRouter(b Backend) router.Router {
    r := &buildRouter{
        backend: b,
    }
    r.initRoutes()
    return r
}
func (r *buildRouter) initRoutes() {
    r.routes = []router.Route{
        router.NewPostRoute("/build", r.postBuild),
    }
}
// NewPostRoute initializes a new route with the http method POST.
func NewPostRoute(path string, handler httputils.APIFunc) Route {
    return NewRoute("POST", path, handler)
}
// NewRoute initializes a new local route for the router.
func NewRoute(method, path string, handler httputils.APIFunc) Route {
    return localRoute{method, path, handler}
}
// localRoute defines an individual API route to connect
// with the docker daemon. It implements Route.
type localRoute struct {
    method  string
    path    string
    handler httputils.APIFunc
}
func (br *buildRouter) postBuild(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
    var (
        authConfigs        = map[string]types.AuthConfig{}
        authConfigsEncoded = r.Header.Get("X-Registry-Config")
        notVerboseBuffer   = bytes.NewBuffer(nil)
    )

    if authConfigsEncoded != "" {
        authConfigsJSON := base64.NewDecoder(base64.URLEncoding, strings.NewReader(authConfigsEncoded))
        if err := json.NewDecoder(authConfigsJSON).Decode(&authConfigs); err != nil {
            // for a pull it is not an error if no auth was given
            // to increase compatibility with the existing api it is defaulting
            // to be empty.
        }
    }

    w.Header().Set("Content-Type", "application/json")

    output := ioutils.NewWriteFlusher(w)
    defer output.Close()
    sf := streamformatter.NewJSONStreamFormatter()
    errf := func(err error) error {
        if httputils.BoolValue(r, "q") && notVerboseBuffer.Len() > 0 {
            output.Write(notVerboseBuffer.Bytes())
        }
        // Do not write the error in the http output if it's still empty.
        // This prevents from writing a 200(OK) when there is an internal error.
        if !output.Flushed() {
            return err
        }
        _, err = w.Write(sf.FormatError(err))
        if err != nil {
            logrus.Warnf("could not write error response: %v", err)
        }
        return nil
    }

    buildOptions, err := newImageBuildOptions(ctx, r)
    if err != nil {
        return errf(err)
    }

    remoteURL := r.FormValue("remote")

    // Currently, only used if context is from a remote url.
    // Look at code in DetectContextFromRemoteURL for more information.
    createProgressReader := func(in io.ReadCloser) io.ReadCloser {
        progressOutput := sf.NewProgressOutput(output, true)
        if buildOptions.SuppressOutput {
            progressOutput = sf.NewProgressOutput(notVerboseBuffer, true)
        }
        return progress.NewProgressReader(in, progressOutput, r.ContentLength, "Downloading context", remoteURL)
    }

    var (
        context        builder.ModifiableContext
        dockerfileName string
        out            io.Writer
    )
    context, dockerfileName, err = builder.DetectContextFromRemoteURL(r.Body, remoteURL, createProgressReader)
    if err != nil {
        return errf(err)
    }
    defer func() {
        if err := context.Close(); err != nil {
            logrus.Debugf("[BUILDER] failed to remove temporary context: %v", err)
        }
    }()
    if len(dockerfileName) > 0 {
        buildOptions.Dockerfile = dockerfileName
    }

    buildOptions.AuthConfigs = authConfigs

    out = output
    if buildOptions.SuppressOutput {
        out = notVerboseBuffer
    }
    out = &syncWriter{w: out}
    stdout := &streamformatter.StdoutFormatter{Writer: out, StreamFormatter: sf}
    stderr := &streamformatter.StderrFormatter{Writer: out, StreamFormatter: sf}

    closeNotifier := make(<-chan bool)
    if notifier, ok := w.(http.CloseNotifier); ok {
        closeNotifier = notifier.CloseNotify()
    }

    imgID, err := br.backend.Build(ctx, buildOptions,
        builder.DockerIgnoreContext{ModifiableContext: context},
        stdout, stderr, out,
        closeNotifier)
    if err != nil {
        return errf(err)
    }

    // Everything worked so if -q was provided the output from the daemon
    // should be just the image ID and we'll print that to stdout.
    if buildOptions.SuppressOutput {
        stdout := &streamformatter.StdoutFormatter{Writer: output, StreamFormatter: sf}
        fmt.Fprintf(stdout, "%s\n", string(imgID))
    }

    return nil
}
// InitRouter initializes the list of routers for the server.
// This method also enables the Go profiler if enableProfiler is true.
func (s *Server) InitRouter(enableProfiler bool, routers ...router.Router) {
    for _, r := range routers {
        s.routers = append(s.routers, r)
    }

    m := s.createMux()
    if enableProfiler {
        profilerSetup(m)
    }
    s.routerSwapper = &routerSwapper{
        router: m,
    }
}
// createMux initializes the main router the server uses.
func (s *Server) createMux() *mux.Router {
    m := mux.NewRouter()

    logrus.Debugf("Registering routers")
    for _, apiRouter := range s.routers {
        for _, r := range apiRouter.Routes() {
            f := s.makeHTTPHandler(r.Handler())

            logrus.Debugf("Registering %s, %s", r.Method(), r.Path())
            m.Path(versionMatcher + r.Path()).Methods(r.Method()).Handler(f)
            m.Path(r.Path()).Methods(r.Method()).Handler(f)
        }
    }

    return m
}
```

调用build函数创建镜像

```go
// Build creates a NewBuilder, which builds the image.
func (bm *BuildManager) Build(clientCtx context.Context, config *types.ImageBuildOptions, context builder.Context, stdout io.Writer, stderr io.Writer, out io.Writer, clientGone <-chan bool) (string, error) {
    b, err := NewBuilder(clientCtx, config, bm.backend, context, nil)
    if err != nil {
        return "", err
    }
    img, err := b.build(config, context, stdout, stderr, out, clientGone)
    return img, err

}
// NewBuilder creates a new Dockerfile builder from an optional dockerfile and a Config.
// If dockerfile is nil, the Dockerfile specified by Config.DockerfileName,
// will be read from the Context passed to Build().
func NewBuilder(clientCtx context.Context, config *types.ImageBuildOptions, backend builder.Backend, context builder.Context, dockerfile io.ReadCloser) (b *Builder, err error) {
    if config == nil {
        config = new(types.ImageBuildOptions)
    }
    if config.BuildArgs == nil {
        config.BuildArgs = make(map[string]string)
    }
    b = &Builder{
        clientCtx:        clientCtx,
        options:          config,
        Stdout:           os.Stdout,
        Stderr:           os.Stderr,
        docker:           backend,
        context:          context,
        runConfig:        new(container.Config),
        tmpContainers:    map[string]struct{}{},
        cancelled:        make(chan struct{}),
        id:               stringid.GenerateNonCryptoID(),
        allowedBuildArgs: make(map[string]bool),
    }
    if dockerfile != nil {
        b.dockerfile, err = parser.Parse(dockerfile)
        if err != nil {
            return nil, err
        }
    }

    return b, nil
}
// build runs the Dockerfile builder from a context and a docker object that allows to make calls
// to Docker.
//
// This will (barring errors):
//
// * read the dockerfile from context
// * parse the dockerfile if not already parsed
// * walk the AST and execute it by dispatching to handlers. If Remove
//   or ForceRemove is set, additional cleanup around containers happens after
//   processing.
// * Tag image, if applicable.
// * Print a happy message and return the image ID.
//
func (b *Builder) build(config *types.ImageBuildOptions, context builder.Context, stdout io.Writer, stderr io.Writer, out io.Writer, clientGone <-chan bool) (string, error) {
    b.options = config
    b.context = context
    b.Stdout = stdout
    b.Stderr = stderr
    b.Output = out

    // If Dockerfile was not parsed yet, extract it from the Context
    if b.dockerfile == nil {
        if err := b.readDockerfile(); err != nil {
            return "", err
        }
    }

    finished := make(chan struct{})
    defer close(finished)
    go func() {
        select {
        case <-finished:
        case <-clientGone:
            b.cancelOnce.Do(func() {
                close(b.cancelled)
            })
        }

    }()

    repoAndTags, err := sanitizeRepoAndTags(config.Tags)
    if err != nil {
        return "", err
    }

    var shortImgID string
    for i, n := range b.dockerfile.Children {
        // we only want to add labels to the last layer
        if i == len(b.dockerfile.Children)-1 {
            b.addLabels()
        }
        select {
        case <-b.cancelled:
            logrus.Debug("Builder: build cancelled!")
            fmt.Fprintf(b.Stdout, "Build cancelled")
            return "", fmt.Errorf("Build cancelled")
        default:
            // Not cancelled yet, keep going...
        }
        if err := b.dispatch(i, n); err != nil {
            if b.options.ForceRemove {
                b.clearTmp()
            }
            return "", err
        }

        // Commit the layer when there are only one children in
        // the dockerfile, this is only the `FROM` tag, and
        // build labels. Otherwise, the new image won't be
        // labeled properly.
        // Commit here, so the ID of the final image is reported
        // properly.
        if len(b.dockerfile.Children) == 1 && len(b.options.Labels) > 0 {
            b.commit("", b.runConfig.Cmd, "")
        }

        shortImgID = stringid.TruncateID(b.image)
        fmt.Fprintf(b.Stdout, " ---> %s\n", shortImgID)
        if b.options.Remove {
            b.clearTmp()
        }
    }

    // check if there are any leftover build-args that were passed but not
    // consumed during build. Return an error, if there are any.
    leftoverArgs := []string{}
    for arg := range b.options.BuildArgs {
        if !b.isBuildArgAllowed(arg) {
            leftoverArgs = append(leftoverArgs, arg)
        }
    }
    if len(leftoverArgs) > 0 {
        return "", fmt.Errorf("One or more build-args %v were not consumed, failing build.", leftoverArgs)
    }

    if b.image == "" {
        return "", fmt.Errorf("No image was generated. Is your Dockerfile empty?")
    }

    for _, rt := range repoAndTags {
        if err := b.docker.TagImage(rt, b.image); err != nil {
            return "", err
        }
    }

    fmt.Fprintf(b.Stdout, "Successfully built %s\n", shortImgID)
    return b.image, nil
}
// Builder is a Dockerfile builder
// It implements the builder.Backend interface.
type Builder struct {
    options *types.ImageBuildOptions

    Stdout io.Writer
    Stderr io.Writer
    Output io.Writer

    docker    builder.Backend
    context   builder.Context
    clientCtx context.Context

    dockerfile       *parser.Node
    runConfig        *container.Config // runconfig for cmd, run, entrypoint etc.
    flags            *BFlags
    tmpContainers    map[string]struct{}
    image            string // imageID
    noBaseImage      bool
    maintainer       string
    cmdSet           bool
    disableCommit    bool
    cacheBusted      bool
    cancelled        chan struct{}
    cancelOnce       sync.Once
    allowedBuildArgs map[string]bool // list of build-time args that are allowed for expansion/substitution and passing to commands in 'run'.

    // TODO: remove once docker.Commit can receive a tag
    id string
}
// Node is a structure used to represent a parse tree.
//
// In the node there are three fields, Value, Next, and Children. Value is the
// current token's string value. Next is always the next non-child token, and
// children contains all the children. Here's an example:
//
// (value next (child child-next child-next-next) next-next)
//
// This data structure is frankly pretty lousy for handling complex languages,
// but lucky for us the Dockerfile isn't very complicated. This structure
// works a little more effectively than a "proper" parse tree for our needs.
//
type Node struct {
    Value      string          // actual content
    Next       *Node           // the next item in the current sexp
    Children   []*Node         // the children of this sexp
    Attributes map[string]bool // special attributes for this node
    Original   string          // original line used before parsing
    Flags      []string        // only top Node should have this set
    StartLine  int             // the line in the original dockerfile where the node begins
    EndLine    int             // the line in the original dockerfile where the node ends
}
// This method is the entrypoint to all statement handling routines.
//
// Almost all nodes will have this structure:
// Child[Node, Node, Node] where Child is from parser.Node.Children and each
// node comes from parser.Node.Next. This forms a "line" with a statement and
// arguments and we process them in this normalized form by hitting
// evaluateTable with the leaf nodes of the command and the Builder object.
//
// ONBUILD is a special case; in this case the parser will emit:
// Child[Node, Child[Node, Node...]] where the first node is the literal
// "onbuild" and the child entrypoint is the command of the ONBUILD statement,
// such as `RUN` in ONBUILD RUN foo. There is special case logic in here to
// deal with that, at least until it becomes more of a general concern with new
// features.
func (b *Builder) dispatch(stepN int, ast *parser.Node) error {
    cmd := ast.Value
    upperCasedCmd := strings.ToUpper(cmd)

    // To ensure the user is given a decent error message if the platform
    // on which the daemon is running does not support a builder command.
    if err := platformSupports(strings.ToLower(cmd)); err != nil {
        return err
    }

    attrs := ast.Attributes
    original := ast.Original
    flags := ast.Flags
    strList := []string{}
    msg := fmt.Sprintf("Step %d : %s", stepN+1, upperCasedCmd)

    if len(ast.Flags) > 0 {
        msg += " " + strings.Join(ast.Flags, " ")
    }

    if cmd == "onbuild" {
        if ast.Next == nil {
            return fmt.Errorf("ONBUILD requires at least one argument")
        }
        ast = ast.Next.Children[0]
        strList = append(strList, ast.Value)
        msg += " " + ast.Value

        if len(ast.Flags) > 0 {
            msg += " " + strings.Join(ast.Flags, " ")
        }

    }

    // count the number of nodes that we are going to traverse first
    // so we can pre-create the argument and message array. This speeds up the
    // allocation of those list a lot when they have a lot of arguments
    cursor := ast
    var n int
    for cursor.Next != nil {
        cursor = cursor.Next
        n++
    }
    msgList := make([]string, n)

    var i int
    // Append the build-time args to config-environment.
    // This allows builder config to override the variables, making the behavior similar to
    // a shell script i.e. `ENV foo bar` overrides value of `foo` passed in build
    // context. But `ENV foo $foo` will use the value from build context if one
    // isn't already been defined by a previous ENV primitive.
    // Note, we get this behavior because we know that ProcessWord() will
    // stop on the first occurrence of a variable name and not notice
    // a subsequent one. So, putting the buildArgs list after the Config.Env
    // list, in 'envs', is safe.
    envs := b.runConfig.Env
    for key, val := range b.options.BuildArgs {
        if !b.isBuildArgAllowed(key) {
            // skip build-args that are not in allowed list, meaning they have
            // not been defined by an "ARG" Dockerfile command yet.
            // This is an error condition but only if there is no "ARG" in the entire
            // Dockerfile, so we'll generate any necessary errors after we parsed
            // the entire file (see 'leftoverArgs' processing in evaluator.go )
            continue
        }
        envs = append(envs, fmt.Sprintf("%s=%s", key, val))
    }
    for ast.Next != nil {
        ast = ast.Next
        var str string
        str = ast.Value
        if replaceEnvAllowed[cmd] {
            var err error
            var words []string

            if allowWordExpansion[cmd] {
                words, err = ProcessWords(str, envs)
                if err != nil {
                    return err
                }
                strList = append(strList, words...)
            } else {
                str, err = ProcessWord(str, envs)
                if err != nil {
                    return err
                }
                strList = append(strList, str)
            }
        } else {
            strList = append(strList, str)
        }
        msgList[i] = ast.Value
        i++
    }

    msg += " " + strings.Join(msgList, " ")
    fmt.Fprintln(b.Stdout, msg)

    // XXX yes, we skip any cmds that are not valid; the parser should have
    // picked these out already.
    if f, ok := evaluateTable[cmd]; ok {
        b.flags = NewBFlags()
        b.flags.Args = flags
        return f(b, strList, attrs, original)
    }

    return fmt.Errorf("Unknown instruction: %s", upperCasedCmd)
}
var evaluateTable map[string]func(*Builder, []string, map[string]bool, string) error
func init() {
    evaluateTable = map[string]func(*Builder, []string, map[string]bool, string) error{
        command.Env:        env,
        command.Label:      label,
        command.Maintainer: maintainer,
        command.Add:        add,
        command.Copy:       dispatchCopy, // copy() is a go builtin
        command.From:       from,
        command.Onbuild:    onbuild,
        command.Workdir:    workdir,
        command.Run:        run,
        command.Cmd:        cmd,
        command.Entrypoint: entrypoint,
        command.Expose:     expose,
        command.Volume:     volume,
        command.User:       user,
        command.StopSignal: stopSignal,
        command.Arg:        arg,
    }
}
// FROM imagename
//
// This sets the image the dockerfile will build on top of.
//
func from(b *Builder, args []string, attributes map[string]bool, original string) error {
    if len(args) != 1 {
        return errExactlyOneArgument("FROM")
    }

    if err := b.flags.Parse(); err != nil {
        return err
    }

    name := args[0]

    var (
        image builder.Image
        err   error
    )

    // Windows cannot support a container with no base image.
    if name == api.NoBaseImageSpecifier {
        if runtime.GOOS == "windows" {
            return fmt.Errorf("Windows does not support FROM scratch")
        }
        b.image = ""
        b.noBaseImage = true
    } else {
        // TODO: don't use `name`, instead resolve it to a digest
        if !b.options.PullParent {
            image, err = b.docker.GetImageOnBuild(name)
            // TODO: shouldn't we error out if error is different from "not found" ?
        }
        if image == nil {
            image, err = b.docker.PullOnBuild(b.clientCtx, name, b.options.AuthConfigs, b.Output)
            if err != nil {
                return err
            }
        }
    }

    return b.processImageFrom(image)
}
func (b *Builder) processImageFrom(img builder.Image) error {
    if img != nil {
        b.image = img.ImageID()

        if img.RunConfig() != nil {
            imgConfig := *img.RunConfig()
            // inherit runConfig labels from the current
            // state if they've been set already.
            // Ensures that images with only a FROM
            // get the labels populated properly.
            if b.runConfig.Labels != nil {
                if imgConfig.Labels == nil {
                    imgConfig.Labels = make(map[string]string)
                }
                for k, v := range b.runConfig.Labels {
                    imgConfig.Labels[k] = v
                }
            }
            b.runConfig = &imgConfig
        }
    }

    // Check to see if we have a default PATH, note that windows won't
    // have one as its set by HCS
    if system.DefaultPathEnv != "" {
        // Convert the slice of strings that represent the current list
        // of env vars into a map so we can see if PATH is already set.
        // If its not set then go ahead and give it our default value
        configEnv := opts.ConvertKVStringsToMap(b.runConfig.Env)
        if _, ok := configEnv["PATH"]; !ok {
            b.runConfig.Env = append(b.runConfig.Env,
                "PATH="+system.DefaultPathEnv)
        }
    }

    if img == nil {
        // Typically this means they used "FROM scratch"
        return nil
    }

    // Process ONBUILD triggers if they exist
    if nTriggers := len(b.runConfig.OnBuild); nTriggers != 0 {
        word := "trigger"
        if nTriggers > 1 {
            word = "triggers"
        }
        fmt.Fprintf(b.Stderr, "# Executing %d build %s...\n", nTriggers, word)
    }

    // Copy the ONBUILD triggers, and remove them from the config, since the config will be committed.
    onBuildTriggers := b.runConfig.OnBuild
    b.runConfig.OnBuild = []string{}

    // parse the ONBUILD triggers by invoking the parser
    for _, step := range onBuildTriggers {
        ast, err := parser.Parse(strings.NewReader(step))
        if err != nil {
            return err
        }

        for i, n := range ast.Children {
            switch strings.ToUpper(n.Value) {
            case "ONBUILD":
                return fmt.Errorf("Chaining ONBUILD via `ONBUILD ONBUILD` isn't allowed")
            case "MAINTAINER", "FROM":
                return fmt.Errorf("%s isn't allowed as an ONBUILD trigger", n.Value)
            }

            if err := b.dispatch(i, n); err != nil {
                return err
            }
        }
    }

    return nil
}
// RUN some command yo
//
// run a command and commit the image. Args are automatically prepended with
// 'sh -c' under linux or 'cmd /S /C' under Windows, in the event there is
// only one argument. The difference in processing:
//
// RUN echo hi          # sh -c echo hi       (Linux)
// RUN echo hi          # cmd /S /C echo hi   (Windows)
// RUN [ "echo", "hi" ] # echo hi
//
func run(b *Builder, args []string, attributes map[string]bool, original string) error {
    if b.image == "" && !b.noBaseImage {
        return fmt.Errorf("Please provide a source image with `from` prior to run")
    }

    if err := b.flags.Parse(); err != nil {
        return err
    }

    args = handleJSONArgs(args, attributes)

    if !attributes["json"] {
        if runtime.GOOS != "windows" {
            args = append([]string{"/bin/sh", "-c"}, args...)
        } else {
            args = append([]string{"cmd", "/S", "/C"}, args...)
        }
    }

    config := &container.Config{
        Cmd:   strslice.StrSlice(args),
        Image: b.image,
    }

    // stash the cmd
    cmd := b.runConfig.Cmd
    if len(b.runConfig.Entrypoint) == 0 && len(b.runConfig.Cmd) == 0 {
        b.runConfig.Cmd = config.Cmd
    }

    // stash the config environment
    env := b.runConfig.Env

    defer func(cmd strslice.StrSlice) { b.runConfig.Cmd = cmd }(cmd)
    defer func(env []string) { b.runConfig.Env = env }(env)

    // derive the net build-time environment for this run. We let config
    // environment override the build time environment.
    // This means that we take the b.buildArgs list of env vars and remove
    // any of those variables that are defined as part of the container. In other
    // words, anything in b.Config.Env. What's left is the list of build-time env
    // vars that we need to add to each RUN command - note the list could be empty.
    //
    // We don't persist the build time environment with container's config
    // environment, but just sort and prepend it to the command string at time
    // of commit.
    // This helps with tracing back the image's actual environment at the time
    // of RUN, without leaking it to the final image. It also aids cache
    // lookup for same image built with same build time environment.
    cmdBuildEnv := []string{}
    configEnv := runconfigopts.ConvertKVStringsToMap(b.runConfig.Env)
    for key, val := range b.options.BuildArgs {
        if !b.isBuildArgAllowed(key) {
            // skip build-args that are not in allowed list, meaning they have
            // not been defined by an "ARG" Dockerfile command yet.
            // This is an error condition but only if there is no "ARG" in the entire
            // Dockerfile, so we'll generate any necessary errors after we parsed
            // the entire file (see 'leftoverArgs' processing in evaluator.go )
            continue
        }
        if _, ok := configEnv[key]; !ok {
            cmdBuildEnv = append(cmdBuildEnv, fmt.Sprintf("%s=%s", key, val))
        }
    }

    // derive the command to use for probeCache() and to commit in this container.
    // Note that we only do this if there are any build-time env vars.  Also, we
    // use the special argument "|#" at the start of the args array. This will
    // avoid conflicts with any RUN command since commands can not
    // start with | (vertical bar). The "#" (number of build envs) is there to
    // help ensure proper cache matches. We don't want a RUN command
    // that starts with "foo=abc" to be considered part of a build-time env var.
    saveCmd := config.Cmd
    if len(cmdBuildEnv) > 0 {
        sort.Strings(cmdBuildEnv)
        tmpEnv := append([]string{fmt.Sprintf("|%d", len(cmdBuildEnv))}, cmdBuildEnv...)
        saveCmd = strslice.StrSlice(append(tmpEnv, saveCmd...))
    }

    b.runConfig.Cmd = saveCmd
    hit, err := b.probeCache()
    if err != nil {
        return err
    }
    if hit {
        return nil
    }

    // set Cmd manually, this is special case only for Dockerfiles
    b.runConfig.Cmd = config.Cmd
    // set build-time environment for 'run'.
    b.runConfig.Env = append(b.runConfig.Env, cmdBuildEnv...)
    // set config as already being escaped, this prevents double escaping on windows
    b.runConfig.ArgsEscaped = true

    logrus.Debugf("[BUILDER] Command to be executed: %v", b.runConfig.Cmd)

    cID, err := b.create()
    if err != nil {
        return err
    }

    if err := b.run(cID); err != nil {
        return err
    }

    // revert to original config environment and set the command string to
    // have the build-time env vars in it (if any) so that future cache look-ups
    // properly match it.
    b.runConfig.Env = env
    b.runConfig.Cmd = saveCmd
    return b.commit(cID, cmd, "run")
}
func (b *Builder) create() (string, error) {
    if b.image == "" && !b.noBaseImage {
        return "", fmt.Errorf("Please provide a source image with `from` prior to run")
    }
    b.runConfig.Image = b.image

    resources := container.Resources{
        CgroupParent: b.options.CgroupParent,
        CPUShares:    b.options.CPUShares,
        CPUPeriod:    b.options.CPUPeriod,
        CPUQuota:     b.options.CPUQuota,
        CpusetCpus:   b.options.CPUSetCPUs,
        CpusetMems:   b.options.CPUSetMems,
        Memory:       b.options.Memory,
        MemorySwap:   b.options.MemorySwap,
        Ulimits:      b.options.Ulimits,
    }

    // TODO: why not embed a hostconfig in builder?
    hostConfig := &container.HostConfig{
        Isolation: b.options.Isolation,
        ShmSize:   b.options.ShmSize,
        Resources: resources,
    }

    config := *b.runConfig

    // Create the container
    c, err := b.docker.ContainerCreate(types.ContainerCreateConfig{
        Config:     b.runConfig,
        HostConfig: hostConfig,
    })
    if err != nil {
        return "", err
    }
    for _, warning := range c.Warnings {
        fmt.Fprintf(b.Stdout, " ---> [Warning] %s\n", warning)
    }

    b.tmpContainers[c.ID] = struct{}{}
    fmt.Fprintf(b.Stdout, " ---> Running in %s\n", stringid.TruncateID(c.ID))

    // override the entry point that may have been picked up from the base image
    if err := b.docker.ContainerUpdateCmdOnBuild(c.ID, config.Cmd); err != nil {
        return "", err
    }

    return c.ID, nil
}
// Config contains the configuration data about a container.
// It should hold only portable information about the container.
// Here, "portable" means "independent from the host we are running on".
// Non-portable information *should* appear in HostConfig.
// All fields added to this struct must be marked `omitempty` to keep getting
// predictable hashes from the old `v1Compatibility` configuration.
type Config struct {
    Hostname        string                // Hostname
    Domainname      string                // Domainname
    User            string                // User that will run the command(s) inside the container
    AttachStdin     bool                  // Attach the standard input, makes possible user interaction
    AttachStdout    bool                  // Attach the standard output
    AttachStderr    bool                  // Attach the standard error
    ExposedPorts    map[nat.Port]struct{} `json:",omitempty"` // List of exposed ports
    PublishService  string                `json:",omitempty"` // Name of the network service exposed by the container
    Tty             bool                  // Attach standard streams to a tty, including stdin if it is not closed.
    OpenStdin       bool                  // Open stdin
    StdinOnce       bool                  // If true, close stdin after the 1 attached client disconnects.
    Env             []string              // List of environment variable to set in the container
    Cmd             strslice.StrSlice     // Command to run when starting the container
    ArgsEscaped     bool                  `json:",omitempty"` // True if command is already escaped (Windows specific)
    Image           string                // Name of the image as it was passed by the operator (eg. could be symbolic)
    Volumes         map[string]struct{}   // List of volumes (mounts) used for the container
    WorkingDir      string                // Current directory (PWD) in the command will be launched
    Entrypoint      strslice.StrSlice     // Entrypoint to run when starting the container
    NetworkDisabled bool                  `json:",omitempty"` // Is network disabled
    MacAddress      string                `json:",omitempty"` // Mac Address of the container
    OnBuild         []string              // ONBUILD metadata that were defined on the image Dockerfile
    Labels          map[string]string     // List of labels set to this container
    StopSignal      string                `json:",omitempty"` // Signal to stop a container
}
// Image represents a Docker image used by the builder.
type Image interface {
    ImageID() string
    RunConfig() *container.Config
}
// ID is the content-addressable ID of an image.
type ID digest.Digest

func (id ID) String() string {
    return digest.Digest(id).String()
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
func (b *Builder) commit(id string, autoCmd strslice.StrSlice, comment string) error {
    if b.disableCommit {
        return nil
    }
    if b.image == "" && !b.noBaseImage {
        return fmt.Errorf("Please provide a source image with `from` prior to commit")
    }
    b.runConfig.Image = b.image //赋值config.Image为目前为止的imageID

    if id == "" {
        cmd := b.runConfig.Cmd
        if runtime.GOOS != "windows" {
            b.runConfig.Cmd = strslice.StrSlice{"/bin/sh", "-c", "#(nop) " + comment}
        } else {
            b.runConfig.Cmd = strslice.StrSlice{"cmd", "/S /C", "REM (nop) " + comment}
        }
        defer func(cmd strslice.StrSlice) { b.runConfig.Cmd = cmd }(cmd)

        hit, err := b.probeCache()
        if err != nil {
            return err
        } else if hit {
            return nil
        }
        id, err = b.create()
        if err != nil {
            return err
        }
    }

    // Note: Actually copy the struct
    autoConfig := *b.runConfig
    autoConfig.Cmd = autoCmd

    commitCfg := &types.ContainerCommitConfig{
        Author: b.maintainer,
        Pause:  true,
        Config: &autoConfig,
    }

    // Commit the container
    imageID, err := b.docker.Commit(id, commitCfg)
    if err != nil {
        return err
    }

    b.image = imageID
    return nil
}
// probeCache checks if `b.docker` implements builder.ImageCache and image-caching
// is enabled (`b.UseCache`).
// If so attempts to look up the current `b.image` and `b.runConfig` pair with `b.docker`.
// If an image is found, probeCache returns `(true, nil)`.
// If no image is found, it returns `(false, nil)`.
// If there is any error, it returns `(false, err)`.
func (b *Builder) probeCache() (bool, error) {
    c, ok := b.docker.(builder.ImageCache)
    if !ok || b.options.NoCache || b.cacheBusted {
        return false, nil
    }
    cache, err := c.GetCachedImageOnBuild(b.image, b.runConfig)
    if err != nil {
        return false, err
    }
    if len(cache) == 0 {
        logrus.Debugf("[BUILDER] Cache miss: %s", b.runConfig.Cmd)
        b.cacheBusted = true
        return false, nil
    }

    fmt.Fprintf(b.Stdout, " ---> Using cache\n")
    logrus.Debugf("[BUILDER] Use cached version: %s", b.runConfig.Cmd)
    b.image = string(cache)

    return true, nil
}
// Commit creates a new filesystem image from the current state of a container.
// The image can optionally be tagged into a repository.
func (daemon *Daemon) Commit(name string, c *types.ContainerCommitConfig) (string, error) {
    container, err := daemon.GetContainer(name)
    if err != nil {
        return "", err
    }

    // It is not possible to commit a running container on Windows
    if runtime.GOOS == "windows" && container.IsRunning() {
        return "", fmt.Errorf("Windows does not support commit of a running container")
    }

    if c.Pause && !container.IsPaused() {
        daemon.containerPause(container)
        defer daemon.containerUnpause(container)
    }

    if c.MergeConfigs {
        if err := merge(c.Config, container.Config); err != nil {
            return "", err
        }
    }

    rwTar, err := daemon.exportContainerRw(container)
    if err != nil {
        return "", err
    }
    defer func() {
        if rwTar != nil {
            rwTar.Close()
        }
    }()

    var history []image.History
    rootFS := image.NewRootFS()

    if container.ImageID != "" {
        img, err := daemon.imageStore.Get(container.ImageID)
        if err != nil {
            return "", err
        }
        history = img.History
        rootFS = img.RootFS
    }

    l, err := daemon.layerStore.Register(rwTar, rootFS.ChainID())
    if err != nil {
        return "", err
    }
    defer layer.ReleaseAndLog(daemon.layerStore, l)

    h := image.History{
        Author:     c.Author,
        Created:    time.Now().UTC(),
        CreatedBy:  strings.Join(container.Config.Cmd, " "),
        Comment:    c.Comment,
        EmptyLayer: true,
    }

    if diffID := l.DiffID(); layer.DigestSHA256EmptyTar != diffID {
        h.EmptyLayer = false
        rootFS.Append(diffID)
    }

    history = append(history, h)

    config, err := json.Marshal(&image.Image{
        V1Image: image.V1Image{
            DockerVersion:   dockerversion.Version,
            Config:          c.Config,
            Architecture:    runtime.GOARCH,
            OS:              runtime.GOOS,
            Container:       container.ID,
            ContainerConfig: *container.Config,
            Author:          c.Author,
            Created:         h.Created,
        },
        RootFS:  rootFS,
        History: history,
    })

    if err != nil {
        return "", err
    }

    id, err := daemon.imageStore.Create(config)
    if err != nil {
        return "", err
    }

    if container.ImageID != "" {
        if err := daemon.imageStore.SetParent(id, container.ImageID); err != nil {
            return "", err
        }
    }

    if c.Repo != "" {
        newTag, err := reference.WithName(c.Repo) // todo: should move this to API layer
        if err != nil {
            return "", err
        }
        if c.Tag != "" {
            if newTag, err = reference.WithTag(newTag, c.Tag); err != nil {
                return "", err
            }
        }
        if err := daemon.TagImage(newTag, id.String()); err != nil {
            return "", err
        }
    }

    attributes := map[string]string{
        "comment": c.Comment,
    }
    daemon.LogContainerEventWithAttributes(container, "commit", attributes)
    return id.String(), nil
}
// Backend abstracts calls to a Docker Daemon.
type Backend interface {
    // TODO: use digest reference instead of name

    // GetImage looks up a Docker image referenced by `name`.
    GetImageOnBuild(name string) (Image, error)
    // Tag an image with newTag
    TagImage(newTag reference.Named, imageName string) error
    // Pull tells Docker to pull image referenced by `name`.
    PullOnBuild(ctx context.Context, name string, authConfigs map[string]types.AuthConfig, output io.Writer) (Image, error)
    // ContainerAttach attaches to container.
    ContainerAttachRaw(cID string, stdin io.ReadCloser, stdout, stderr io.Writer, stream bool) error
    // ContainerCreate creates a new Docker container and returns potential warnings
    ContainerCreate(types.ContainerCreateConfig) (types.ContainerCreateResponse, error)
    // ContainerRm removes a container specified by `id`.
    ContainerRm(name string, config *types.ContainerRmConfig) error
    // Commit creates a new Docker image from an existing Docker container.
    Commit(string, *types.ContainerCommitConfig) (string, error)
    // Kill stops the container execution abruptly.
    ContainerKill(containerID string, sig uint64) error
    // Start starts a new container
    ContainerStart(containerID string, hostConfig *container.HostConfig) error
    // ContainerWait stops processing until the given container is stopped.
    ContainerWait(containerID string, timeout time.Duration) (int, error)
    // ContainerUpdateCmd updates container.Path and container.Args
    ContainerUpdateCmdOnBuild(containerID string, cmd []string) error

    // ContainerCopy copies/extracts a source FileInfo to a destination path inside a container
    // specified by a container object.
    // TODO: make an Extract method instead of passing `decompress`
    // TODO: do not pass a FileInfo, instead refactor the archive package to export a Walk function that can be used
    // with Context.Walk
    //ContainerCopy(name string, res string) (io.ReadCloser, error)
    // TODO: use copyBackend api
    CopyOnBuild(containerID string, destPath string, src FileInfo, decompress bool) error
}
```

总结：

* `created`表示镜像创建时间

* `author`表示`MAINTAINER`

>The MAINTAINER instruction allows you to set the Author field of the generated images.

问题：

* `Config`与`ContainerConfig`区别？

* `Config.Image`表示什么？是`parent imageID`？若不是有什么作用？

>Image           string                // Name of the image as it was passed by the operator (eg. could be symbolic)

```go
// Commit creates a new filesystem image from the current state of a container.
// The image can optionally be tagged into a repository.
func (daemon *Daemon) Commit(name string, c *types.ContainerCommitConfig) (string, error) {
    ...
    id, err := daemon.imageStore.Create(config)
    ...
}

// Daemon holds information about the Docker daemon.
type Daemon struct {
    ID                        string
    repository                string
    containers                container.Store
    execCommands              *exec.Store
    referenceStore            reference.Store
    downloadManager           *xfer.LayerDownloadManager
    uploadManager             *xfer.LayerUploadManager
    distributionMetadataStore dmetadata.Store
    trustKey                  libtrust.PrivateKey
    idIndex                   *truncindex.TruncIndex
    configStore               *Config
    statsCollector            *statsCollector
    defaultLogConfig          containertypes.LogConfig
    RegistryService           *registry.Service
    EventsService             *events.Events
    netController             libnetwork.NetworkController
    volumes                   *store.VolumeStore
    discoveryWatcher          discoveryReloader
    root                      string
    seccompEnabled            bool
    shutdown                  bool
    uidMaps                   []idtools.IDMap
    gidMaps                   []idtools.IDMap
    layerStore                layer.Store
    imageStore                image.Store
    nameIndex                 *registrar.Registrar
    linkIndex                 *linkIndex
    containerd                libcontainerd.Client
    defaultIsolation          containertypes.Isolation // Default isolation mode on Windows
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
```

`config.Image`表示`docker build`过程中产生的`imageID`，例如（31f630c65071、2a1c91448f5f、21ed6e7fbb73）并不一定是父镜像`imageID`

```sh
$ docker build -t svendowideit/ambassador .
Sending build context to Docker daemon 15.36 kB
Step 1 : FROM alpine:3.2
 ---> 31f630c65071
Step 2 : MAINTAINER SvenDowideit@home.org.au
 ---> Using cache
 ---> 2a1c91448f5f
Step 3 : RUN apk update &&      apk add socat &&        rm -r /var/cache/
 ---> Using cache
 ---> 21ed6e7fbb73
Step 4 : CMD env | grep _TCP= | (sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat -t 100000000 TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3 \&/' && echo wait) | sh
 ---> Using cache
 ---> 7ea8aef582cc
Successfully built 7ea8aef582cc
```

* `Config`与`ContainerConfig`区别？

>// Config is the configuration of the container received from the client
   
>// ContainerConfig is the configuration of the container that is committed into the image

```go
// Commit creates a new filesystem image from the current state of a container.
// The image can optionally be tagged into a repository.
func (daemon *Daemon) Commit(name string, c *types.ContainerCommitConfig) (string, error) {
    container, err := daemon.GetContainer(name)
    if err != nil {
        return "", err
    }

    // It is not possible to commit a running container on Windows
    if runtime.GOOS == "windows" && container.IsRunning() {
        return "", fmt.Errorf("Windows does not support commit of a running container")
    }

    if c.Pause && !container.IsPaused() {
        daemon.containerPause(container)
        defer daemon.containerUnpause(container)
    }

    if c.MergeConfigs {
        if err := merge(c.Config, container.Config); err != nil {
            return "", err
        }
    }

    rwTar, err := daemon.exportContainerRw(container)
    if err != nil {
        return "", err
    }
    defer func() {
        if rwTar != nil {
            rwTar.Close()
        }
    }()

    var history []image.History
    rootFS := image.NewRootFS()

    if container.ImageID != "" {
        img, err := daemon.imageStore.Get(container.ImageID)
        if err != nil {
            return "", err
        }
        history = img.History
        rootFS = img.RootFS
    }

    l, err := daemon.layerStore.Register(rwTar, rootFS.ChainID())
    if err != nil {
        return "", err
    }
    defer layer.ReleaseAndLog(daemon.layerStore, l)

    h := image.History{
        Author:     c.Author,
        Created:    time.Now().UTC(),
        CreatedBy:  strings.Join(container.Config.Cmd, " "),
        Comment:    c.Comment,
        EmptyLayer: true,
    }

    if diffID := l.DiffID(); layer.DigestSHA256EmptyTar != diffID {
        h.EmptyLayer = false
        rootFS.Append(diffID)
    }

    history = append(history, h)

    config, err := json.Marshal(&image.Image{
        V1Image: image.V1Image{
            DockerVersion:   dockerversion.Version,
            Config:          c.Config,
            Architecture:    runtime.GOARCH,
            OS:              runtime.GOOS,
            Container:       container.ID,
            ContainerConfig: *container.Config,
            Author:          c.Author,
            Created:         h.Created,
        },
        RootFS:  rootFS,
        History: history,
    })

    if err != nil {
        return "", err
    }

    id, err := daemon.imageStore.Create(config)
    if err != nil {
        return "", err
    }

    if container.ImageID != "" {
        if err := daemon.imageStore.SetParent(id, container.ImageID); err != nil {
            return "", err
        }
    }

    if c.Repo != "" {
        newTag, err := reference.WithName(c.Repo) // todo: should move this to API layer
        if err != nil {
            return "", err
        }
        if c.Tag != "" {
            if newTag, err = reference.WithTag(newTag, c.Tag); err != nil {
                return "", err
            }
        }
        if err := daemon.TagImage(newTag, id.String()); err != nil {
            return "", err
        }
    }

    attributes := map[string]string{
        "comment": c.Comment,
    }
    daemon.LogContainerEventWithAttributes(container, "commit", attributes)
    return id.String(), nil
}
func (b *Builder) commit(id string, autoCmd strslice.StrSlice, comment string) error {
    if b.disableCommit {
        return nil
    }
    if b.image == "" && !b.noBaseImage {
        return fmt.Errorf("Please provide a source image with `from` prior to commit")
    }
    b.runConfig.Image = b.image

    if id == "" {
        cmd := b.runConfig.Cmd
        if runtime.GOOS != "windows" {
            b.runConfig.Cmd = strslice.StrSlice{"/bin/sh", "-c", "#(nop) " + comment}
        } else {
            b.runConfig.Cmd = strslice.StrSlice{"cmd", "/S /C", "REM (nop) " + comment}
        }
        defer func(cmd strslice.StrSlice) { b.runConfig.Cmd = cmd }(cmd)

        hit, err := b.probeCache()
        if err != nil {
            return err
        } else if hit {
            return nil
        }
        id, err = b.create()
        if err != nil {
            return err
        }
    }

    // Note: Actually copy the struct
    autoConfig := *b.runConfig
    autoConfig.Cmd = autoCmd

    commitCfg := &types.ContainerCommitConfig{
        Author: b.maintainer,
        Pause:  true,
        Config: &autoConfig,
    }

    // Commit the container
    imageID, err := b.docker.Commit(id, commitCfg)
    if err != nil {
        return err
    }

    b.image = imageID
    return nil
}
func (b *Builder) create() (string, error) {
    if b.image == "" && !b.noBaseImage {
        return "", fmt.Errorf("Please provide a source image with `from` prior to run")
    }
    b.runConfig.Image = b.image

    resources := container.Resources{
        CgroupParent: b.options.CgroupParent,
        CPUShares:    b.options.CPUShares,
        CPUPeriod:    b.options.CPUPeriod,
        CPUQuota:     b.options.CPUQuota,
        CpusetCpus:   b.options.CPUSetCPUs,
        CpusetMems:   b.options.CPUSetMems,
        Memory:       b.options.Memory,
        MemorySwap:   b.options.MemorySwap,
        Ulimits:      b.options.Ulimits,
    }

    // TODO: why not embed a hostconfig in builder?
    hostConfig := &container.HostConfig{
        Isolation: b.options.Isolation,
        ShmSize:   b.options.ShmSize,
        Resources: resources,
    }

    config := *b.runConfig

    // Create the container
    c, err := b.docker.ContainerCreate(types.ContainerCreateConfig{
        Config:     b.runConfig,
        HostConfig: hostConfig,
    })
    if err != nil {
        return "", err
    }
    for _, warning := range c.Warnings {
        fmt.Fprintf(b.Stdout, " ---> [Warning] %s\n", warning)
    }

    b.tmpContainers[c.ID] = struct{}{}
    fmt.Fprintf(b.Stdout, " ---> Running in %s\n", stringid.TruncateID(c.ID))

    // override the entry point that may have been picked up from the base image
    if err := b.docker.ContainerUpdateCmdOnBuild(c.ID, config.Cmd); err != nil {
        return "", err
    }

    return c.ID, nil
}
// ContainerCreateConfig is the parameter set to ContainerCreate()
type ContainerCreateConfig struct {
    Name             string
    Config           *container.Config
    HostConfig       *container.HostConfig
    NetworkingConfig *network.NetworkingConfig
    AdjustCPUShares  bool
}
// ContainerCreate creates a container.
func (daemon *Daemon) ContainerCreate(params types.ContainerCreateConfig) (types.ContainerCreateResponse, error) {
    if params.Config == nil {
        return types.ContainerCreateResponse{}, fmt.Errorf("Config cannot be empty in order to create a container")
    }

    warnings, err := daemon.verifyContainerSettings(params.HostConfig, params.Config, false)
    if err != nil {
        return types.ContainerCreateResponse{Warnings: warnings}, err
    }

    err = daemon.verifyNetworkingConfig(params.NetworkingConfig)
    if err != nil {
        return types.ContainerCreateResponse{}, err
    }

    if params.HostConfig == nil {
        params.HostConfig = &containertypes.HostConfig{}
    }
    err = daemon.adaptContainerSettings(params.HostConfig, params.AdjustCPUShares)
    if err != nil {
        return types.ContainerCreateResponse{Warnings: warnings}, err
    }

    container, err := daemon.create(params)
    if err != nil {
        return types.ContainerCreateResponse{Warnings: warnings}, daemon.imageNotExistToErrcode(err)
    }

    return types.ContainerCreateResponse{ID: container.ID, Warnings: warnings}, nil
}
// Create creates a new container from the given configuration with a given name.
func (daemon *Daemon) create(params types.ContainerCreateConfig) (retC *container.Container, retErr error) {
    var (
        container *container.Container
        img       *image.Image
        imgID     image.ID
        err       error
    )

    if params.Config.Image != "" {
        img, err = daemon.GetImage(params.Config.Image)
        if err != nil {
            return nil, err
        }
        imgID = img.ID()
    }

    if err := daemon.mergeAndVerifyConfig(params.Config, img); err != nil {
        return nil, err
    }

    if container, err = daemon.newContainer(params.Name, params.Config, imgID); err != nil {
        return nil, err
    }
    defer func() {
        if retErr != nil {
            if err := daemon.ContainerRm(container.ID, &types.ContainerRmConfig{ForceRemove: true}); err != nil {
                logrus.Errorf("Clean up Error! Cannot destroy container %s: %v", container.ID, err)
            }
        }
    }()

    if err := daemon.setSecurityOptions(container, params.HostConfig); err != nil {
        return nil, err
    }

    // Set RWLayer for container after mount labels have been set
    if err := daemon.setRWLayer(container); err != nil {
        return nil, err
    }

    if err := daemon.Register(container); err != nil {
        return nil, err
    }
    rootUID, rootGID, err := idtools.GetRootUIDGID(daemon.uidMaps, daemon.gidMaps)
    if err != nil {
        return nil, err
    }
    if err := idtools.MkdirAs(container.Root, 0700, rootUID, rootGID); err != nil {
        return nil, err
    }

    if err := daemon.setHostConfig(container, params.HostConfig); err != nil {
        return nil, err
    }
    defer func() {
        if retErr != nil {
            if err := daemon.removeMountPoints(container, true); err != nil {
                logrus.Error(err)
            }
        }
    }()

    if err := daemon.createContainerPlatformSpecificSettings(container, params.Config, params.HostConfig); err != nil {
        return nil, err
    }

    var endpointsConfigs map[string]*networktypes.EndpointSettings
    if params.NetworkingConfig != nil {
        endpointsConfigs = params.NetworkingConfig.EndpointsConfig
    }

    if err := daemon.updateContainerNetworkSettings(container, endpointsConfigs); err != nil {
        return nil, err
    }

    if err := container.ToDiskLocking(); err != nil {
        logrus.Errorf("Error saving new container to disk: %v", err)
        return nil, err
    }
    daemon.LogContainerEvent(container, "create")
    return container, nil
}
// GetContainer looks for a container using the provided information, which could be
// one of the following inputs from the caller:
//  - A full container ID, which will exact match a container in daemon's list
//  - A container name, which will only exact match via the GetByName() function
//  - A partial container ID prefix (e.g. short ID) of any length that is
//    unique enough to only return a single container object
//  If none of these searches succeed, an error is returned
func (daemon *Daemon) GetContainer(prefixOrName string) (*container.Container, error) {
    if containerByID := daemon.containers.Get(prefixOrName); containerByID != nil {
        // prefix is an exact match to a full container ID
        return containerByID, nil
    }

    // GetByName will match only an exact name provided; we ignore errors
    if containerByName, _ := daemon.GetByName(prefixOrName); containerByName != nil {
        // prefix is an exact match to a full container Name
        return containerByName, nil
    }

    containerID, indexError := daemon.idIndex.Get(prefixOrName)
    if indexError != nil {
        // When truncindex defines an error type, use that instead
        if indexError == truncindex.ErrNotExist {
            err := fmt.Errorf("No such container: %s", prefixOrName)
            return nil, errors.NewRequestNotFoundError(err)
        }
        return nil, indexError
    }
    return daemon.containers.Get(containerID), nil
}
// MAINTAINER some text <maybe@an.email.address>
//
// Sets the maintainer metadata.
func maintainer(b *Builder, args []string, attributes map[string]bool, original string) error {
    if len(args) != 1 {
        return errExactlyOneArgument("MAINTAINER")
    }

    if err := b.flags.Parse(); err != nil {
        return err
    }

    b.maintainer = args[0]
    return b.commit("", b.runConfig.Cmd, fmt.Sprintf("MAINTAINER %s", b.maintainer))
}
```