<h2>POC ASP.NET Core Logging with ElasticSearch, Kibana, and Docker</h2> 

<h3>Tech:</h3>

 - ASP.NET Core 3.1
 - Docker
 - ElasticSearch
 - Kibana
 - Serilog

<h3>What is ElasticSearch?</h3> 

In simple terms, ElasticSearch is an open source database that is well suited to indexing logs and analytical data.

<h3>What is Kibana?</h3> 

Kibana is an open source data visualization user interface for ElasticSearch. Think of ElasticSearch as the database and Kibana as the web user interface which you can use to build graphs and query data in ElasticSearch.

<h3>What is Serilog?</h3> 

Serilog is a plugin for ASP.NET Core that makes logging easy. There are various sinks available for Serilog - for instance you get plain text, SQL and ElasticSearch sinks to name a few.

<h3>Create a new file named docker-compose.yml:</h3> 

```json
version: '3.1'

services:

  elasticsearch:
   container_name: elasticsearch
   image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
   ports:
    - 9200:9200
   volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data
   environment:
    - xpack.monitoring.enabled=true
    - xpack.watcher.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    - discovery.type=single-node
   networks:
    - elastic

  kibana:
   container_name: kibana
   image: docker.elastic.co/kibana/kibana:7.6.2
   ports:
    - 5601:5601
   depends_on:
    - elasticsearch
   environment:
    - ELASTICSEARCH_URL=http://localhost:9200
   networks:
    - elastic
  
networks:
  elastic:
    driver: bridge

volumes:
  elasticsearch-data:
```

Run the docker compose command

> docker-compose up -d

Verify Elasticsearch is up and running

> http://localhost:9200

Verify that Kibana is up and running

> http://localhost:5601

<h3>Commands VsCode:</h3> 

> dotnet new mvc --no-https -o Elastic.Kibana.Serilog

> dotnet add package Serilog.AspNetCore

> dotnet add package Serilog.Enrichers.Environment

> dotnet add package Serilog.Sinks.Debug

> dotnet add package Serilog.Sinks.Elasticsearch

> dotnet restore

Adding Serilog log level verbosity in appsettings.json

Remove the Logging section in appsettings.json and replace it with the following configuration below so that we can tell Serilog what the minimum log level verbosity should be, and what url to use for logging to Elasticsearch.

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Information",
        "System": "Warning"
      }
    }
  },
  "ElasticConfiguration": {
    "Uri": "http://localhost:9200"
  },
  "AllowedHosts": "*"
}
```

Configuring Logging in Program.cs

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Serilog;
using Serilog.Sinks.Elasticsearch;
using System;
using System.Reflection;
```

```csharp
public static void Main(string[] args)
{
	//configure logging first
	ConfigureLogging();

	//then create the host, so that if the host fails we can log errors
	CreateHost(args);
}
```

```csharp
private static void ConfigureLogging()
{
	var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
	var configuration = new ConfigurationBuilder()
		.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
		.AddJsonFile(
			$"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")}.json",
			optional: true)
		.Build();

	Log.Logger = new LoggerConfiguration()
		.Enrich.FromLogContext()
		.Enrich.WithMachineName()
		.WriteTo.Debug()
		.WriteTo.Console()
		.WriteTo.Elasticsearch(ConfigureElasticSink(configuration, environment))
		.Enrich.WithProperty("Environment", environment)
		.ReadFrom.Configuration(configuration)
		.CreateLogger();
}

private static ElasticsearchSinkOptions ConfigureElasticSink(IConfigurationRoot configuration, string environment)
{
	return new ElasticsearchSinkOptions(new Uri(configuration["ElasticConfiguration:Uri"]))
	{
		AutoRegisterTemplate = true,
		IndexFormat = $"{Assembly.GetExecutingAssembly().GetName().Name.ToLower().Replace(".", "-")}-{environment?.ToLower().Replace(".", "-")}-{DateTime.UtcNow:yyyy-MM}"
	};
}
```

```csharp
private static void CreateHost(string[] args)
{
	try
	{
		CreateHostBuilder(args).Build().Run();
	}
	catch (System.Exception ex)
	{
		Log.Fatal($"Failed to start {Assembly.GetExecutingAssembly().GetName().Name}", ex);
		throw;
	}
}

public static IHostBuilder CreateHostBuilder(string[] args) =>
	Host.CreateDefaultBuilder(args)
		.ConfigureWebHostDefaults(webBuilder =>
		{
			webBuilder.UseStartup<Startup>();
		})
		.ConfigureAppConfiguration(configuration =>
		{
			configuration.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
			configuration.AddJsonFile(
				$"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")}.json",
				optional: true);
		})
		.UseSerilog();
```

Run the MVC Application

> http://localhost:5000/

Let's open up Kibana page, and then click the Discover link in the navigation:

> http://localhost:5601

Create an Index Pattern in Kibana to Show Data
    - Define index pattern: elastic-kibana-serilog-*
    - Configure settings: @timestamp


<h2>Reference</h2>

[Logging with ElasticSearch, Kibana, ASP.NET Core and Docker](https://www.humankode.com/asp-net-core/logging-with-elasticsearch-kibana-asp-net-core-and-docker)