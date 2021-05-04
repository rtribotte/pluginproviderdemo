# Provider Plugin Demo

[Traefik](https://traefik.io) plugins are developed using the [Go language](https://golang.org).

Rather than being pre-compiled and linked, however, plugins are executed on the fly by [Yaegi](https://github.com/traefik/yaegi), an embedded Go interpreter.

## Usage

For a plugin to be active for a given Traefik instance, it must be declared in the static configuration.

Plugins are parsed and loaded exclusively during startup, which allows Traefik to check the integrity of the code and catch errors early on.
If an error occurs during loading, the plugin is disabled.

For security reasons, it is not possible to start a new plugin or modify an existing one while Traefik is running.

Plugin dependencies must be [vendored](https://golang.org/ref/mod#tmp_25) for each plugin.
Vendored packages should be included in the plugin's GitHub repository. ([Go modules](https://blog.golang.org/using-go-modules) are not supported.)

### Configuration

For each plugin, the Traefik static configuration must define the module name (as is usual for Go packages).

The following declaration (given here in YAML) defines a plugin:

```yaml
# Static configuration
pilot:
  token: xxxxx

experimental:
  plugins:
    example:
      moduleName: github.com/traefik/pluginproviderdemo
      version: v0.1.0

providers:
  plugin:
    example:
      pollInterval: 2s
```

or:

```yaml
# Static configuration
# Dev mode
entryPoints:
  web:
    address: :80

log:
  level: DEBUG

pilot:
  token: xxxxx

experimental:
  devPlugin:
    goPath: /plugins/go
    moduleName: github.com/traefik/pluginproviderdemo

providers:
  plugin:
    # in dev mode (`devPlugin`) the name of the plugin is always `dev`.
    dev:
      pollInterval: 2s
```

## Defining a Plugin

A plugin package must define the following exported Go objects:

- A type `type Config struct { ... }`. The struct fields are arbitrary.
- A function `func CreateConfig() *Config`.
- A function `New(ctx context.Context, config *Config, name string) (*Provider, error)`.

The provider must follow this interface:

```go
type PluginProvider interface {
	Init() error
	Provide(cfgChan chan<- json.Marshaler) error
	Stop() error
}
```

The Go objects used to build the dynamic configuration are in the following repository: https://github.com/traefik/genconf

Example:

```go
// Package example a example plugin.
package example

import (
	"context"
	"encoding/json"

	"github.com/traefik/genconf/dynamic"
	"github.com/traefik/genconf/dynamic/tls"
)

// Config the plugin configuration.
type Config struct {
	// ...
}

// CreateConfig creates the default plugin configuration.
func CreateConfig() *Config {
	return &Config{
		// ...
	}
}

// Provider a plugin.
type Provider struct {
	name     string
    // ...
}

// New created a new plugin.
func New(ctx context.Context, config *Config, name string) (*Provider, error) {
	// ...
	return &Provider{
		// ...
	}, nil
}

// Init the provider.
func (p *Provider) Init() error {
	// ...
	return nil
}

// Provide creates and send dynamic configuration.
func (p *Provider) Provide(cfgChan chan<- json.Marshaler) error {
	// ...
	cfgChan <- cfg
	// ...
	return nil
}

// Stop to stop the provider and the related go routines.
func (p *Provider) Stop() error {
	// ...
	return nil
}
```

## Traefik Pilot

Traefik plugins are stored and hosted as public GitHub repositories.

Every 30 minutes, the Traefik Pilot online service polls Github to find plugins and add them to its catalog.

### Prerequisites

To be recognized by Traefik Pilot, your repository must meet the following criteria:

- The `traefik-plugin` topic must be set.
- The `.traefik.yml` manifest must exist, and be filled with valid contents.

If your repository fails to meet either of these prerequisites, Traefik Pilot will not see it.

### Manifest

A manifest is also mandatory, and it should be named `.traefik.yml` and stored at the root of your project.

This YAML file provides Traefik Pilot with information about your plugin, such as a description, a full name, and so on.

Here is an example of a typical `.traefik.yml`file:

```yaml
# The name of your plugin as displayed in the Traefik Pilot web UI.
displayName: Name of your plugin

type: provider

# The import path of your plugin.
import: github.com/username/my-plugin

# The base package name of your plugin
basePkg: myplugin

# A brief description of what your plugin is doing.
summary: Description of what my plugin is doing

# Medias associated to the plugin (optional)
iconPath: foo/icon.png
bannerPath: foo/banner.png

# Configuration data for your plugin.
# This is mandatory,
# and Traefik Pilot will try to execute the plugin with the data you provide as part of its startup validity tests.
testData:
  Headers:
    Foo: Bar
```

Properties include:

- `displayName` (required): The name of your plugin as displayed in the Traefik Pilot web UI.
- `type` (required): the type of the plugin (i.e. `provider`).
- `import` (required): The import path of your plugin.
- `summary` (required): A brief description of what your plugin is doing.
- `testData` (required): Configuration data for your plugin. This is mandatory, and Traefik Pilot will try to execute the plugin with the data you provide as part of its startup validity tests.
- `basePkg` (optional): The base package name of your plugin.
- `iconPath` (optional): A local path in the repository to the icon of the project.
- `bannerPath` (optional): A local path in the repository to the image that will be used when you will share your plugin page in social medias.

There should also be a `go.mod` file at the root of your project. Traefik Pilot will use this file to validate the name of the project.

### Plugin Package Name

Traefik needs a package name to lookup and interpret the code of your plugin. Any misconfiguration leads to a startup failure.

To do so, Traefik uses either the defined package in the manifest with the `basePkg` property or tries to use the import path to guess the package of your plugin.
Guessing remains to use the Github project name as the base package name.

#### Hyphen Characters In Github Project Name

If the Github project name contains hyphens (`-`), Traefik will replace them for underscores (`_`) characters:

`my-plugin` --> `my_plugin`

In that case, please use the appropriate form for your package name in the code or use the `basePkg` property in the manifest to define a custom package.

### Tags and Dependencies

Traefik Pilot gets your sources from a Go module proxy, so your plugins need to be versioned with a git tag.

Last but not least, if your plugin has Go package dependencies, you need to vendor them and add them to your GitHub repository.

If something goes wrong with the integration of your plugin, Traefik Pilot will create an issue inside your Github repository and will stop trying to add your repo until you close the issue.

## Troubleshooting

If Traefik Pilot fails to recognize your plugin, you will need to make one or more changes to your GitHub repository.

In order for your plugin to be successfully imported by Traefik Pilot, consult this checklist:

- The `traefik-plugin` topic must be set on your repository.
- There must be a `.traefik.yml` file at the root of your project describing your plugin, and it must have a valid `testData` property for testing purposes.
- There must be a valid `go.mod` file at the root of your project.
- Your plugin must be versioned with a git tag.
- If you have package dependencies, they must be vendored and added to your GitHub repository.

## Sample Code

This repository includes an example plugin, `demo`, for you to use as a reference for developing your own plugins.

[![Build Status](https://github.com/traefik/pluginproviderdemo/workflows/Main/badge.svg?branch=master)](https://github.com/traefik/pluginproviderdemo/actions)
