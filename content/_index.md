---
title: "A fast and simple MongoDB driver for Go"
layout: index
date: 2023-02-08T14:42:49+04:00
draft: false
---

{{< tabs/wrapper welcome >}}
{{< tabs/header/wrapper >}}
  {{< tabs/header/item `connect` `Connect` `main.go` >}}
  {{< tabs/header/item `model` `Define` `models/model.go` >}}
  {{< tabs/header/item `handler` `Use` `handlers/handler.go` >}}
{{< /tabs/header/wrapper >}}
{{< tabs/item connect >}}
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
{{< /tabs/item >}}
{{< tabs/item model >}}
{{< highlight go  >}}
package models
    
import (
  "reflect"
      
  "github.com/golungo/lungo"
  "github.com/golungo/lungo/query"
)
    
type Child struct {
  ID          lungo.ObjectID `json:"_id" bson:"_id"`
  Title       string         `json:"title" bson:"title"`
}
    
type Parent struct {
  query.Model `json:"-" bson:"-"`
  ID          lungo.ObjectID `json:"_id" bson:"_id"`
  ChildID     lungo.ObjectID `json:"childId" bson:"childId"`
  Child       []Child        `json:"child" bson:"-"`
}
    
func GetCollection() query.Model {
  var model Parent
     
  return model.Init(
    "parents", reflect.TypeOf(model),
  ).Virtual(
    "childs", "childId", "_id", "child", false,
  )
} 
{{< /highlight  >}}
{{< /tabs/item >}}
{{< tabs/item handler >}}
{{< highlight go  >}}
package handlers
    
import (
  "encoding/json"
    
  "golungo/example/models"
    
  "github.com/golungo/lungo"
)
          
func Search(objectID lungo.ObjectID) ([]models.Parent, error) {
  var Parents []models.Parent
    
  filter := lungo.filter{
    "_id": objectID
  }
    
  populate := lungo.Populate{
    "childs"
  }
            
  result, err := models.GetCollection().Match(filter).Lookup(populate).Limit(limit).Exec()
  if err != nil {
    return Parents, err
  }
            
  if err := json.Unmarshal(result, &Parents); err != nil {
    return Parents, err
  }
            
  return Parents, nil
}
{{< /highlight  >}}
{{< /tabs/item >}}
{{< /tabs/wrapper >}}