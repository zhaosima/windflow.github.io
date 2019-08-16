---
title: why aspnetcore can not effect host urls in appsettings.json
date: 2019-04-15 18:16:13
tags:
---

## what's the matter?
If we want change the default 5000,5001 host url,the simple way is

```Java
    webHostBuilder
    .UseUrls("http://localhost:6666")
```

What if we want read from config?Should we just add this to appsettings.json?

```Java

    "urls":"http://localhost:6666",
```

But it doesn't work.

So let's check out the source code and find the right way.

## how things work?
First,let's find out what UseUrls do.
```Java
        public static IWebHostBuilder UseUrls(this IWebHostBuilder hostBuilder, params string[] urls)
        {
            if (urls == null)
            {
                throw new ArgumentNullException(nameof(urls));
            }

            return hostBuilder.UseSetting(WebHostDefaults.ServerUrlsKey, string.Join(ServerUrlsSeparator, urls));
        }

        public IWebHostBuilder UseSetting(string key, string value)
        {
            _config[key] = value;
            return this;
        }
```
So,it's just set the key-value of config.Next we'll find out how the config init.

```Java

        public WebHostBuilder()
        {
            _hostingEnvironment = new HostingEnvironment();

            _config = new ConfigurationBuilder()
                .AddEnvironmentVariables(prefix: "ASPNETCORE_")
                .Build();

            // blablabla
        }
```
config builder read enviorment variables to build the config. 
```Java

        public static IConfigurationBuilder AddEnvironmentVariables(this IConfigurationBuilder configurationBuilder)
        {
            configurationBuilder.Add(new EnvironmentVariablesConfigurationSource());
            return configurationBuilder;
        }

        // EnvironmentVariablesConfigurationSource
        public IConfigurationProvider Build(IConfigurationBuilder builder)
        {
            return new EnvironmentVariablesConfigurationProvider(Prefix);
        }

        // EnvironmentVariablesConfigurationProvider
        internal void Load(IDictionary envVariables)
        {
            var data = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase); // ignorecase accecpt X_YYYY ,X_yyyy or sth else.  

            var filteredEnvVariables = envVariables
                .Cast<DictionaryEntry>()
                .SelectMany(AzureEnvToAppEnv)
                .Where(entry => ((string)entry.Key).StartsWith(_prefix, StringComparison.OrdinalIgnoreCase));

            foreach (var envVariable in filteredEnvVariables)
            {
                var key = ((string)envVariable.Key).Substring(_prefix.Length);
                data[key] = (string)envVariable.Value;
            }

            Data = data;
        }
```
It load every environment variables which has "ASPNETCORE_" prefix and remove the prefix as key.So we can add a environment variable such as 
```bash
    ASPNETCORE_URLS='http://localhost:6666'
```
This is the first way to config the host. But why can't set urls in the appsettings?
Let's move on.

The normal way of adding appsetings to config is like this.
```Java
        builder.ConfigureAppConfiguration((hostingContext, config) =>
        {
            var env = hostingContext.HostingEnvironment;

            config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                  .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true);

            // blablabla
        }

        public IWebHostBuilder ConfigureAppConfiguration(Action<WebHostBuilderContext, IConfigurationBuilder> configureDelegate)
        {
            _configureAppConfigurationBuilder += configureDelegate;
            return this;
        }
```
The _configureAppConfigurationBuilder works on Builer's build action.

```Java
    public IWebHost Build()
    {
        var hostingServices = BuildCommonServices(out var hostingStartupErrors);
        var applicationServices = hostingServices.Clone();
        var hostingServiceProvider = GetProviderFromFactory(hostingServices);
        
        var host = new WebHost(
            applicationServices,
            hostingServiceProvider,
            _options,
            _config,
            hostingStartupErrors);

        host.Initialize();
    }

    private IServiceCollection BuildCommonServices(out AggregateException hostingStartupErrors)
    {
        // something build before

        var builder = new ConfigurationBuilder()
                .SetBasePath(_hostingEnvironment.ContentRootPath)
                .AddConfiguration(_config, shouldDisposeConfiguration: true);

        _configureAppConfigurationBuilder?.Invoke(_context, builder);

        var configuration = builder.Build();
        // something build after
    }
```
When it build the common services,It DO NOT use the original config,but combine the _config with appsettings.json to a new config.And We take a look at the host constructor, the host use the ORIGINAL config.THAT'S WHY the host can't read the urls config in appsettings.json. It's totally in the different place!

But don't be sad.WebHost has an extension allow you to add outter config to the _config.
```Java
    public static IWebHostBuilder UseConfiguration(this IWebHostBuilder hostBuilder, Configuration configuration)
    {
        foreach (var setting in configuration.AsEnumerable(makePathsRelative: true))
        {
            hostBuilder.UseSetting(setting.Key, setting.Value);
        }

        return hostBuilder;
    }
```
So you can make a new ConfigBuilder,load from a config file,build a config,add to hostbuilder finally.It's the second way.
```Java
    IWebHost wbh;
    var config = new ConfigurationBuilder().
        SetBasePath(Directory.GetCurrentDirectory())
        .AddJsonFile("webhost.json", optional: true)
        .Build();

    wbh.UseConfiguration(config);
```

## how the urls config take effect? 
```Java
    // class WebHost
    private void EnsureServer()
    {
        if (Server == null)
        {
            Server = _applicationServices.GetRequiredService<IServer>();

            var serverAddressesFeature = Server.Features?.Get<IServerAddressesFeature>();
            var addresses = serverAddressesFeature?.Addresses;
            if (addresses != null && !addresses.IsReadOnly && addresses.Count == 0)
            {
                var urls = _config[WebHostDefaults.ServerUrlsKey] ?? _config[DeprecatedServerUrlsKey];
                if (!string.IsNullOrEmpty(urls))
                {
                    serverAddressesFeature.PreferHostingUrls = WebHostUtilities.ParseBool(_config, WebHostDefaults.PreferHostingUrlsKey);

                    foreach (var value in urls.Split(new[] { ';' }, StringSplitOptions.RemoveEmptyEntries))
                    {
                        addresses.Add(value);
                    }
                }
            }
        }
    }

    // static class WebHostDefaults
    public static readonly string ServerUrlsKey = "urls";
```
The EnsureServer() method will init the server address from the config["urls"].

All things done!




