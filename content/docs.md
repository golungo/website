---
title: "Documentation"
layout: page
date: 2023-02-08T14:42:49+04:00
draft: false
description: A documentation for lungo - a fast and simple MongoDB driver for Go
---

### Here is an example how to use lungo.

We will try to build a couple of endpoints:
- Get Category with populated items inside
- Get Item by ID with populated category inside

Overall project structure will look like this:
{{< highlight bash >}}
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
{{< /highlight >}}

Let's create directory for our example and initialize the Go module:
{{< highlight bash  >}}
mkdir example
cd example
go mod init golungo/example
{{< /highlight  >}}



## Connect

Let's start with connection.
Create main.go file and put the following code.

{{< highlight go >}}
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
{{< /highlight  >}}

## Define models

Since I want to show you how to populate data across the models, let's create two models:
- Category Model
- Item Model

Each model will "inherit" the lungo/query Model, to provide Methods like Match, Lookup, Virtual and Exec to our model, and it must remain "invisible" to json and bson packages.
As well, each model must provide a "GetModel" method, where it will be Initialized and configured before each request to mongodb.

### Category Model 

Our category will contain only two  actual properties and one virtual property:
- ID - ObjectID of a document
- Title - Category title
- Items - A "Virtual" property, where we will be populating our items.

{{< highlight go  >}}
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
{{< /highlight  >}}

### Item Model

Items will contain four actual properties and one virtual as well:
- ID - ObjectID of a mongo document
- Title - Item title
- Content - item content
- CategoryID - ID of item category, which we will be using to join with categories collection
- Category - A "Virtual" property, where we will be populating the item category data

{{< highlight go >}}
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
{{< /highlight  >}}

Let's talk about "Virtual" properties. 
The main purpose of a virtual property is to provide a way to work with the result of mongodb $lookup operation.

The `.Virtual()` method takes as an arguments:
- The target collection name, to search documents from
- The local field name, to join with, defined in 'bson' struct tag
- The foreign field name, to join with, defined in 'bson' struct tag
- The name of a virtual field, to access the result, also defined in 'bson' struct tag

After that, using `lungo.Fields` type, we can provide the information which fields we want to lookup and populate.

## Controllers

We will create two controllers, for data fetching of Categories and Items.
Let's write the `GetCategoryById()` controller first.

### Get Category By Id 

This method will take as an argument the seeking Category ID with a type of `lungo.ObjectID`, which is basically the proxied version of `primitive.ObjectID`.

{{< highlight go  >}}
package controllers

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
{{< /highlight  >}}

### Get Item By Id 

Same way as the previous one, this controller will take an argument of itemId with type `lungo.ObjectID`.

As you will see next, it is basically the same method, but for Item model.

{{< highlight go  >}}
package controllers

import (
	"encoding/json"

	"golungo/example/models"

	"github.com/golungo/lungo"
)

func GetItemById(itemId lungo.ObjectID) (models.Item, error) {
	var Items []models.Item

	filter := lungo.Filter{
		"_id": itemId
	}

	populate := lungo.Fields{
		"category"
	}

	result, err := models.GetItemsCollection().Match(filter).Lookup(populate).Exec()
	if err != nil {
		return Items, err
	}

	if err := json.Unmarshal(result, &Items); err != nil {
		return Items, err
	}

	if len(Categories) == 0 {
		return Items, err
	}

	return Items[0], nil
}
{{< /highlight  >}}


## Route Handlers

For this example i will use the Gin Web Framework to make it simple.

Let's write the Get Category by id route handler, responding to `/api/v1/categories/:categoryId` and the Get Item By Id route handler, responding to `/api/v1/items/:itemId`.

They will be basically the same methods :)

### Get Category by Id

{{< highlight go  >}}
package handlers

import (
	"golungo/example/controllers"
	
	"github.com/gin-gonic/gin"
	"github.com/golungo/lungo"
)

func GetCategoryById(c *gin.Context) {
	categoryId, err := lungo.ObjectIDFromHex(c.Param("categoryId"))
	if err != nil {
		c.JSON(404, "Not Found")
		c.Abort()
		return
	}

	Category, err := controllers.GetCategoryById(categoryId)
	if err != nil {
		c.JSON(404, "Not Found")
		c.Abort()
		return
	}

	c.JSON(200, Category)
	c.Abort()
} 

{{< /highlight >}}

### Get Item by Id

{{< highlight go  >}}
package handlers

import (
	"golungo/example/controllers"

	"github.com/gin-gonic/gin"
	"github.com/golungo/lungo"
)

func GetItemById(c *gin.Context) {
	itemId, err := lungo.ObjectIDFromHex(c.Param("itemId"))
	if err != nil {
		c.JSON(404, "Not Found")
		c.Abort()
		return
	}

	Item, err := controllers.GetItemById(itemId)
	if err != nil {
		c.JSON(404, "Not Found")
		c.Abort()
		return
	}

	c.JSON(200, Item)
	c.Abort()
} 

{{< /highlight >}}

## Wrapping it all up

Now, when we have our models, controllers and handlers - we can wrap it all up in `main.go` file and run it!

Our final main.go file will look like this:
{{< highlight go  >}}
package main

import (
	"os"

	"golungo/example/handlers"

	"github.com/golungo/lungo"
	"github.com/gin-gonic/gin"
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

	router := gin.Default()
	router.GET("/api/v1/categories/:categoryId", handlers.GetCategoryById)
	router.GET("/api/v1/items/:itemId", handlers.GetItemById)
	router.Run(":8080")
}
{{< /highlight >}}

Now, when you will make a GET request to `http://localhost:8080/api/v1/categories/:categoryId`, you will receive the category with all the items, inside it.
And same for `http://localhost:8080/api/v1/items/:itemId` - there you will see populated category.


## Example Repository and playground

I made an example repository with this small project, populated with data and configured MongoDB so you can fork it and play around.

<a href="https://github.com/golungo/example" target="_blank" rel="nofollow">
	https://github.com/golungo/example
</a>