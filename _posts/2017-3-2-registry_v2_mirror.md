---
layout: post
title: Registry v2 mirror Support……
date: 2017-3-2 18:10:31
category: 技术
tags: Docker-registry
excerpt: Registry v2 mirror Support……
---

参考[registry源码v2.6.0](https://github.com/docker/distribution/tree/v2.6.0)

registry入口：docker/distribution/registry/registry.go

```go

// ServeCmd is a cobra command for running the registry.
var ServeCmd = &cobra.Command{
	Use:   "serve <config>",
	Short: "`serve` stores and distributes Docker images",
	Long:  "`serve` stores and distributes Docker images.",
	Run: func(cmd *cobra.Command, args []string) {

		// setup context
		ctx := context.WithVersion(context.Background(), version.Version)

		config, err := resolveConfiguration(args)
		if err != nil {
			fmt.Fprintf(os.Stderr, "configuration error: %v\n", err)
			cmd.Usage()
			os.Exit(1)
		}

		if config.HTTP.Debug.Addr != "" {
			go func(addr string) {
				log.Infof("debug server listening %v", addr)
				if err := http.ListenAndServe(addr, nil); err != nil {
					log.Fatalf("error listening on debug interface: %v", err)
				}
			}(config.HTTP.Debug.Addr)
		}

		registry, err := NewRegistry(ctx, config)
		if err != nil {
			log.Fatalln(err)
		}

		if err = registry.ListenAndServe(); err != nil {
			log.Fatalln(err)
		}
	},
}

```

解析配置文件：

```go
func resolveConfiguration(args []string) (*configuration.Configuration, error) {
	var configurationPath string

	if len(args) > 0 {
		configurationPath = args[0]
	} else if os.Getenv("REGISTRY_CONFIGURATION_PATH") != "" {
		configurationPath = os.Getenv("REGISTRY_CONFIGURATION_PATH")
	}

	if configurationPath == "" {
		return nil, fmt.Errorf("configuration path unspecified")
	}

	fp, err := os.Open(configurationPath)
	if err != nil {
		return nil, err
	}

	defer fp.Close()

	config, err := configuration.Parse(fp)
	if err != nil {
		return nil, fmt.Errorf("error parsing %s: %v", configurationPath, err)
	}

	return config, nil
}

// Parse parses an input configuration yaml document into a Configuration struct
// This should generally be capable of handling old configuration format versions
//
// Environment variables may be used to override configuration parameters other than version,
// following the scheme below:
// Configuration.Abc may be replaced by the value of REGISTRY_ABC,
// Configuration.Abc.Xyz may be replaced by the value of REGISTRY_ABC_XYZ, and so forth
func Parse(rd io.Reader) (*Configuration, error) {
	in, err := ioutil.ReadAll(rd)
	if err != nil {
		return nil, err
	}

	p := NewParser("registry", []VersionedParseInfo{
		{
			Version: MajorMinorVersion(0, 1),
			ParseAs: reflect.TypeOf(v0_1Configuration{}),
			ConversionFunc: func(c interface{}) (interface{}, error) {
				if v0_1, ok := c.(*v0_1Configuration); ok {
					if v0_1.Loglevel == Loglevel("") {
						v0_1.Loglevel = Loglevel("info")
					}
					if v0_1.Storage.Type() == "" {
						return nil, fmt.Errorf("No storage configuration provided")
					}
					return (*Configuration)(v0_1), nil
				}
				return nil, fmt.Errorf("Expected *v0_1Configuration, received %#v", c)
			},
		},
	})

	config := new(Configuration)
	err = p.Parse(in, config)
	if err != nil {
		return nil, err
	}

	return config, nil
}

// Configuration is a versioned registry configuration, intended to be provided by a yaml file, and
// optionally modified by environment variables.
//
// Note that yaml field names should never include _ characters, since this is the separator used
// in environment variable names.
type Configuration struct {
	// Version is the version which defines the format of the rest of the configuration
	Version Version `yaml:"version"`

	// Log supports setting various parameters related to the logging
	// subsystem.
	Log struct {
		// AccessLog configures access logging.
		AccessLog struct {
			// Disabled disables access logging.
			Disabled bool `yaml:"disabled,omitempty"`
		} `yaml:"accesslog,omitempty"`

		// Level is the granularity at which registry operations are logged.
		Level Loglevel `yaml:"level"`

		// Formatter overrides the default formatter with another. Options
		// include "text", "json" and "logstash".
		Formatter string `yaml:"formatter,omitempty"`

		// Fields allows users to specify static string fields to include in
		// the logger context.
		Fields map[string]interface{} `yaml:"fields,omitempty"`

		// Hooks allows users to configure the log hooks, to enabling the
		// sequent handling behavior, when defined levels of log message emit.
		Hooks []LogHook `yaml:"hooks,omitempty"`
	}

	// Loglevel is the level at which registry operations are logged. This is
	// deprecated. Please use Log.Level in the future.
	Loglevel Loglevel `yaml:"loglevel,omitempty"`

	// Storage is the configuration for the registry's storage driver
	Storage Storage `yaml:"storage"`

	// Auth allows configuration of various authorization methods that may be
	// used to gate requests.
	Auth Auth `yaml:"auth,omitempty"`

	// Middleware lists all middlewares to be used by the registry.
	Middleware map[string][]Middleware `yaml:"middleware,omitempty"`

	// Reporting is the configuration for error reporting
	Reporting Reporting `yaml:"reporting,omitempty"`

	// HTTP contains configuration parameters for the registry's http
	// interface.
	HTTP struct {
		// Addr specifies the bind address for the registry instance.
		Addr string `yaml:"addr,omitempty"`

		// Net specifies the net portion of the bind address. A default empty value means tcp.
		Net string `yaml:"net,omitempty"`

		// Host specifies an externally-reachable address for the registry, as a fully
		// qualified URL.
		Host string `yaml:"host,omitempty"`

		Prefix string `yaml:"prefix,omitempty"`

		// Secret specifies the secret key which HMAC tokens are created with.
		Secret string `yaml:"secret,omitempty"`

		// RelativeURLs specifies that relative URLs should be returned in
		// Location headers
		RelativeURLs bool `yaml:"relativeurls,omitempty"`

		// TLS instructs the http server to listen with a TLS configuration.
		// This only support simple tls configuration with a cert and key.
		// Mostly, this is useful for testing situations or simple deployments
		// that require tls. If more complex configurations are required, use
		// a proxy or make a proposal to add support here.
		TLS struct {
			// Certificate specifies the path to an x509 certificate file to
			// be used for TLS.
			Certificate string `yaml:"certificate,omitempty"`

			// Key specifies the path to the x509 key file, which should
			// contain the private portion for the file specified in
			// Certificate.
			Key string `yaml:"key,omitempty"`

			// Specifies the CA certs for client authentication
			// A file may contain multiple CA certificates encoded as PEM
			ClientCAs []string `yaml:"clientcas,omitempty"`

			// LetsEncrypt is used to configuration setting up TLS through
			// Let's Encrypt instead of manually specifying certificate and
			// key. If a TLS certificate is specified, the Let's Encrypt
			// section will not be used.
			LetsEncrypt struct {
				// CacheFile specifies cache file to use for lets encrypt
				// certificates and keys.
				CacheFile string `yaml:"cachefile,omitempty"`

				// Email is the email to use during Let's Encrypt registration
				Email string `yaml:"email,omitempty"`
			} `yaml:"letsencrypt,omitempty"`
		} `yaml:"tls,omitempty"`

		// Headers is a set of headers to include in HTTP responses. A common
		// use case for this would be security headers such as
		// Strict-Transport-Security. The map keys are the header names, and
		// the values are the associated header payloads.
		Headers http.Header `yaml:"headers,omitempty"`

		// Debug configures the http debug interface, if specified. This can
		// include services such as pprof, expvar and other data that should
		// not be exposed externally. Left disabled by default.
		Debug struct {
			// Addr specifies the bind address for the debug server.
			Addr string `yaml:"addr,omitempty"`
		} `yaml:"debug,omitempty"`

		// HTTP2 configuration options
		HTTP2 struct {
			// Specifies wether the registry should disallow clients attempting
			// to connect via http2. If set to true, only http/1.1 is supported.
			Disabled bool `yaml:"disabled,omitempty"`
		} `yaml:"http2,omitempty"`
	} `yaml:"http,omitempty"`

	// Notifications specifies configuration about various endpoint to which
	// registry events are dispatched.
	Notifications Notifications `yaml:"notifications,omitempty"`

	// Redis configures the redis pool available to the registry webapp.
	Redis struct {
		// Addr specifies the the redis instance available to the application.
		Addr string `yaml:"addr,omitempty"`

		// Password string to use when making a connection.
		Password string `yaml:"password,omitempty"`

		// DB specifies the database to connect to on the redis instance.
		DB int `yaml:"db,omitempty"`

		DialTimeout  time.Duration `yaml:"dialtimeout,omitempty"`  // timeout for connect
		ReadTimeout  time.Duration `yaml:"readtimeout,omitempty"`  // timeout for reads of data
		WriteTimeout time.Duration `yaml:"writetimeout,omitempty"` // timeout for writes of data

		// Pool configures the behavior of the redis connection pool.
		Pool struct {
			// MaxIdle sets the maximum number of idle connections.
			MaxIdle int `yaml:"maxidle,omitempty"`

			// MaxActive sets the maximum number of connections that should be
			// opened before blocking a connection request.
			MaxActive int `yaml:"maxactive,omitempty"`

			// IdleTimeout sets the amount time to wait before closing
			// inactive connections.
			IdleTimeout time.Duration `yaml:"idletimeout,omitempty"`
		} `yaml:"pool,omitempty"`
	} `yaml:"redis,omitempty"`

	Health Health `yaml:"health,omitempty"`

	Proxy Proxy `yaml:"proxy,omitempty"`

	// Compatibility is used for configurations of working with older or deprecated features.
	Compatibility struct {
		// Schema1 configures how schema1 manifests will be handled
		Schema1 struct {
			// TrustKey is the signing key to use for adding the signature to
			// schema1 manifests.
			TrustKey string `yaml:"signingkeyfile,omitempty"`
		} `yaml:"schema1,omitempty"`
	} `yaml:"compatibility,omitempty"`

	// Validation configures validation options for the registry.
	Validation struct {
		// Enabled enables the other options in this section.
		Enabled bool `yaml:"enabled,omitempty"`
		// Manifests configures manifest validation.
		Manifests struct {
			// URLs configures validation for URLs in pushed manifests.
			URLs struct {
				// Allow specifies regular expressions (https://godoc.org/regexp/syntax)
				// that URLs in pushed manifests must match.
				Allow []string `yaml:"allow,omitempty"`
				// Deny specifies regular expressions (https://godoc.org/regexp/syntax)
				// that URLs in pushed manifests must not match.
				Deny []string `yaml:"deny,omitempty"`
			} `yaml:"urls,omitempty"`
		} `yaml:"manifests,omitempty"`
	} `yaml:"validation,omitempty"`

	// Policy configures registry policy options.
	Policy struct {
		// Repository configures policies for repositories
		Repository struct {
			// Classes is a list of repository classes which the
			// registry allows content for. This class is matched
			// against the configuration media type inside uploaded
			// manifests. When non-empty, the registry will enforce
			// the class in authorized resources.
			Classes []string `yaml:"classes"`
		} `yaml:"repository,omitempty"`
	} `yaml:"policy,omitempty"`
}
```

配置文件示例：

```
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
proxy:
  remoteurl: https://registry-1.docker.io
  username: [username]
  password: [password]
```

解析完配置文件后，创建Registry Instance，如下：

```go
// A Registry represents a complete instance of the registry.
// TODO(aaronl): It might make sense for Registry to become an interface.
type Registry struct {
	config *configuration.Configuration
	app    *handlers.App
	server *http.Server
}

// NewRegistry creates a new registry from a context and configuration struct.
func NewRegistry(ctx context.Context, config *configuration.Configuration) (*Registry, error) {
	var err error
	ctx, err = configureLogging(ctx, config)
	if err != nil {
		return nil, fmt.Errorf("error configuring logger: %v", err)
	}

	// inject a logger into the uuid library. warns us if there is a problem
	// with uuid generation under low entropy.
	uuid.Loggerf = context.GetLogger(ctx).Warnf

	app := handlers.NewApp(ctx, config)
	// TODO(aaronl): The global scope of the health checks means NewRegistry
	// can only be called once per process.
	app.RegisterHealthChecks()
	handler := configureReporting(app)
	handler = alive("/", handler)
	handler = health.Handler(handler)
	handler = panicHandler(handler)
	if !config.Log.AccessLog.Disabled {
		handler = gorhandlers.CombinedLoggingHandler(os.Stdout, handler)
	}

	server := &http.Server{
		Handler: handler,
	}

	return &Registry{
		app:    app,
		config: config,
		server: server,
	}, nil
}
```

`NewApp`函数：

```go
// NewApp takes a configuration and returns a configured app, ready to serve
// requests. The app only implements ServeHTTP and can be wrapped in other
// handlers accordingly.
func NewApp(ctx context.Context, config *configuration.Configuration) *App {
	app := &App{
		Config:  config,
		Context: ctx,
		router:  v2.RouterWithPrefix(config.HTTP.Prefix),
		isCache: config.Proxy.RemoteURL != "",
	}

	// Register the handler dispatchers.
	app.register(v2.RouteNameBase, func(ctx *Context, r *http.Request) http.Handler {
		return http.HandlerFunc(apiBase)
	})
	app.register(v2.RouteNameManifest, imageManifestDispatcher)
	app.register(v2.RouteNameCatalog, catalogDispatcher)
	app.register(v2.RouteNameTags, tagsDispatcher)
	app.register(v2.RouteNameBlob, blobDispatcher)
	app.register(v2.RouteNameBlobUpload, blobUploadDispatcher)
	app.register(v2.RouteNameBlobUploadChunk, blobUploadDispatcher)

	// override the storage driver's UA string for registry outbound HTTP requests
	storageParams := config.Storage.Parameters()
	if storageParams == nil {
		storageParams = make(configuration.Parameters)
	}
	storageParams["useragent"] = fmt.Sprintf("docker-distribution/%s %s", version.Version, runtime.Version())

	var err error
	app.driver, err = factory.Create(config.Storage.Type(), storageParams)
	if err != nil {
		// TODO(stevvooe): Move the creation of a service into a protected
		// method, where this is created lazily. Its status can be queried via
		// a health check.
		panic(err)
	}

	purgeConfig := uploadPurgeDefaultConfig()
	if mc, ok := config.Storage["maintenance"]; ok {
		if v, ok := mc["uploadpurging"]; ok {
			purgeConfig, ok = v.(map[interface{}]interface{})
			if !ok {
				panic("uploadpurging config key must contain additional keys")
			}
		}
		if v, ok := mc["readonly"]; ok {
			readOnly, ok := v.(map[interface{}]interface{})
			if !ok {
				panic("readonly config key must contain additional keys")
			}
			if readOnlyEnabled, ok := readOnly["enabled"]; ok {
				app.readOnly, ok = readOnlyEnabled.(bool)
				if !ok {
					panic("readonly's enabled config key must have a boolean value")
				}
			}
		}
	}

	startUploadPurger(app, app.driver, ctxu.GetLogger(app), purgeConfig)

	app.driver, err = applyStorageMiddleware(app.driver, config.Middleware["storage"])
	if err != nil {
		panic(err)
	}

	app.configureSecret(config)
	app.configureEvents(config)
	app.configureRedis(config)
	app.configureLogHook(config)

	options := registrymiddleware.GetRegistryOptions()
	if config.Compatibility.Schema1.TrustKey != "" {
		app.trustKey, err = libtrust.LoadKeyFile(config.Compatibility.Schema1.TrustKey)
		if err != nil {
			panic(fmt.Sprintf(`could not load schema1 "signingkey" parameter: %v`, err))
		}
	} else {
		// Generate an ephemeral key to be used for signing converted manifests
		// for clients that don't support schema2.
		app.trustKey, err = libtrust.GenerateECP256PrivateKey()
		if err != nil {
			panic(err)
		}
	}

	options = append(options, storage.Schema1SigningKey(app.trustKey))

	if config.HTTP.Host != "" {
		u, err := url.Parse(config.HTTP.Host)
		if err != nil {
			panic(fmt.Sprintf(`could not parse http "host" parameter: %v`, err))
		}
		app.httpHost = *u
	}

	if app.isCache {
		options = append(options, storage.DisableDigestResumption)
	}

	// configure deletion
	if d, ok := config.Storage["delete"]; ok {
		e, ok := d["enabled"]
		if ok {
			if deleteEnabled, ok := e.(bool); ok && deleteEnabled {
				options = append(options, storage.EnableDelete)
			}
		}
	}

	// configure redirects
	var redirectDisabled bool
	if redirectConfig, ok := config.Storage["redirect"]; ok {
		v := redirectConfig["disable"]
		switch v := v.(type) {
		case bool:
			redirectDisabled = v
		default:
			panic(fmt.Sprintf("invalid type for redirect config: %#v", redirectConfig))
		}
	}
	if redirectDisabled {
		ctxu.GetLogger(app).Infof("backend redirection disabled")
	} else {
		options = append(options, storage.EnableRedirect)
	}

	// configure validation
	if config.Validation.Enabled {
		if len(config.Validation.Manifests.URLs.Allow) == 0 && len(config.Validation.Manifests.URLs.Deny) == 0 {
			// If Allow and Deny are empty, allow nothing.
			options = append(options, storage.ManifestURLsAllowRegexp(regexp.MustCompile("^$")))
		} else {
			if len(config.Validation.Manifests.URLs.Allow) > 0 {
				for i, s := range config.Validation.Manifests.URLs.Allow {
					// Validate via compilation.
					if _, err := regexp.Compile(s); err != nil {
						panic(fmt.Sprintf("validation.manifests.urls.allow: %s", err))
					}
					// Wrap with non-capturing group.
					config.Validation.Manifests.URLs.Allow[i] = fmt.Sprintf("(?:%s)", s)
				}
				re := regexp.MustCompile(strings.Join(config.Validation.Manifests.URLs.Allow, "|"))
				options = append(options, storage.ManifestURLsAllowRegexp(re))
			}
			if len(config.Validation.Manifests.URLs.Deny) > 0 {
				for i, s := range config.Validation.Manifests.URLs.Deny {
					// Validate via compilation.
					if _, err := regexp.Compile(s); err != nil {
						panic(fmt.Sprintf("validation.manifests.urls.deny: %s", err))
					}
					// Wrap with non-capturing group.
					config.Validation.Manifests.URLs.Deny[i] = fmt.Sprintf("(?:%s)", s)
				}
				re := regexp.MustCompile(strings.Join(config.Validation.Manifests.URLs.Deny, "|"))
				options = append(options, storage.ManifestURLsDenyRegexp(re))
			}
		}
	}

	// configure storage caches
	if cc, ok := config.Storage["cache"]; ok {
		v, ok := cc["blobdescriptor"]
		if !ok {
			// Backwards compatible: "layerinfo" == "blobdescriptor"
			v = cc["layerinfo"]
		}

		switch v {
		case "redis":
			if app.redis == nil {
				panic("redis configuration required to use for layerinfo cache")
			}
			cacheProvider := rediscache.NewRedisBlobDescriptorCacheProvider(app.redis)
			localOptions := append(options, storage.BlobDescriptorCacheProvider(cacheProvider))
			app.registry, err = storage.NewRegistry(app, app.driver, localOptions...)
			if err != nil {
				panic("could not create registry: " + err.Error())
			}
			ctxu.GetLogger(app).Infof("using redis blob descriptor cache")
		case "inmemory":
			cacheProvider := memorycache.NewInMemoryBlobDescriptorCacheProvider()
			localOptions := append(options, storage.BlobDescriptorCacheProvider(cacheProvider))
			app.registry, err = storage.NewRegistry(app, app.driver, localOptions...)
			if err != nil {
				panic("could not create registry: " + err.Error())
			}
			ctxu.GetLogger(app).Infof("using inmemory blob descriptor cache")
		default:
			if v != "" {
				ctxu.GetLogger(app).Warnf("unknown cache type %q, caching disabled", config.Storage["cache"])
			}
		}
	}

	if app.registry == nil {
		// configure the registry if no cache section is available.
		app.registry, err = storage.NewRegistry(app.Context, app.driver, options...)
		if err != nil {
			panic("could not create registry: " + err.Error())
		}
	}

	app.registry, err = applyRegistryMiddleware(app, app.registry, config.Middleware["registry"])
	if err != nil {
		panic(err)
	}

	authType := config.Auth.Type()

	if authType != "" {
		accessController, err := auth.GetAccessController(config.Auth.Type(), config.Auth.Parameters())
		if err != nil {
			panic(fmt.Sprintf("unable to configure authorization (%s): %v", authType, err))
		}
		app.accessController = accessController
		ctxu.GetLogger(app).Debugf("configured %q access controller", authType)
	}

	// configure as a pull through cache
	if config.Proxy.RemoteURL != "" {
		app.registry, err = proxy.NewRegistryPullThroughCache(ctx, app.registry, app.driver, config.Proxy)
		if err != nil {
			panic(err.Error())
		}
		app.isCache = true
		ctxu.GetLogger(app).Info("Registry configured as a proxy cache to ", config.Proxy.RemoteURL)
	}

	return app
}

// App is a global registry application object. Shared resources can be placed
// on this object that will be accessible from all requests. Any writable
// fields should be protected.
type App struct {
	context.Context

	Config *configuration.Configuration

	router           *mux.Router                 // main application router, configured with dispatchers
	driver           storagedriver.StorageDriver // driver maintains the app global storage driver instance.
	registry         distribution.Namespace      // registry is the primary registry backend for the app instance.
	accessController auth.AccessController       // main access controller for application

	// httpHost is a parsed representation of the http.host parameter from
	// the configuration. Only the Scheme and Host fields are used.
	httpHost url.URL

	// events contains notification related configuration.
	events struct {
		sink   notifications.Sink
		source notifications.SourceRecord
	}

	redis *redis.Pool

	// trustKey is a deprecated key used to sign manifests converted to
	// schema1 for backward compatibility. It should not be used for any
	// other purposes.
	trustKey libtrust.PrivateKey

	// isCache is true if this registry is configured as a pull through cache
	isCache bool

	// readOnly is true if the registry is in a read-only maintenance mode
	readOnly bool
}
```

configure as a pull through cache：

```go
// configure as a pull through cache
if config.Proxy.RemoteURL != "" {
    app.registry, err = proxy.NewRegistryPullThroughCache(ctx, app.registry, app.driver, config.Proxy)
    if err != nil {
        panic(err.Error())
    }
    app.isCache = true
    ctxu.GetLogger(app).Info("Registry configured as a proxy cache to ", config.Proxy.RemoteURL)
}

// NewRegistryPullThroughCache creates a registry acting as a pull through cache
func NewRegistryPullThroughCache(ctx context.Context, registry distribution.Namespace, driver driver.StorageDriver, config configuration.Proxy) (distribution.Namespace, error) {
	remoteURL, err := url.Parse(config.RemoteURL)
	if err != nil {
		return nil, err
	}

	v := storage.NewVacuum(ctx, driver)
	s := scheduler.New(ctx, driver, "/scheduler-state.json")
	s.OnBlobExpire(func(ref reference.Reference) error {
		var r reference.Canonical
		var ok bool
		if r, ok = ref.(reference.Canonical); !ok {
			return fmt.Errorf("unexpected reference type : %T", ref)
		}

		repo, err := registry.Repository(ctx, r)
		if err != nil {
			return err
		}

		blobs := repo.Blobs(ctx)

		// Clear the repository reference and descriptor caches
		err = blobs.Delete(ctx, r.Digest())
		if err != nil {
			return err
		}

		err = v.RemoveBlob(r.Digest().String())
		if err != nil {
			return err
		}

		return nil
	})

	s.OnManifestExpire(func(ref reference.Reference) error {
		var r reference.Canonical
		var ok bool
		if r, ok = ref.(reference.Canonical); !ok {
			return fmt.Errorf("unexpected reference type : %T", ref)
		}

		repo, err := registry.Repository(ctx, r)
		if err != nil {
			return err
		}

		manifests, err := repo.Manifests(ctx)
		if err != nil {
			return err
		}
		err = manifests.Delete(ctx, r.Digest())
		if err != nil {
			return err
		}
		return nil
	})

	err = s.Start()
	if err != nil {
		return nil, err
	}

	cs, err := configureAuth(config.Username, config.Password, config.RemoteURL)
	if err != nil {
		return nil, err
	}

	return &proxyingRegistry{
		embedded:  registry,
		scheduler: s,
		remoteURL: *remoteURL,
		authChallenger: &remoteAuthChallenger{
			remoteURL: *remoteURL,
			cm:        challenge.NewSimpleManager(),
			cs:        cs,
		},
	}, nil
}
```

执行健康检查（对应health）：

```go
// TODO(aaronl): The global scope of the health checks means NewRegistry
// can only be called once per process.
app.RegisterHealthChecks()

// RegisterHealthChecks is an awful hack to defer health check registration
// control to callers. This should only ever be called once per registry
// process, typically in a main function. The correct way would be register
// health checks outside of app, since multiple apps may exist in the same
// process. Because the configuration and app are tightly coupled,
// implementing this properly will require a refactor. This method may panic
// if called twice in the same process.
func (app *App) RegisterHealthChecks(healthRegistries ...*health.Registry) {
	if len(healthRegistries) > 1 {
		panic("RegisterHealthChecks called with more than one registry")
	}
	healthRegistry := health.DefaultRegistry
	if len(healthRegistries) == 1 {
		healthRegistry = healthRegistries[0]
	}

	if app.Config.Health.StorageDriver.Enabled {
		interval := app.Config.Health.StorageDriver.Interval
		if interval == 0 {
			interval = defaultCheckInterval
		}

		storageDriverCheck := func() error {
			_, err := app.driver.List(app, "/") // "/" should always exist
			return err                          // any error will be treated as failure
		}

		if app.Config.Health.StorageDriver.Threshold != 0 {
			healthRegistry.RegisterPeriodicThresholdFunc("storagedriver_"+app.Config.Storage.Type(), interval, app.Config.Health.StorageDriver.Threshold, storageDriverCheck)
		} else {
			healthRegistry.RegisterPeriodicFunc("storagedriver_"+app.Config.Storage.Type(), interval, storageDriverCheck)
		}
	}

	for _, fileChecker := range app.Config.Health.FileCheckers {
		interval := fileChecker.Interval
		if interval == 0 {
			interval = defaultCheckInterval
		}
		ctxu.GetLogger(app).Infof("configuring file health check path=%s, interval=%d", fileChecker.File, interval/time.Second)
		healthRegistry.Register(fileChecker.File, health.PeriodicChecker(checks.FileChecker(fileChecker.File), interval))
	}

	for _, httpChecker := range app.Config.Health.HTTPCheckers {
		interval := httpChecker.Interval
		if interval == 0 {
			interval = defaultCheckInterval
		}

		statusCode := httpChecker.StatusCode
		if statusCode == 0 {
			statusCode = 200
		}

		checker := checks.HTTPChecker(httpChecker.URI, statusCode, httpChecker.Timeout, httpChecker.Headers)

		if httpChecker.Threshold != 0 {
			ctxu.GetLogger(app).Infof("configuring HTTP health check uri=%s, interval=%d, threshold=%d", httpChecker.URI, interval/time.Second, httpChecker.Threshold)
			healthRegistry.Register(httpChecker.URI, health.PeriodicThresholdChecker(checker, interval, httpChecker.Threshold))
		} else {
			ctxu.GetLogger(app).Infof("configuring HTTP health check uri=%s, interval=%d", httpChecker.URI, interval/time.Second)
			healthRegistry.Register(httpChecker.URI, health.PeriodicChecker(checker, interval))
		}
	}

	for _, tcpChecker := range app.Config.Health.TCPCheckers {
		interval := tcpChecker.Interval
		if interval == 0 {
			interval = defaultCheckInterval
		}

		checker := checks.TCPChecker(tcpChecker.Addr, tcpChecker.Timeout)

		if tcpChecker.Threshold != 0 {
			ctxu.GetLogger(app).Infof("configuring TCP health check addr=%s, interval=%d, threshold=%d", tcpChecker.Addr, interval/time.Second, tcpChecker.Threshold)
			healthRegistry.Register(tcpChecker.Addr, health.PeriodicThresholdChecker(checker, interval, tcpChecker.Threshold))
		} else {
			ctxu.GetLogger(app).Infof("configuring TCP health check addr=%s, interval=%d", tcpChecker.Addr, interval/time.Second)
			healthRegistry.Register(tcpChecker.Addr, health.PeriodicChecker(checker, interval))
		}
	}
}
// RegisterPeriodicThresholdFunc allows the convenience of registering a
// PeriodicChecker from an arbitrary func() error.
func (registry *Registry) RegisterPeriodicThresholdFunc(name string, period time.Duration, threshold int, check CheckFunc) {
	registry.Register(name, PeriodicThresholdChecker(CheckFunc(check), period, threshold))
}

// PeriodicThresholdChecker wraps an updater to provide a periodic checker that
// uses a threshold before it changes status
func PeriodicThresholdChecker(check Checker, period time.Duration, threshold int) Checker {
	tu := NewThresholdStatusUpdater(threshold)
	go func() {
		t := time.NewTicker(period)
		for {
			<-t.C
			tu.Update(check.Check())
		}
	}()

	return tu
}
```

生成registry instance后，执行registry instance，监听端口，处理请求：

```go
// ListenAndServe runs the registry's HTTP server.
func (registry *Registry) ListenAndServe() error {
	config := registry.config

	ln, err := listener.NewListener(config.HTTP.Net, config.HTTP.Addr)
	if err != nil {
		return err
	}

	if config.HTTP.TLS.Certificate != "" || config.HTTP.TLS.LetsEncrypt.CacheFile != "" {
		tlsConf := &tls.Config{
			ClientAuth:               tls.NoClientCert,
			NextProtos:               nextProtos(config),
			MinVersion:               tls.VersionTLS10,
			PreferServerCipherSuites: true,
			CipherSuites: []uint16{
				tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
				tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
				tls.TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,
				tls.TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA,
				tls.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,
				tls.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,
				tls.TLS_RSA_WITH_AES_128_CBC_SHA,
				tls.TLS_RSA_WITH_AES_256_CBC_SHA,
			},
		}

		if config.HTTP.TLS.LetsEncrypt.CacheFile != "" {
			if config.HTTP.TLS.Certificate != "" {
				return fmt.Errorf("cannot specify both certificate and Let's Encrypt")
			}
			var m letsencrypt.Manager
			if err := m.CacheFile(config.HTTP.TLS.LetsEncrypt.CacheFile); err != nil {
				return err
			}
			if !m.Registered() {
				if err := m.Register(config.HTTP.TLS.LetsEncrypt.Email, nil); err != nil {
					return err
				}
			}
			tlsConf.GetCertificate = m.GetCertificate
		} else {
			tlsConf.Certificates = make([]tls.Certificate, 1)
			tlsConf.Certificates[0], err = tls.LoadX509KeyPair(config.HTTP.TLS.Certificate, config.HTTP.TLS.Key)
			if err != nil {
				return err
			}
		}

		if len(config.HTTP.TLS.ClientCAs) != 0 {
			pool := x509.NewCertPool()

			for _, ca := range config.HTTP.TLS.ClientCAs {
				caPem, err := ioutil.ReadFile(ca)
				if err != nil {
					return err
				}

				if ok := pool.AppendCertsFromPEM(caPem); !ok {
					return fmt.Errorf("Could not add CA to pool")
				}
			}

			for _, subj := range pool.Subjects() {
				context.GetLogger(registry.app).Debugf("CA Subject: %s", string(subj))
			}

			tlsConf.ClientAuth = tls.RequireAndVerifyClientCert
			tlsConf.ClientCAs = pool
		}

		ln = tls.NewListener(ln, tlsConf)
		context.GetLogger(registry.app).Infof("listening on %v, tls", ln.Addr())
	} else {
		context.GetLogger(registry.app).Infof("listening on %v", ln.Addr())
	}

	return registry.server.Serve(ln)
}

// NewListener announces on laddr and net. Accepted values of the net are
// 'unix' and 'tcp'
func NewListener(net, laddr string) (net.Listener, error) {
	switch net {
	case "unix":
		return newUnixListener(laddr)
	case "tcp", "": // an empty net means tcp
		return newTCPListener(laddr)
	default:
		return nil, fmt.Errorf("unknown address type %s", net)
	}
}
func newTCPListener(laddr string) (net.Listener, error) {
	ln, err := net.Listen("tcp", laddr)
	if err != nil {
		return nil, err
	}

	return tcpKeepAliveListener{ln.(*net.TCPListener)}, nil
}
```

分析请求：`GET /v2/library/centos/blobs/sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4`

```go
// register a handler with the application, by route name. The handler will be
// passed through the application filters and context will be constructed at
// request time.
func (app *App) register(routeName string, dispatch dispatchFunc) {

	// TODO(stevvooe): This odd dispatcher/route registration is by-product of
	// some limitations in the gorilla/mux router. We are using it to keep
	// routing consistent between the client and server, but we may want to
	// replace it with manual routing and structure-based dispatch for better
	// control over the request execution.

	app.router.GetRoute(routeName).Handler(app.dispatcher(dispatch))
}

app.register(v2.RouteNameBlob, blobDispatcher)

// dispatcher returns a handler that constructs a request specific context and
// handler, using the dispatch factory function.
func (app *App) dispatcher(dispatch dispatchFunc) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		for headerName, headerValues := range app.Config.HTTP.Headers {
			for _, value := range headerValues {
				w.Header().Add(headerName, value)
			}
		}

		context := app.context(w, r)

		if err := app.authorized(w, r, context); err != nil {
			ctxu.GetLogger(context).Warnf("error authorizing context: %v", err)
			return
		}

		// Add username to request logging
		context.Context = ctxu.WithLogger(context.Context, ctxu.GetLogger(context.Context, auth.UserNameKey))

		if app.nameRequired(r) {
			nameRef, err := reference.ParseNamed(getName(context))
			if err != nil {
				ctxu.GetLogger(context).Errorf("error parsing reference from context: %v", err)
				context.Errors = append(context.Errors, distribution.ErrRepositoryNameInvalid{
					Name:   getName(context),
					Reason: err,
				})
				if err := errcode.ServeJSON(w, context.Errors); err != nil {
					ctxu.GetLogger(context).Errorf("error serving error json: %v (from %v)", err, context.Errors)
				}
				return
			}
			repository, err := app.registry.Repository(context, nameRef)

			if err != nil {
				ctxu.GetLogger(context).Errorf("error resolving repository: %v", err)

				switch err := err.(type) {
				case distribution.ErrRepositoryUnknown:
					context.Errors = append(context.Errors, v2.ErrorCodeNameUnknown.WithDetail(err))
				case distribution.ErrRepositoryNameInvalid:
					context.Errors = append(context.Errors, v2.ErrorCodeNameInvalid.WithDetail(err))
				case errcode.Error:
					context.Errors = append(context.Errors, err)
				}

				if err := errcode.ServeJSON(w, context.Errors); err != nil {
					ctxu.GetLogger(context).Errorf("error serving error json: %v (from %v)", err, context.Errors)
				}
				return
			}

			// assign and decorate the authorized repository with an event bridge.
			context.Repository = notifications.Listen(
				repository,
				app.eventBridge(context, r))

			context.Repository, err = applyRepoMiddleware(app, context.Repository, app.Config.Middleware["repository"])
			if err != nil {
				ctxu.GetLogger(context).Errorf("error initializing repository middleware: %v", err)
				context.Errors = append(context.Errors, errcode.ErrorCodeUnknown.WithDetail(err))

				if err := errcode.ServeJSON(w, context.Errors); err != nil {
					ctxu.GetLogger(context).Errorf("error serving error json: %v (from %v)", err, context.Errors)
				}
				return
			}
		}

		dispatch(context, r).ServeHTTP(w, r)
		// Automated error response handling here. Handlers may return their
		// own errors if they need different behavior (such as range errors
		// for layer upload).
		if context.Errors.Len() > 0 {
			if err := errcode.ServeJSON(w, context.Errors); err != nil {
				ctxu.GetLogger(context).Errorf("error serving error json: %v (from %v)", err, context.Errors)
			}

			app.logError(context, context.Errors)
		}
	})
}
// context constructs the context object for the application. This only be
// called once per request.
func (app *App) context(w http.ResponseWriter, r *http.Request) *Context {
	ctx := defaultContextManager.context(app, w, r)
	ctx = ctxu.WithVars(ctx, r)
	ctx = ctxu.WithLogger(ctx, ctxu.GetLogger(ctx,
		"vars.name",
		"vars.reference",
		"vars.digest",
		"vars.uuid"))

	context := &Context{
		App:     app,
		Context: ctx,
	}

	if app.httpHost.Scheme != "" && app.httpHost.Host != "" {
		// A "host" item in the configuration takes precedence over
		// X-Forwarded-Proto and X-Forwarded-Host headers, and the
		// hostname in the request.
		context.urlBuilder = v2.NewURLBuilder(&app.httpHost, false)
	} else {
		context.urlBuilder = v2.NewURLBuilderFromRequest(r, app.Config.HTTP.RelativeURLs)
	}

	return context
}
// nameRequired returns true if the route requires a name.
func (app *App) nameRequired(r *http.Request) bool {
	route := mux.CurrentRoute(r)
	routeName := route.GetName()
	return route == nil || (routeName != v2.RouteNameBase && routeName != v2.RouteNameCatalog)
}
// blobDispatcher uses the request context to build a blobHandler.
func blobDispatcher(ctx *Context, r *http.Request) http.Handler {
	dgst, err := getDigest(ctx)
	if err != nil {

		if err == errDigestNotAvailable {
			return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
				ctx.Errors = append(ctx.Errors, v2.ErrorCodeDigestInvalid.WithDetail(err))
			})
		}

		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx.Errors = append(ctx.Errors, v2.ErrorCodeDigestInvalid.WithDetail(err))
		})
	}

	blobHandler := &blobHandler{
		Context: ctx,
		Digest:  dgst,
	}

	mhandler := handlers.MethodHandler{
		"GET":  http.HandlerFunc(blobHandler.GetBlob),
		"HEAD": http.HandlerFunc(blobHandler.GetBlob),
	}

	if !ctx.readOnly {
		mhandler["DELETE"] = http.HandlerFunc(blobHandler.DeleteBlob)
	}

	return mhandler
}
// GetBlob fetches the binary data from backend storage returns it in the
// response.
func (bh *blobHandler) GetBlob(w http.ResponseWriter, r *http.Request) {
	context.GetLogger(bh).Debug("GetBlob")
	blobs := bh.Repository.Blobs(bh)
	desc, err := blobs.Stat(bh, bh.Digest)
	if err != nil {
		if err == distribution.ErrBlobUnknown {
			bh.Errors = append(bh.Errors, v2.ErrorCodeBlobUnknown.WithDetail(bh.Digest))
		} else {
			bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		}
		return
	}

	if err := blobs.ServeBlob(bh, w, r, desc.Digest); err != nil {
		context.GetLogger(bh).Debugf("unexpected error getting blob HTTP handler: %v", err)
		bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		return
	}
}
func (pr *proxyingRegistry) Repository(ctx context.Context, name reference.Named) (distribution.Repository, error) {
	c := pr.authChallenger

	tr := transport.NewTransport(http.DefaultTransport,
		auth.NewAuthorizer(c.challengeManager(), auth.NewTokenHandler(http.DefaultTransport, c.credentialStore(), name.Name(), "pull")))

	localRepo, err := pr.embedded.Repository(ctx, name)
	if err != nil {
		return nil, err
	}
	localManifests, err := localRepo.Manifests(ctx, storage.SkipLayerVerification())
	if err != nil {
		return nil, err
	}

	remoteRepo, err := client.NewRepository(ctx, name, pr.remoteURL.String(), tr)
	if err != nil {
		return nil, err
	}

	remoteManifests, err := remoteRepo.Manifests(ctx)
	if err != nil {
		return nil, err
	}

	return &proxiedRepository{
		blobStore: &proxyBlobStore{
			localStore:     localRepo.Blobs(ctx),
			remoteStore:    remoteRepo.Blobs(ctx),
			scheduler:      pr.scheduler,
			repositoryName: name,
			authChallenger: pr.authChallenger,
		},
		manifests: &proxyManifestStore{
			repositoryName:  name,
			localManifests:  localManifests, // Options?
			remoteManifests: remoteManifests,
			ctx:             ctx,
			scheduler:       pr.scheduler,
			authChallenger:  pr.authChallenger,
		},
		name: name,
		tags: &proxyTagService{
			localTags:      localRepo.Tags(ctx),
			remoteTags:     remoteRepo.Tags(ctx),
			authChallenger: pr.authChallenger,
		},
	}, nil
}

func (pr *proxiedRepository) Blobs(ctx context.Context) distribution.BlobStore {
	return pr.blobStore
}

type proxyBlobStore struct {
	localStore     distribution.BlobStore
	remoteStore    distribution.BlobService
	scheduler      *scheduler.TTLExpirationScheduler
	repositoryName reference.Named
	authChallenger authChallenger
}
// BlobServer can serve blobs via http.
type BlobServer interface {
	// ServeBlob attempts to serve the blob, identifed by dgst, via http. The
	// service may decide to redirect the client elsewhere or serve the data
	// directly.
	//
	// This handler only issues successful responses, such as 2xx or 3xx,
	// meaning it serves data or issues a redirect. If the blob is not
	// available, an error will be returned and the caller may still issue a
	// response.
	//
	// The implementation may serve the same blob from a different digest
	// domain. The appropriate headers will be set for the blob, unless they
	// have already been set by the caller.
	ServeBlob(ctx context.Context, w http.ResponseWriter, r *http.Request, dgst digest.Digest) error
}
func (pbs *proxyBlobStore) ServeBlob(ctx context.Context, w http.ResponseWriter, r *http.Request, dgst digest.Digest) error {
	served, err := pbs.serveLocal(ctx, w, r, dgst)
	if err != nil {
		context.GetLogger(ctx).Errorf("Error serving blob from local storage: %s", err.Error())
		return err
	}

	if served {
		return nil
	}

	if err := pbs.authChallenger.tryEstablishChallenges(ctx); err != nil {
		return err
	}

	mu.Lock()
	_, ok := inflight[dgst]
	if ok {
		mu.Unlock()
		_, err := pbs.copyContent(ctx, dgst, w)
		return err
	}
	inflight[dgst] = struct{}{}
	mu.Unlock()

	go func(dgst digest.Digest) {
		if err := pbs.storeLocal(ctx, dgst); err != nil {
			context.GetLogger(ctx).Errorf("Error committing to storage: %s", err.Error())
		}

		blobRef, err := reference.WithDigest(pbs.repositoryName, dgst)
		if err != nil {
			context.GetLogger(ctx).Errorf("Error creating reference: %s", err)
			return
		}

		pbs.scheduler.AddBlob(blobRef, repositoryTTL)
	}(dgst)

	_, err = pbs.copyContent(ctx, dgst, w)
	if err != nil {
		return err
	}
	return nil
}
```

现在来看`ServeBlob`函数逻辑（具体mirror逻辑）：

```go
// GetBlob fetches the binary data from backend storage returns it in the
// response.
func (bh *blobHandler) GetBlob(w http.ResponseWriter, r *http.Request) {
	context.GetLogger(bh).Debug("GetBlob")
	blobs := bh.Repository.Blobs(bh)
	desc, err := blobs.Stat(bh, bh.Digest)
	if err != nil {
		if err == distribution.ErrBlobUnknown {
			bh.Errors = append(bh.Errors, v2.ErrorCodeBlobUnknown.WithDetail(bh.Digest))
		} else {
			bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		}
		return
	}

	if err := blobs.ServeBlob(bh, w, r, desc.Digest); err != nil {
		context.GetLogger(bh).Debugf("unexpected error getting blob HTTP handler: %v", err)
		bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		return
	}
}

func (pbs *proxyBlobStore) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	desc, err := pbs.localStore.Stat(ctx, dgst)
	if err == nil {
		return desc, err
	}

	if err != distribution.ErrBlobUnknown {
		return distribution.Descriptor{}, err
	}

	if err := pbs.authChallenger.tryEstablishChallenges(ctx); err != nil {
		return distribution.Descriptor{}, err
	}

	return pbs.remoteStore.Stat(ctx, dgst)
}

func (pr *proxyingRegistry) Repository(ctx context.Context, name reference.Named) (distribution.Repository, error) {
	c := pr.authChallenger

	tr := transport.NewTransport(http.DefaultTransport,
		auth.NewAuthorizer(c.challengeManager(), auth.NewTokenHandler(http.DefaultTransport, c.credentialStore(), name.Name(), "pull")))

	localRepo, err := pr.embedded.Repository(ctx, name)
	if err != nil {
		return nil, err
	}
	localManifests, err := localRepo.Manifests(ctx, storage.SkipLayerVerification())
	if err != nil {
		return nil, err
	}

	remoteRepo, err := client.NewRepository(ctx, name, pr.remoteURL.String(), tr)
	if err != nil {
		return nil, err
	}

	remoteManifests, err := remoteRepo.Manifests(ctx)
	if err != nil {
		return nil, err
	}

	return &proxiedRepository{
		blobStore: &proxyBlobStore{
			localStore:     localRepo.Blobs(ctx),
			remoteStore:    remoteRepo.Blobs(ctx),
			scheduler:      pr.scheduler,
			repositoryName: name,
			authChallenger: pr.authChallenger,
		},
		manifests: &proxyManifestStore{
			repositoryName:  name,
			localManifests:  localManifests, // Options?
			remoteManifests: remoteManifests,
			ctx:             ctx,
			scheduler:       pr.scheduler,
			authChallenger:  pr.authChallenger,
		},
		name: name,
		tags: &proxyTagService{
			localTags:      localRepo.Tags(ctx),
			remoteTags:     remoteRepo.Tags(ctx),
			authChallenger: pr.authChallenger,
		},
	}, nil
}

// NewRegistry creates a new registry instance from the provided driver. The
// resulting registry may be shared by multiple goroutines but is cheap to
// allocate. If the Redirect option is specified, the backend blob server will
// attempt to use (StorageDriver).URLFor to serve all blobs.
func NewRegistry(ctx context.Context, driver storagedriver.StorageDriver, options ...RegistryOption) (distribution.Namespace, error) {
	// create global statter
	statter := &blobStatter{
		driver: driver,
	}

	bs := &blobStore{
		driver:  driver,
		statter: statter,
	}

	registry := &registry{
		blobStore: bs,
		blobServer: &blobServer{
			driver:  driver,
			statter: statter,
			pathFn:  bs.path,
		},
		statter:                statter,
		resumableDigestEnabled: true,
	}

	for _, option := range options {
		if err := option(registry); err != nil {
			return nil, err
		}
	}

	return registry, nil
}
// registry is the top-level implementation of Registry for use in the storage
// package. All instances should descend from this object.
type registry struct {
	blobStore                    *blobStore
	blobServer                   *blobServer
	statter                      *blobStatter // global statter service.
	blobDescriptorCacheProvider  cache.BlobDescriptorCacheProvider
	deleteEnabled                bool
	resumableDigestEnabled       bool
	schema1SigningKey            libtrust.PrivateKey
	blobDescriptorServiceFactory distribution.BlobDescriptorServiceFactory
	manifestURLs                 manifestURLs
}
// Repository returns an instance of the repository tied to the registry.
// Instances should not be shared between goroutines but are cheap to
// allocate. In general, they should be request scoped.
func (reg *registry) Repository(ctx context.Context, canonicalName reference.Named) (distribution.Repository, error) {
	var descriptorCache distribution.BlobDescriptorService
	if reg.blobDescriptorCacheProvider != nil {
		var err error
		descriptorCache, err = reg.blobDescriptorCacheProvider.RepositoryScoped(canonicalName.Name())
		if err != nil {
			return nil, err
		}
	}

	return &repository{
		ctx:             ctx,
		registry:        reg,
		name:            canonicalName,
		descriptorCache: descriptorCache,
	}, nil
}
// repository provides name-scoped access to various services.
type repository struct {
	*registry
	ctx             context.Context
	name            reference.Named
	descriptorCache distribution.BlobDescriptorService
}
// Blobs returns an instance of the BlobStore. Instantiation is cheap and
// may be context sensitive in the future. The instance should be used similar
// to a request local.
func (repo *repository) Blobs(ctx context.Context) distribution.BlobStore {
	var statter distribution.BlobDescriptorService = &linkedBlobStatter{
		blobStore:   repo.blobStore,
		repository:  repo,
		linkPathFns: []linkPathFunc{blobLinkPath},
	}

	if repo.descriptorCache != nil {
		statter = cache.NewCachedBlobStatter(repo.descriptorCache, statter)
	}

	if repo.registry.blobDescriptorServiceFactory != nil {
		statter = repo.registry.blobDescriptorServiceFactory.BlobAccessController(statter)
	}

	return &linkedBlobStore{
		registry:             repo.registry,
		blobStore:            repo.blobStore,
		blobServer:           repo.blobServer,
		blobAccessController: statter,
		repository:           repo,
		ctx:                  ctx,

		// TODO(stevvooe): linkPath limits this blob store to only layers.
		// This instance cannot be used for manifest checks.
		linkPathFns:            []linkPathFunc{blobLinkPath},
		deleteEnabled:          repo.registry.deleteEnabled,
		resumableDigestEnabled: repo.resumableDigestEnabled,
	}
}

// blobLinkPath provides the path to the blob link, also known as layers.
func blobLinkPath(name string, dgst digest.Digest) (string, error) {
	return pathFor(layerLinkPathSpec{name: name, digest: dgst})
}

// blobLinkPathSpec specifies a path for a blob link, which is a file with a
// blob id. The blob link will contain a content addressable blob id reference
// into the blob store. The format of the contents is as follows:
//
// 	<algorithm>:<hex digest of layer data>
//
// The following example of the file contents is more illustrative:
//
// 	sha256:96443a84ce518ac22acb2e985eda402b58ac19ce6f91980bde63726a79d80b36
//
// This  indicates that there is a blob with the id/digest, calculated via
// sha256 that can be fetched from the blob store.
type layerLinkPathSpec struct {
	name   string
	digest digest.Digest
}

const (
	storagePathVersion = "v2"                // fixed storage layout version
	storagePathRoot    = "/docker/registry/" // all driver paths have a prefix

	// TODO(stevvooe): Get rid of the "storagePathRoot". Initially, we though
	// storage path root would configurable for all drivers through this
	// package. In reality, we've found it simpler to do this on a per driver
	// basis.
)

// pathFor maps paths based on "object names" and their ids. The "object
// names" mapped by are internal to the storage system.
//
// The path layout in the storage backend is roughly as follows:
//
//		<root>/v2
//			-> repositories/
// 				-><name>/
// 					-> _manifests/
// 						revisions
//							-> <manifest digest path>
//								-> link
// 						tags/<tag>
//							-> current/link
// 							-> index
//								-> <algorithm>/<hex digest>/link
// 					-> _layers/
// 						<layer links to blob store>
// 					-> _uploads/<id>
// 						data
// 						startedat
// 						hashstates/<algorithm>/<offset>
//			-> blob/<algorithm>
//				<split directory content addressable storage>
//
// The storage backend layout is broken up into a content-addressable blob
// store and repositories. The content-addressable blob store holds most data
// throughout the backend, keyed by algorithm and digests of the underlying
// content. Access to the blob store is controlled through links from the
// repository to blobstore.
//
// A repository is made up of layers, manifests and tags. The layers component
// is just a directory of layers which are "linked" into a repository. A layer
// can only be accessed through a qualified repository name if it is linked in
// the repository. Uploads of layers are managed in the uploads directory,
// which is key by upload id. When all data for an upload is received, the
// data is moved into the blob store and the upload directory is deleted.
// Abandoned uploads can be garbage collected by reading the startedat file
// and removing uploads that have been active for longer than a certain time.
//
// The third component of the repository directory is the manifests store,
// which is made up of a revision store and tag store. Manifests are stored in
// the blob store and linked into the revision store.
// While the registry can save all revisions of a manifest, no relationship is
// implied as to the ordering of changes to a manifest. The tag store provides
// support for name, tag lookups of manifests, using "current/link" under a
// named tag directory. An index is maintained to support deletions of all
// revisions of a given manifest tag.
//
// We cover the path formats implemented by this path mapper below.
//
//	Manifests:
//
// 	manifestRevisionsPathSpec:      <root>/v2/repositories/<name>/_manifests/revisions/
// 	manifestRevisionPathSpec:      <root>/v2/repositories/<name>/_manifests/revisions/<algorithm>/<hex digest>/
// 	manifestRevisionLinkPathSpec:  <root>/v2/repositories/<name>/_manifests/revisions/<algorithm>/<hex digest>/link
//
//	Tags:
//
// 	manifestTagsPathSpec:                  <root>/v2/repositories/<name>/_manifests/tags/
// 	manifestTagPathSpec:                   <root>/v2/repositories/<name>/_manifests/tags/<tag>/
// 	manifestTagCurrentPathSpec:            <root>/v2/repositories/<name>/_manifests/tags/<tag>/current/link
// 	manifestTagIndexPathSpec:              <root>/v2/repositories/<name>/_manifests/tags/<tag>/index/
// 	manifestTagIndexEntryPathSpec:         <root>/v2/repositories/<name>/_manifests/tags/<tag>/index/<algorithm>/<hex digest>/
// 	manifestTagIndexEntryLinkPathSpec:     <root>/v2/repositories/<name>/_manifests/tags/<tag>/index/<algorithm>/<hex digest>/link
//
// 	Blobs:
//
// 	layerLinkPathSpec:            <root>/v2/repositories/<name>/_layers/<algorithm>/<hex digest>/link
//
//	Uploads:
//
// 	uploadDataPathSpec:             <root>/v2/repositories/<name>/_uploads/<id>/data
// 	uploadStartedAtPathSpec:        <root>/v2/repositories/<name>/_uploads/<id>/startedat
// 	uploadHashStatePathSpec:        <root>/v2/repositories/<name>/_uploads/<id>/hashstates/<algorithm>/<offset>
//
//	Blob Store:
//
//	blobsPathSpec:                  <root>/v2/blobs/
// 	blobPathSpec:                   <root>/v2/blobs/<algorithm>/<first two hex bytes of digest>/<hex digest>
// 	blobDataPathSpec:               <root>/v2/blobs/<algorithm>/<first two hex bytes of digest>/<hex digest>/data
// 	blobMediaTypePathSpec:               <root>/v2/blobs/<algorithm>/<first two hex bytes of digest>/<hex digest>/data
//
// For more information on the semantic meaning of each path and their
// contents, please see the path spec documentation.
func pathFor(spec pathSpec) (string, error) {

	// Switch on the path object type and return the appropriate path. At
	// first glance, one may wonder why we don't use an interface to
	// accomplish this. By keep the formatting separate from the pathSpec, we
	// keep separate the path generation componentized. These specs could be
	// passed to a completely different mapper implementation and generate a
	// different set of paths.
	//
	// For example, imagine migrating from one backend to the other: one could
	// build a filesystem walker that converts a string path in one version,
	// to an intermediate path object, than can be consumed and mapped by the
	// other version.

	rootPrefix := []string{storagePathRoot, storagePathVersion}
	repoPrefix := append(rootPrefix, "repositories")

	switch v := spec.(type) {

	case manifestRevisionsPathSpec:
		return path.Join(append(repoPrefix, v.name, "_manifests", "revisions")...), nil

	case manifestRevisionPathSpec:
		components, err := digestPathComponents(v.revision, false)
		if err != nil {
			return "", err
		}

		return path.Join(append(append(repoPrefix, v.name, "_manifests", "revisions"), components...)...), nil
	case manifestRevisionLinkPathSpec:
		root, err := pathFor(manifestRevisionPathSpec{
			name:     v.name,
			revision: v.revision,
		})

		if err != nil {
			return "", err
		}

		return path.Join(root, "link"), nil
	case manifestTagsPathSpec:
		return path.Join(append(repoPrefix, v.name, "_manifests", "tags")...), nil
	case manifestTagPathSpec:
		root, err := pathFor(manifestTagsPathSpec{
			name: v.name,
		})

		if err != nil {
			return "", err
		}

		return path.Join(root, v.tag), nil
	case manifestTagCurrentPathSpec:
		root, err := pathFor(manifestTagPathSpec{
			name: v.name,
			tag:  v.tag,
		})

		if err != nil {
			return "", err
		}

		return path.Join(root, "current", "link"), nil
	case manifestTagIndexPathSpec:
		root, err := pathFor(manifestTagPathSpec{
			name: v.name,
			tag:  v.tag,
		})

		if err != nil {
			return "", err
		}

		return path.Join(root, "index"), nil
	case manifestTagIndexEntryLinkPathSpec:
		root, err := pathFor(manifestTagIndexEntryPathSpec{
			name:     v.name,
			tag:      v.tag,
			revision: v.revision,
		})

		if err != nil {
			return "", err
		}

		return path.Join(root, "link"), nil
	case manifestTagIndexEntryPathSpec:
		root, err := pathFor(manifestTagIndexPathSpec{
			name: v.name,
			tag:  v.tag,
		})

		if err != nil {
			return "", err
		}

		components, err := digestPathComponents(v.revision, false)
		if err != nil {
			return "", err
		}

		return path.Join(root, path.Join(components...)), nil
	case layerLinkPathSpec:
		components, err := digestPathComponents(v.digest, false)
		if err != nil {
			return "", err
		}

		// TODO(stevvooe): Right now, all blobs are linked under "_layers". If
		// we have future migrations, we may want to rename this to "_blobs".
		// A migration strategy would simply leave existing items in place and
		// write the new paths, commit a file then delete the old files.

		blobLinkPathComponents := append(repoPrefix, v.name, "_layers")

		return path.Join(path.Join(append(blobLinkPathComponents, components...)...), "link"), nil
	case blobsPathSpec:
		blobsPathPrefix := append(rootPrefix, "blobs")
		return path.Join(blobsPathPrefix...), nil
	case blobPathSpec:
		components, err := digestPathComponents(v.digest, true)
		if err != nil {
			return "", err
		}

		blobPathPrefix := append(rootPrefix, "blobs")
		return path.Join(append(blobPathPrefix, components...)...), nil
	case blobDataPathSpec:
		components, err := digestPathComponents(v.digest, true)
		if err != nil {
			return "", err
		}

		components = append(components, "data")
		blobPathPrefix := append(rootPrefix, "blobs")
		return path.Join(append(blobPathPrefix, components...)...), nil

	case uploadDataPathSpec:
		return path.Join(append(repoPrefix, v.name, "_uploads", v.id, "data")...), nil
	case uploadStartedAtPathSpec:
		return path.Join(append(repoPrefix, v.name, "_uploads", v.id, "startedat")...), nil
	case uploadHashStatePathSpec:
		offset := fmt.Sprintf("%d", v.offset)
		if v.list {
			offset = "" // Limit to the prefix for listing offsets.
		}
		return path.Join(append(repoPrefix, v.name, "_uploads", v.id, "hashstates", string(v.alg), offset)...), nil
	case repositoriesRootPathSpec:
		return path.Join(repoPrefix...), nil
	default:
		// TODO(sday): This is an internal error. Ensure it doesn't escape (panic?).
		return "", fmt.Errorf("unknown path spec: %#v", v)
	}
}

// digestPathComponents provides a consistent path breakdown for a given
// digest. For a generic digest, it will be as follows:
//
// 	<algorithm>/<hex digest>
//
// If multilevel is true, the first two bytes of the digest will separate
// groups of digest folder. It will be as follows:
//
// 	<algorithm>/<first two bytes of digest>/<full digest>
//
func digestPathComponents(dgst digest.Digest, multilevel bool) ([]string, error) {
	if err := dgst.Validate(); err != nil {
		return nil, err
	}

	algorithm := blobAlgorithmReplacer.Replace(string(dgst.Algorithm()))
	hex := dgst.Hex()
	prefix := []string{algorithm}

	var suffix []string

	if multilevel {
		suffix = append(suffix, hex[:2])
	}

	suffix = append(suffix, hex)

	return append(prefix, suffix...), nil
}

// linkedBlobStore provides a full BlobService that namespaces the blobs to a
// given repository. Effectively, it manages the links in a given repository
// that grant access to the global blob store.
type linkedBlobStore struct {
	*blobStore
	registry               *registry
	blobServer             distribution.BlobServer
	blobAccessController   distribution.BlobDescriptorService
	repository             distribution.Repository
	ctx                    context.Context // only to be used where context can't come through method args
	deleteEnabled          bool
	resumableDigestEnabled bool

	// linkPathFns specifies one or more path functions allowing one to
	// control the repository blob link set to which the blob store
	// dispatches. This is required because manifest and layer blobs have not
	// yet been fully merged. At some point, this functionality should be
	// removed the blob links folder should be merged. The first entry is
	// treated as the "canonical" link location and will be used for writes.
	linkPathFns []linkPathFunc

	// linkDirectoryPathSpec locates the root directories in which one might find links
	linkDirectoryPathSpec pathSpec
}
func (lbs *linkedBlobStore) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	return lbs.blobAccessController.Stat(ctx, dgst)
}
type linkedBlobStatter struct {
	*blobStore
	repository distribution.Repository

	// linkPathFns specifies one or more path functions allowing one to
	// control the repository blob link set to which the blob store
	// dispatches. This is required because manifest and layer blobs have not
	// yet been fully merged. At some point, this functionality should be
	// removed an the blob links folder should be merged. The first entry is
	// treated as the "canonical" link location and will be used for writes.
	linkPathFns []linkPathFunc
}
func (lbs *linkedBlobStatter) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	var (
		found  bool
		target digest.Digest
	)

	// try the many link path functions until we get success or an error that
	// is not PathNotFoundError.
	for _, linkPathFn := range lbs.linkPathFns {
		var err error
		target, err = lbs.resolveWithLinkFunc(ctx, dgst, linkPathFn)

		if err == nil {
			found = true
			break // success!
		}

		switch err := err.(type) {
		case driver.PathNotFoundError:
			// do nothing, just move to the next linkPathFn
		default:
			return distribution.Descriptor{}, err
		}
	}

	if !found {
		return distribution.Descriptor{}, distribution.ErrBlobUnknown
	}

	if target != dgst {
		// Track when we are doing cross-digest domain lookups. ie, sha512 to sha256.
		context.GetLogger(ctx).Warnf("looking up blob with canonical target: %v -> %v", dgst, target)
	}

	// TODO(stevvooe): Look up repository local mediatype and replace that on
	// the returned descriptor.

	return lbs.blobStore.statter.Stat(ctx, target)
}
// resolveTargetWithFunc allows us to read a link to a resource with different
// linkPathFuncs to let us try a few different paths before returning not
// found.
func (lbs *linkedBlobStatter) resolveWithLinkFunc(ctx context.Context, dgst digest.Digest, linkPathFn linkPathFunc) (digest.Digest, error) {
	blobLinkPath, err := linkPathFn(lbs.repository.Named().Name(), dgst)
	if err != nil {
		return "", err
	}

	return lbs.blobStore.readlink(ctx, blobLinkPath)
}
// readlink returns the linked digest at path.
func (bs *blobStore) readlink(ctx context.Context, path string) (digest.Digest, error) {
	content, err := bs.driver.GetContent(ctx, path)
	if err != nil {
		return "", err
	}

	linked, err := digest.ParseDigest(string(content))
	if err != nil {
		return "", err
	}

	return linked, nil
}
// ParseDigest parses s and returns the validated digest object. An error will
// be returned if the format is invalid.
func ParseDigest(s string) (Digest, error) {
	d := Digest(s)

	return d, d.Validate()
}
// Digest allows simple protection of hex formatted digest strings, prefixed
// by their algorithm. Strings of type Digest have some guarantee of being in
// the correct format and it provides quick access to the components of a
// digest string.
//
// The following is an example of the contents of Digest types:
//
// 	sha256:7173b809ca12ec5dee4506cd86be934c4596dd234ee82c0662eac04a8c2c71dc
//
// This allows to abstract the digest behind this type and work only in those
// terms.
type Digest string


type blobStatter struct {
	driver driver.StorageDriver
}

// Stat implements BlobStatter.Stat by returning the descriptor for the blob
// in the main blob store. If this method returns successfully, there is
// strong guarantee that the blob exists and is available.
func (bs *blobStatter) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	path, err := pathFor(blobDataPathSpec{
		digest: dgst,
	})

	if err != nil {
		return distribution.Descriptor{}, err
	}

	fi, err := bs.driver.Stat(ctx, path)
	if err != nil {
		switch err := err.(type) {
		case driver.PathNotFoundError:
			return distribution.Descriptor{}, distribution.ErrBlobUnknown
		default:
			return distribution.Descriptor{}, err
		}
	}

	if fi.IsDir() {
		// NOTE(stevvooe): This represents a corruption situation. Somehow, we
		// calculated a blob path and then detected a directory. We log the
		// error and then error on the side of not knowing about the blob.
		context.GetLogger(ctx).Warnf("blob path should not be a directory: %q", path)
		return distribution.Descriptor{}, distribution.ErrBlobUnknown
	}

	// TODO(stevvooe): Add method to resolve the mediatype. We can store and
	// cache a "global" media type for the blob, even if a specific repo has a
	// mediatype that overrides the main one.

	return distribution.Descriptor{
		Size: fi.Size(),

		// NOTE(stevvooe): The central blob store firewalls media types from
		// other users. The caller should look this up and override the value
		// for the specific repository.
		MediaType: "application/octet-stream",
		Digest:    dgst,
	}, nil
}
// Descriptor describes targeted content. Used in conjunction with a blob
// store, a descriptor can be used to fetch, store and target any kind of
// blob. The struct also describes the wire protocol format. Fields should
// only be added but never changed.
type Descriptor struct {
	// MediaType describe the type of the content. All text based formats are
	// encoded as utf-8.
	MediaType string `json:"mediaType,omitempty"`

	// Size in bytes of content.
	Size int64 `json:"size,omitempty"`

	// Digest uniquely identifies the content. A byte stream can be verified
	// against against this digest.
	Digest digest.Digest `json:"digest,omitempty"`

	// URLs contains the source URLs of this content.
	URLs []string `json:"urls,omitempty"`

	// NOTE: Before adding a field here, please ensure that all
	// other options have been exhausted. Much of the type relationships
	// depend on the simplicity of this type.
}

// blobDataPathSpec contains the path for the registry global blob store. For
// now, this contains layer data, exclusively.
type blobDataPathSpec struct {
	digest digest.Digest
}

// 	blobDataPathSpec:               <root>/v2/blobs/<algorithm>/<first two hex bytes of digest>/<hex digest>/data

case blobDataPathSpec:
    components, err := digestPathComponents(v.digest, true)
    if err != nil {
        return "", err
    }

    components = append(components, "data")
    blobPathPrefix := append(rootPrefix, "blobs")
    return path.Join(append(blobPathPrefix, components...)...), nil

// Stat retrieves the FileInfo for the given path, including the current size
// in bytes and the creation time.
func (d *driver) Stat(ctx context.Context, subPath string) (storagedriver.FileInfo, error) {
	fullPath := d.fullPath(subPath)

	fi, err := os.Stat(fullPath)
	if err != nil {
		if os.IsNotExist(err) {
			return nil, storagedriver.PathNotFoundError{Path: subPath}
		}

		return nil, err
	}

	return fileInfo{
		path:     subPath,
		FileInfo: fi,
	}, nil
}
// fullPath returns the absolute path of a key within the Driver's storage.
func (d *driver) fullPath(subPath string) string {
	return path.Join(d.rootDirectory, subPath)
}
type fileInfo struct {
	os.FileInfo
	path string
}
```

```
// 	Blobs:
//
// 	layerLinkPathSpec:            <root>/v2/repositories/<name>/_layers/<algorithm>/<hex digest>/link
//
```

也即先验证是否存在`<root>/v2/repositories/<name>/_layers/<algorithm>/<hex digest>/link`文件，若存在（文件内容应该与digest相同），则获取文件`<root>/v2/blobs/<algorithm>/<first two hex bytes of digest>/<hex digest>/data`信息并构建`Descriptor`结构返回；否则进行如下操作：

```go
// GetBlob fetches the binary data from backend storage returns it in the
// response.
func (bh *blobHandler) GetBlob(w http.ResponseWriter, r *http.Request) {
	context.GetLogger(bh).Debug("GetBlob")
	blobs := bh.Repository.Blobs(bh)
	desc, err := blobs.Stat(bh, bh.Digest)
	if err != nil {
		if err == distribution.ErrBlobUnknown {
			bh.Errors = append(bh.Errors, v2.ErrorCodeBlobUnknown.WithDetail(bh.Digest))
		} else {
			bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		}
		return
	}

	if err := blobs.ServeBlob(bh, w, r, desc.Digest); err != nil {
		context.GetLogger(bh).Debugf("unexpected error getting blob HTTP handler: %v", err)
		bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		return
	}
}

func (pbs *proxyBlobStore) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	desc, err := pbs.localStore.Stat(ctx, dgst)
	if err == nil {
		return desc, err
	}

	if err != distribution.ErrBlobUnknown {
		return distribution.Descriptor{}, err
	}

	if err := pbs.authChallenger.tryEstablishChallenges(ctx); err != nil {
		return distribution.Descriptor{}, err
	}

	return pbs.remoteStore.Stat(ctx, dgst)
}
// NewRegistryPullThroughCache creates a registry acting as a pull through cache
func NewRegistryPullThroughCache(ctx context.Context, registry distribution.Namespace, driver driver.StorageDriver, config configuration.Proxy) (distribution.Namespace, error) {
	remoteURL, err := url.Parse(config.RemoteURL)
	if err != nil {
		return nil, err
	}

	v := storage.NewVacuum(ctx, driver)
	s := scheduler.New(ctx, driver, "/scheduler-state.json")
	s.OnBlobExpire(func(ref reference.Reference) error {
		var r reference.Canonical
		var ok bool
		if r, ok = ref.(reference.Canonical); !ok {
			return fmt.Errorf("unexpected reference type : %T", ref)
		}

		repo, err := registry.Repository(ctx, r)
		if err != nil {
			return err
		}

		blobs := repo.Blobs(ctx)

		// Clear the repository reference and descriptor caches
		err = blobs.Delete(ctx, r.Digest())
		if err != nil {
			return err
		}

		err = v.RemoveBlob(r.Digest().String())
		if err != nil {
			return err
		}

		return nil
	})

	s.OnManifestExpire(func(ref reference.Reference) error {
		var r reference.Canonical
		var ok bool
		if r, ok = ref.(reference.Canonical); !ok {
			return fmt.Errorf("unexpected reference type : %T", ref)
		}

		repo, err := registry.Repository(ctx, r)
		if err != nil {
			return err
		}

		manifests, err := repo.Manifests(ctx)
		if err != nil {
			return err
		}
		err = manifests.Delete(ctx, r.Digest())
		if err != nil {
			return err
		}
		return nil
	})

	err = s.Start()
	if err != nil {
		return nil, err
	}

	cs, err := configureAuth(config.Username, config.Password, config.RemoteURL)
	if err != nil {
		return nil, err
	}

	return &proxyingRegistry{
		embedded:  registry,
		scheduler: s,
		remoteURL: *remoteURL,
		authChallenger: &remoteAuthChallenger{
			remoteURL: *remoteURL,
			cm:        challenge.NewSimpleManager(),
			cs:        cs,
		},
	}, nil
}
func (pr *proxyingRegistry) Repository(ctx context.Context, name reference.Named) (distribution.Repository, error) {
	c := pr.authChallenger

	tr := transport.NewTransport(http.DefaultTransport,
		auth.NewAuthorizer(c.challengeManager(), auth.NewTokenHandler(http.DefaultTransport, c.credentialStore(), name.Name(), "pull")))

	localRepo, err := pr.embedded.Repository(ctx, name)
	if err != nil {
		return nil, err
	}
	localManifests, err := localRepo.Manifests(ctx, storage.SkipLayerVerification())
	if err != nil {
		return nil, err
	}

	remoteRepo, err := client.NewRepository(ctx, name, pr.remoteURL.String(), tr)
	if err != nil {
		return nil, err
	}

	remoteManifests, err := remoteRepo.Manifests(ctx)
	if err != nil {
		return nil, err
	}

	return &proxiedRepository{
		blobStore: &proxyBlobStore{
			localStore:     localRepo.Blobs(ctx),
			remoteStore:    remoteRepo.Blobs(ctx),
			scheduler:      pr.scheduler,
			repositoryName: name,
			authChallenger: pr.authChallenger,
		},
		manifests: &proxyManifestStore{
			repositoryName:  name,
			localManifests:  localManifests, // Options?
			remoteManifests: remoteManifests,
			ctx:             ctx,
			scheduler:       pr.scheduler,
			authChallenger:  pr.authChallenger,
		},
		name: name,
		tags: &proxyTagService{
			localTags:      localRepo.Tags(ctx),
			remoteTags:     remoteRepo.Tags(ctx),
			authChallenger: pr.authChallenger,
		},
	}, nil
}
type remoteAuthChallenger struct {
	remoteURL url.URL
	sync.Mutex
	cm challenge.Manager
	cs auth.CredentialStore
}
// tryEstablishChallenges will attempt to get a challenge type for the upstream if none currently exist
func (r *remoteAuthChallenger) tryEstablishChallenges(ctx context.Context) error {
	r.Lock()
	defer r.Unlock()

	remoteURL := r.remoteURL
	remoteURL.Path = "/v2/"
	challenges, err := r.cm.GetChallenges(remoteURL)
	if err != nil {
		return err
	}

	if len(challenges) > 0 {
		return nil
	}

	// establish challenge type with upstream
	if err := ping(r.cm, remoteURL.String(), challengeHeader); err != nil {
		return err
	}

	context.GetLogger(ctx).Infof("Challenge established with upstream : %s %s", remoteURL, r.cm)
	return nil
}
// NewSimpleManager returns an instance of
// Manger which only maps endpoints to challenges
// based on the responses which have been added the
// manager. The simple manager will make no attempt to
// perform requests on the endpoints or cache the responses
// to a backend.
func NewSimpleManager() Manager {
	return &simpleManager{
		Challanges: make(map[string][]Challenge),
	}
}
type simpleManager struct {
	sync.RWMutex
	Challanges map[string][]Challenge
} 
func (m *simpleManager) GetChallenges(endpoint url.URL) ([]Challenge, error) {
	normalizeURL(&endpoint)

	m.RLock()
	defer m.RUnlock()
	challenges := m.Challanges[endpoint.String()]
	return challenges, nil
}
func ping(manager challenge.Manager, endpoint, versionHeader string) error {
	resp, err := http.Get(endpoint)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if err := manager.AddResponse(resp); err != nil {
		return err
	}

	return nil
}
func (m *simpleManager) AddResponse(resp *http.Response) error {
	challenges := ResponseChallenges(resp)
	if resp.Request == nil {
		return fmt.Errorf("missing request reference")
	}
	urlCopy := url.URL{
		Path:   resp.Request.URL.Path,
		Host:   resp.Request.URL.Host,
		Scheme: resp.Request.URL.Scheme,
	}
	normalizeURL(&urlCopy)

	m.Lock()
	defer m.Unlock()
	m.Challanges[urlCopy.String()] = challenges
	return nil
}
// ResponseChallenges returns a list of authorization challenges
// for the given http Response. Challenges are only checked if
// the response status code was a 401.
func ResponseChallenges(resp *http.Response) []Challenge {
	if resp.StatusCode == http.StatusUnauthorized {
		// Parse the WWW-Authenticate Header and store the challenges
		// on this endpoint object.
		return parseAuthHeader(resp.Header)
	}

	return nil
}
func parseAuthHeader(header http.Header) []Challenge {
	challenges := []Challenge{}
	for _, h := range header[http.CanonicalHeaderKey("WWW-Authenticate")] {
		v, p := parseValueAndParams(h)
		if v != "" {
			challenges = append(challenges, Challenge{Scheme: v, Parameters: p})
		}
	}
	return challenges
}
func parseValueAndParams(header string) (value string, params map[string]string) {
	params = make(map[string]string)
	value, s := expectToken(header)
	if value == "" {
		return
	}
	value = strings.ToLower(value)
	s = "," + skipSpace(s)
	for strings.HasPrefix(s, ",") {
		var pkey string
		pkey, s = expectToken(skipSpace(s[1:]))
		if pkey == "" {
			return
		}
		if !strings.HasPrefix(s, "=") {
			return
		}
		var pvalue string
		pvalue, s = expectTokenOrQuoted(s[1:])
		if pvalue == "" {
			return
		}
		pkey = strings.ToLower(pkey)
		params[pkey] = pvalue
		s = skipSpace(s)
	}
	return
}
```

在执行完`tryEstablishChallenges`后（……），检测到当前不存在文件`<root>/v2/repositories/<name>/_layers/<algorithm>/<hex digest>/link`的情况下，执行`pbs.remoteStore.Stat(ctx, dgst)`操作，如下：

```go
func (pbs *proxyBlobStore) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	desc, err := pbs.localStore.Stat(ctx, dgst)
	if err == nil {
		return desc, err
	}

	if err != distribution.ErrBlobUnknown {
		return distribution.Descriptor{}, err
	}

	if err := pbs.authChallenger.tryEstablishChallenges(ctx); err != nil {
		return distribution.Descriptor{}, err
	}

	return pbs.remoteStore.Stat(ctx, dgst)
}
func (pr *proxyingRegistry) Repository(ctx context.Context, name reference.Named) (distribution.Repository, error) {
	c := pr.authChallenger

	tr := transport.NewTransport(http.DefaultTransport,
		auth.NewAuthorizer(c.challengeManager(), auth.NewTokenHandler(http.DefaultTransport, c.credentialStore(), name.Name(), "pull")))

	localRepo, err := pr.embedded.Repository(ctx, name)
	if err != nil {
		return nil, err
	}
	localManifests, err := localRepo.Manifests(ctx, storage.SkipLayerVerification())
	if err != nil {
		return nil, err
	}

	remoteRepo, err := client.NewRepository(ctx, name, pr.remoteURL.String(), tr)
	if err != nil {
		return nil, err
	}

	remoteManifests, err := remoteRepo.Manifests(ctx)
	if err != nil {
		return nil, err
	}

	return &proxiedRepository{
		blobStore: &proxyBlobStore{
			localStore:     localRepo.Blobs(ctx),
			remoteStore:    remoteRepo.Blobs(ctx),
			scheduler:      pr.scheduler,
			repositoryName: name,
			authChallenger: pr.authChallenger,
		},
		manifests: &proxyManifestStore{
			repositoryName:  name,
			localManifests:  localManifests, // Options?
			remoteManifests: remoteManifests,
			ctx:             ctx,
			scheduler:       pr.scheduler,
			authChallenger:  pr.authChallenger,
		},
		name: name,
		tags: &proxyTagService{
			localTags:      localRepo.Tags(ctx),
			remoteTags:     remoteRepo.Tags(ctx),
			authChallenger: pr.authChallenger,
		},
	}, nil
}
// NewRepository creates a new Repository for the given repository name and base URL.
func NewRepository(ctx context.Context, name reference.Named, baseURL string, transport http.RoundTripper) (distribution.Repository, error) {
	ub, err := v2.NewURLBuilderFromString(baseURL, false)
	if err != nil {
		return nil, err
	}

	client := &http.Client{
		Transport:     transport,
		CheckRedirect: checkHTTPRedirect,
		// TODO(dmcgowan): create cookie jar
	}

	return &repository{
		client:  client,
		ub:      ub,
		name:    name,
		context: ctx,
	}, nil
}
type repository struct {
	client  *http.Client
	ub      *v2.URLBuilder
	context context.Context
	name    reference.Named
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
func (bs *blobs) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	return bs.statter.Stat(ctx, dgst)

}
type blobStatter struct {
	name   reference.Named
	ub     *v2.URLBuilder
	client *http.Client
}
// NewCachedBlobStatter creates a new statter which prefers a cache and
// falls back to a backend.
func NewCachedBlobStatter(cache distribution.BlobDescriptorService, backend distribution.BlobDescriptorService) distribution.BlobDescriptorService {
	return &cachedBlobStatter{
		cache:   cache,
		backend: backend,
	}
}
type cachedBlobStatter struct {
	cache   distribution.BlobDescriptorService
	backend distribution.BlobDescriptorService
	tracker MetricsTracker
}
func (cbds *cachedBlobStatter) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	desc, err := cbds.cache.Stat(ctx, dgst)
	if err != nil {
		if err != distribution.ErrBlobUnknown {
			context.GetLogger(ctx).Errorf("error retrieving descriptor from cache: %v", err)
		}

		goto fallback
	}

	if cbds.tracker != nil {
		cbds.tracker.Hit()
	}
	return desc, nil
fallback:
	if cbds.tracker != nil {
		cbds.tracker.Miss()
	}
	desc, err = cbds.backend.Stat(ctx, dgst)
	if err != nil {
		return desc, err
	}

	if err := cbds.cache.SetDescriptor(ctx, dgst, desc); err != nil {
		context.GetLogger(ctx).Errorf("error adding descriptor %v to cache: %v", desc.Digest, err)
	}

	return desc, err

}
func (bs *blobStatter) Stat(ctx context.Context, dgst digest.Digest) (distribution.Descriptor, error) {
	ref, err := reference.WithDigest(bs.name, dgst)
	if err != nil {
		return distribution.Descriptor{}, err
	}
	u, err := bs.ub.BuildBlobURL(ref)
	if err != nil {
		return distribution.Descriptor{}, err
	}

	resp, err := bs.client.Head(u)
	if err != nil {
		return distribution.Descriptor{}, err
	}
	defer resp.Body.Close()

	if SuccessStatus(resp.StatusCode) {
		lengthHeader := resp.Header.Get("Content-Length")
		if lengthHeader == "" {
			return distribution.Descriptor{}, fmt.Errorf("missing content-length header for request: %s", u)
		}

		length, err := strconv.ParseInt(lengthHeader, 10, 64)
		if err != nil {
			return distribution.Descriptor{}, fmt.Errorf("error parsing content-length: %v", err)
		}

		return distribution.Descriptor{
			MediaType: resp.Header.Get("Content-Type"),
			Size:      length,
			Digest:    dgst,
		}, nil
	} else if resp.StatusCode == http.StatusNotFound {
		return distribution.Descriptor{}, distribution.ErrBlobUnknown
	}
	return distribution.Descriptor{}, HandleErrorResponse(resp)
}
// WithDigest combines the name from "name" and the digest from "digest" to form
// a reference incorporating both the name and the digest.
func WithDigest(name Named, digest digest.Digest) (Canonical, error) {
	if !anchoredDigestRegexp.MatchString(digest.String()) {
		return nil, ErrDigestInvalidFormat
	}
	if tagged, ok := name.(Tagged); ok {
		return reference{
			name:   name.Name(),
			tag:    tagged.Tag(),
			digest: digest,
		}, nil
	}
	return canonicalReference{
		name:   name.Name(),
		digest: digest,
	}, nil
}
type canonicalReference struct {
	name   string
	digest digest.Digest
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

{
    Name:        RouteNameBlob,
    Path:        "/v2/{name:" + reference.NameRegexp.String() + "}/blobs/{digest:" + digest.DigestRegexp.String() + "}",
    Entity:      "Blob",
    Description: "Operations on blobs identified by `name` and `digest`. Used to fetch or delete layers by digest.",
    Methods: []MethodDescriptor{
        {
            Method:      "GET",
            Description: "Retrieve the blob from the registry identified by `digest`. A `HEAD` request can also be issued to this endpoint to obtain resource information without receiving all data.",
            Requests: []RequestDescriptor{
                {
                    Name: "Fetch Blob",
                    Headers: []ParameterDescriptor{
                        hostHeader,
                        authHeader,
                    },
                    PathParameters: []ParameterDescriptor{
                        nameParameterDescriptor,
                        digestPathParameter,
                    },
                    Successes: []ResponseDescriptor{
                        {
                            Description: "The blob identified by `digest` is available. The blob content will be present in the body of the request.",
                            StatusCode:  http.StatusOK,
                            Headers: []ParameterDescriptor{
                                {
                                    Name:        "Content-Length",
                                    Type:        "integer",
                                    Description: "The length of the requested blob content.",
                                    Format:      "<length>",
                                },
                                digestHeader,
                            },
                            Body: BodyDescriptor{
                                ContentType: "application/octet-stream",
                                Format:      "<blob binary data>",
                            },
                        },
                        {
                            Description: "The blob identified by `digest` is available at the provided location.",
                            StatusCode:  http.StatusTemporaryRedirect,
                            Headers: []ParameterDescriptor{
                                {
                                    Name:        "Location",
                                    Type:        "url",
                                    Description: "The location where the layer should be accessible.",
                                    Format:      "<blob location>",
                                },
                                digestHeader,
                            },
                        },
                    },
                    Failures: []ResponseDescriptor{
                        {
                            Description: "There was a problem with the request that needs to be addressed by the client, such as an invalid `name` or `tag`.",
                            StatusCode:  http.StatusBadRequest,
                            ErrorCodes: []errcode.ErrorCode{
                                ErrorCodeNameInvalid,
                                ErrorCodeDigestInvalid,
                            },
                            Body: BodyDescriptor{
                                ContentType: "application/json; charset=utf-8",
                                Format:      errorsBody,
                            },
                        },
                        {
                            Description: "The blob, identified by `name` and `digest`, is unknown to the registry.",
                            StatusCode:  http.StatusNotFound,
                            Body: BodyDescriptor{
                                ContentType: "application/json; charset=utf-8",
                                Format:      errorsBody,
                            },
                            ErrorCodes: []errcode.ErrorCode{
                                ErrorCodeNameUnknown,
                                ErrorCodeBlobUnknown,
                            },
                        },
                        unauthorizedResponseDescriptor,
                        repositoryNotFoundResponseDescriptor,
                        deniedResponseDescriptor,
                        tooManyRequestsDescriptor,
                    },
                },
                {
                    Name:        "Fetch Blob Part",
                    Description: "This endpoint may also support RFC7233 compliant range requests. Support can be detected by issuing a HEAD request. If the header `Accept-Range: bytes` is returned, range requests can be used to fetch partial content.",
                    Headers: []ParameterDescriptor{
                        hostHeader,
                        authHeader,
                        {
                            Name:        "Range",
                            Type:        "string",
                            Description: "HTTP Range header specifying blob chunk.",
                            Format:      "bytes=<start>-<end>",
                        },
                    },
                    PathParameters: []ParameterDescriptor{
                        nameParameterDescriptor,
                        digestPathParameter,
                    },
                    Successes: []ResponseDescriptor{
                        {
                            Description: "The blob identified by `digest` is available. The specified chunk of blob content will be present in the body of the request.",
                            StatusCode:  http.StatusPartialContent,
                            Headers: []ParameterDescriptor{
                                {
                                    Name:        "Content-Length",
                                    Type:        "integer",
                                    Description: "The length of the requested blob chunk.",
                                    Format:      "<length>",
                                },
                                {
                                    Name:        "Content-Range",
                                    Type:        "byte range",
                                    Description: "Content range of blob chunk.",
                                    Format:      "bytes <start>-<end>/<size>",
                                },
                            },
                            Body: BodyDescriptor{
                                ContentType: "application/octet-stream",
                                Format:      "<blob binary data>",
                            },
                        },
                    },
                    Failures: []ResponseDescriptor{
                        {
                            Description: "There was a problem with the request that needs to be addressed by the client, such as an invalid `name` or `tag`.",
                            StatusCode:  http.StatusBadRequest,
                            ErrorCodes: []errcode.ErrorCode{
                                ErrorCodeNameInvalid,
                                ErrorCodeDigestInvalid,
                            },
                            Body: BodyDescriptor{
                                ContentType: "application/json; charset=utf-8",
                                Format:      errorsBody,
                            },
                        },
                        {
                            StatusCode: http.StatusNotFound,
                            ErrorCodes: []errcode.ErrorCode{
                                ErrorCodeNameUnknown,
                                ErrorCodeBlobUnknown,
                            },
                            Body: BodyDescriptor{
                                ContentType: "application/json; charset=utf-8",
                                Format:      errorsBody,
                            },
                        },
                        {
                            Description: "The range specification cannot be satisfied for the requested content. This can happen when the range is not formatted correctly or if the range is outside of the valid size of the content.",
                            StatusCode:  http.StatusRequestedRangeNotSatisfiable,
                        },
                        unauthorizedResponseDescriptor,
                        repositoryNotFoundResponseDescriptor,
                        deniedResponseDescriptor,
                        tooManyRequestsDescriptor,
                    },
                },
            },
        },
        {
            Method:      "DELETE",
            Description: "Delete the blob identified by `name` and `digest`",
            Requests: []RequestDescriptor{
                {
                    Headers: []ParameterDescriptor{
                        hostHeader,
                        authHeader,
                    },
                    PathParameters: []ParameterDescriptor{
                        nameParameterDescriptor,
                        digestPathParameter,
                    },
                    Successes: []ResponseDescriptor{
                        {
                            StatusCode: http.StatusAccepted,
                            Headers: []ParameterDescriptor{
                                {
                                    Name:        "Content-Length",
                                    Type:        "integer",
                                    Description: "0",
                                    Format:      "0",
                                },
                                digestHeader,
                            },
                        },
                    },
                    Failures: []ResponseDescriptor{
                        {
                            Name:       "Invalid Name or Digest",
                            StatusCode: http.StatusBadRequest,
                            ErrorCodes: []errcode.ErrorCode{
                                ErrorCodeDigestInvalid,
                                ErrorCodeNameInvalid,
                            },
                        },
                        {
                            Description: "The blob, identified by `name` and `digest`, is unknown to the registry.",
                            StatusCode:  http.StatusNotFound,
                            Body: BodyDescriptor{
                                ContentType: "application/json; charset=utf-8",
                                Format:      errorsBody,
                            },
                            ErrorCodes: []errcode.ErrorCode{
                                ErrorCodeNameUnknown,
                                ErrorCodeBlobUnknown,
                            },
                        },
                        {
                            Description: "Blob delete is not allowed because the registry is configured as a pull-through cache or `delete` has been disabled",
                            StatusCode:  http.StatusMethodNotAllowed,
                            Body: BodyDescriptor{
                                ContentType: "application/json; charset=utf-8",
                                Format:      errorsBody,
                            },
                            ErrorCodes: []errcode.ErrorCode{
                                errcode.ErrorCodeUnsupported,
                            },
                        },
                        unauthorizedResponseDescriptor,
                        repositoryNotFoundResponseDescriptor,
                        deniedResponseDescriptor,
                        tooManyRequestsDescriptor,
                    },
                },
            },
        },

        // TODO(stevvooe): We may want to add a PUT request here to
        // kickoff an upload of a blob, integrated with the blob upload
        // API.
    },
}
```

也即向upstream（后端：remoteurl: https://registry-1.docker.io）发出请求`HEAD /v2/library/centos/blobs/sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4`，并构建`distribution.Descriptor`结构（包含`<root>/v2/blobs/<algorithm>/<first two hex bytes of digest>/<hex digest>/data`文件大小，内容类型和digest）返回

在执行完`desc, err := blobs.Stat(bh, bh.Digest)`后，接下来执行真正的获取blobs数据文件操作，如下：

```go
// GetBlob fetches the binary data from backend storage returns it in the
// response.
func (bh *blobHandler) GetBlob(w http.ResponseWriter, r *http.Request) {
	context.GetLogger(bh).Debug("GetBlob")
	blobs := bh.Repository.Blobs(bh)
	desc, err := blobs.Stat(bh, bh.Digest)
	if err != nil {
		if err == distribution.ErrBlobUnknown {
			bh.Errors = append(bh.Errors, v2.ErrorCodeBlobUnknown.WithDetail(bh.Digest))
		} else {
			bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		}
		return
	}

	if err := blobs.ServeBlob(bh, w, r, desc.Digest); err != nil {
		context.GetLogger(bh).Debugf("unexpected error getting blob HTTP handler: %v", err)
		bh.Errors = append(bh.Errors, errcode.ErrorCodeUnknown.WithDetail(err))
		return
	}
}
func (pbs *proxyBlobStore) ServeBlob(ctx context.Context, w http.ResponseWriter, r *http.Request, dgst digest.Digest) error {
	served, err := pbs.serveLocal(ctx, w, r, dgst)
	if err != nil {
		context.GetLogger(ctx).Errorf("Error serving blob from local storage: %s", err.Error())
		return err
	}

	if served {
		return nil
	}

	if err := pbs.authChallenger.tryEstablishChallenges(ctx); err != nil {
		return err
	}

	mu.Lock()
	_, ok := inflight[dgst]
	if ok {
		mu.Unlock()
		_, err := pbs.copyContent(ctx, dgst, w)
		return err
	}
	inflight[dgst] = struct{}{}
	mu.Unlock()

	go func(dgst digest.Digest) {
		if err := pbs.storeLocal(ctx, dgst); err != nil {
			context.GetLogger(ctx).Errorf("Error committing to storage: %s", err.Error())
		}

		blobRef, err := reference.WithDigest(pbs.repositoryName, dgst)
		if err != nil {
			context.GetLogger(ctx).Errorf("Error creating reference: %s", err)
			return
		}

		pbs.scheduler.AddBlob(blobRef, repositoryTTL)
	}(dgst)

	_, err = pbs.copyContent(ctx, dgst, w)
	if err != nil {
		return err
	}
	return nil
}
func (pbs *proxyBlobStore) serveLocal(ctx context.Context, w http.ResponseWriter, r *http.Request, dgst digest.Digest) (bool, error) {
	localDesc, err := pbs.localStore.Stat(ctx, dgst)
	if err != nil {
		// Stat can report a zero sized file here if it's checked between creation
		// and population.  Return nil error, and continue
		return false, nil
	}

	if err == nil {
		proxyMetrics.BlobPush(uint64(localDesc.Size))
		return true, pbs.localStore.ServeBlob(ctx, w, r, dgst)
	}

	return false, nil

}
// inflight tracks currently downloading blobs
var inflight = make(map[digest.Digest]struct{})

func (pbs *proxyBlobStore) storeLocal(ctx context.Context, dgst digest.Digest) error {
	defer func() {
		mu.Lock()
		delete(inflight, dgst)
		mu.Unlock()
	}()

	var desc distribution.Descriptor
	var err error
	var bw distribution.BlobWriter

	bw, err = pbs.localStore.Create(ctx)
	if err != nil {
		return err
	}

	desc, err = pbs.copyContent(ctx, dgst, bw)
	if err != nil {
		return err
	}

	_, err = bw.Commit(ctx, desc)
	if err != nil {
		return err
	}

	return nil
}

func (pbs *proxyBlobStore) copyContent(ctx context.Context, dgst digest.Digest, writer io.Writer) (distribution.Descriptor, error) {
	desc, err := pbs.remoteStore.Stat(ctx, dgst)
	if err != nil {
		return distribution.Descriptor{}, err
	}

	if w, ok := writer.(http.ResponseWriter); ok {
		setResponseHeaders(w, desc.Size, desc.MediaType, dgst)
	}

	remoteReader, err := pbs.remoteStore.Open(ctx, dgst)
	if err != nil {
		return distribution.Descriptor{}, err
	}

	defer remoteReader.Close()

	_, err = io.CopyN(writer, remoteReader, desc.Size)
	if err != nil {
		return distribution.Descriptor{}, err
	}

	proxyMetrics.BlobPush(uint64(desc.Size))

	return desc, nil
}
```

