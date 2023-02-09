---
title: "Documentation"
layout: page
date: 2023-02-08T14:42:49+04:00
draft: false
description: A documentation for lungo - a fast and simple MongoDB driver for Go
---

Here is a small example how to use lungo.
We will try to build a couple of endpoints:
- Get Category with populated items inside
- Get Item by ID with populated category inside

Overall project structure will look like this:

{{% highlight bash %}}
- example
- - main.go
- - models
- - - Category.go
- - - Item.go
- - handlers
- - - GetCategoryById.go
- - - GetItemById.go
- - go.mod
- - go.sum
{{% /highlight %}}

Let's create directory for our example and initialize the Go module:
{{% highlight bash  %}}
mkdir example
cd example
go mod init golungo/example
{{% /highlight  %}}



## Connect

Let's start with connection.
Create main.go file and put the following code.

{{% highlight go %}}
package main

import (
	"os"
	"github.com/golungo/lungo"
)

func main() {
	uri := os.GetEnv("MONGO_URI")
	if err := lungo.Init(uri); err != nil {
		panic(err)
	}
	if err := lungo.Connect(); err != nil {
		panic(err)
	}
	defer func() {
		if err := lungo.Disconnect(); err != nil {
			panic(err)
		}
	}()
}
{{% /highlight  %}}

## Define models

Since I want to show you how to populate data across the models, let's create two models:
- Category Model
- Item Model

Each model will "inherit" the lungo/query Model, to provide Methods like Match, Lookup, Virtual and Exec to our model, and it must remain "invisible" to json and bson packages.
As well, each model must provide a "GetModel" method, where it will be Initialized and configured before each request to mongodb.

#### Category Model 

Our category will contain only two  actual properties and one virtual property:
- ID - ObjectID of a document
- Title - Category title
- Items - A "Virtual" property, where we will be populating our items.

{{% highlight go  %}}
package models

import (
	"reflect"
	
	"github.com/golungo/lungo"
	"github.com/golungo/lungo/query"
)

type Category struct {
	query.Model    `json:"-" bson:"-"`
	ID             lungo.ObjectID    `json:"_id" bson:"_id"`
	Items          []Item            `json:"items,omitempty" bson:"-"`
}

func GetCategoryCollection() query.Model {
	var model Category
	
	return model.Init(
		"categories", reflect.TypeOf(model),
	).Virtual(
		"items", "_id", "categoryId", "items", false,
	)
}
{{% /highlight  %}}

#### Item Model

Items will contain four actual properties and one virtual as well:
- ID - ObjectID of a mongo document
- Title - Item title
- Content - item content
- CategoryID - ID of item category, which we will be using to join with categories collection
- Category - A "Virtual" property, where we will be populating the item category data

{{% highlight go %}}
package models

import (
	"reflect"
	
	"github.com/golungo/lungo"
	"github.com/golungo/lungo/query"
)

type Item struct {
	query.Model `json:"-" bson:"-"`
	ID          lungo.ObjectID   `json:"_id" bson:"_id"`
	Title       string           `json:"title" bson:"title"`
	Content     string           `json:"content" bson:"content"`
	CategoryID  lungo.ObjectID   `json:"categoryId" bson:"categoryId"`
	Category    []Category       `json:"category" bson:"-"`
}

func GetItemCollection() query.Model {
	var model Item
	
	return model.Init(
		"items", reflect.TypeOf(model),
	).Virtual(
		"categories", "categoryId", "_id", "category", false,
	)
}
{{% /highlight  %}}

Let's talk about "Virtual" properties. 
The main purpose of a virtual property is to provide a way to work with the result of mongodb $lookup operation.

The `.Virtual()` method takes as an arguments:
- The target collection name, to search documents from
- The local field name, to join with, defined in 'bson' struct tag
- The foreign field name, to join with, defined in 'bson' struct tag
- The name of a virtual field, to access the result, also defined in 'bson' struct tag

After that, using `lungo.Fields` type, we can provide the information which fields we want to lookup and populate.

Let's write our `GetCategoryById()` handler method 

## Get Category Handler

{{% highlight go  %}}
package handlers

import (
	"encoding/json"

	"golungo/example/models"

	"github.com/golungo/lungo"
)

func GetCategoryById(categoryId lungo.ObjectID) (models.Category, error) {
	var Categories []models.Category

	filter := lungo.Filter{
		"_id": categoryId
	}

	populate := lungo.Fields{
		"items"
	}

	result, err := models.GetCategoryCollection().Match(filter).Lookup(populate).Exec()
	if err != nil {
		return Categories, err
	}

	if err := json.Unmarshal(result, &Categories); err != nil {
		return Categories, err
	}

	if len(Categories) == 0 {
		return Categories, err
	}

	return Categories[0], nil
}
{{% /highlight  %}}