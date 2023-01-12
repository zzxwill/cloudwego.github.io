---
title: "Extend the Templates of Service Generated Code"
date: 2022-11-03
weight: 5
description: >
---

## Introduction

From version v0.4.3, Kitex tool provides a new flag named `-template-extension`, to support extending the templates of Service generated code.

- **Usage**: `kitex -template-extension extensions.json YOUR_IDL`.

The **extensions.json** is a JSON file, which contains the serialization result of a [TemplateExtension](https://pkg.go.dev/github.com/cloudwego/kitex/tool/internal_pkg/generator#TemplateExtension) object. The fields of this object will be injected into the backend of the Kitex code generation tool, to insert certain codes at specific points.

Kitex will generate a package for each service definition in the IDL under the folder **kitex_gen**, in which the APIs `NewClient`, `NewServer`, etc. are provided.

The content provided by extensions.json will apply to all packages of all service definitions, the fields `extend_client`, `extend_server`, `extend_invoker` are extensions for file client.go, server.go, invoker.go.

### Application Scenario

It is applicable to the scenario for customizing the unified suite.

Enterprise users have a unified customization of the framework, thus there is a set of fixed option configurations normally. We suggest that these options be encapsulated in suite, so that only one option needs to be configured during initialization. However, business developers still need to configure this suite option. Actually, business developers do not need to pay attention to this configuration, because business developers do not need to pay attention to infrastructure capabilities.

In ByteDance, a bytedSuite will be injected into the generated code. In order to facilitate the use of the external framework customization, the configuration customization capability is provided. Of course, if you want to further shield this detail from business developers, you can also encapsulate the Kitex tool and make this unified configuration built-in.

## Example

Assume we have an extensions.json file:

```json
{
    "dependencies": {
        "example.com/my/pkg": "pkg"
    },
    "extend_client": {
        "import_paths": ["example.com/my/pkg"],
        "extend_option": "options = append(options, client.WithSuite(pkg.MyClientSuite()))",
        "extend_file": "func Hello() {\nprintln(\"hello world\")\n}"
    },
    "extend_server": {
        "import_paths": ["example.com/my/pkg"],
        "extend_option": "options = append(options, server.WithSuite(pkg.MyServerSuite()))",
        "extend_file": "func Hello() {\nprintln(\"hello world\")\n}"
    }
}
```

- **dependencies**: Defines a set of packages that may be used in the template and the names they should be aliased as.

- **extend_client**: Customization for client.go

    - **import_paths**: Declare the package list that the code injected into client.go needs to import. Those packages must be declared in the **dependencies**.

    - **extend_option**: The code snippet will be injected into the `NewClient` function, where the default options are constructed. Thus, you can inject your own suite option.

    - **extend_file**: The code snippet will be directly appended to client.go. You can add extra functions or constants here.

- **extend_server** field works like the **extend_client** field.

An IDL that applied the extensions.json above may produce a cient.go like this (the injected part is highlighted):

```go {linenos=table,hl_lines=[8,22,"53-55"]}
// Code generated by Kitex v0.4.2. DO NOT EDIT.

package demoservice

import (
	"context"
	demo "example.com/demo/kitex_gen/demo"
	pkg "example.com/my/pkg"
	client "github.com/cloudwego/kitex/client"
	callopt "github.com/cloudwego/kitex/client/callopt"
)

// Client is designed to provide IDL-compatible methods with call-option parameter for kitex framework.
type Client interface {
	Test(ctx context.Context, request *demo.DemoTestRequest, callOptions ...callopt.Option) (r *demo.DemoTestResponse, err error)
}

// NewClient creates a client for the service defined in IDL.
func NewClient(destService string, opts ...client.Option) (Client, error) {
	var options []client.Option
	options = append(options, client.WithDestService(destService))
	options = append(options, client.WithSuite(pkg.MyClientSuite()))

	options = append(options, opts...)

	kc, err := client.NewClient(serviceInfo(), options...)
	if err != nil {
		return nil, err
	}
	return &kDemoServiceClient{
		kClient: newServiceClient(kc),
	}, nil
}

// MustNewClient creates a client for the service defined in IDL. It panics if any error occurs.
func MustNewClient(destService string, opts ...client.Option) Client {
	kc, err := NewClient(destService, opts...)
	if err != nil {
		panic(err)
	}
	return kc
}

type kDemoServiceClient struct {
	*kClient
}

func (p *kDemoServiceClient) Test(ctx context.Context, request *demo.DemoTestRequest, callOptions ...callopt.Option) (r *demo.DemoTestResponse, err error) {
	ctx = client.NewCtxWithCallOptions(ctx, callOptions)
	return p.kClient.Test(ctx, request)
}

func Hello() {
	println("hello world")
}
```