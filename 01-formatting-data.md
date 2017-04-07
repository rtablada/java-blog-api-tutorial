# Formatting Data

To get started we need to work on formatting POJOs into a format that matches the JSON API specification.

Let's start by creating a file in `src/main/java/com/ryantablada/entities/Post.java`.
This will eventually become our Post model for JPA, but for now a POJO will do.
We'll add fields for `title` and `content` along with getters and setters.

```java
package com.ryantablada.entities;

public class Post {
  String title;
  String content;

  public String getTitle() {
    return this.title;
  }

  public void setTitle(String val) {
    this.title = val;
  }

  public String getContent() {
    return this.content;
  }

  public void setContent(String val) {
    this.content = val;
  }
}
```

## Creating a `GET` Route (for one Post)

To make a new route for our API, we'll respond to `/posts/1` and return a single fake `Post` object.
We'll first need a RestController in `src/main/java/com/ryantablada/controllers/PostController.java`:

```java
package com.ryantablada.controllers;

import org.springframework.web.bind.annotation.RestController;

@RestController
public class PostController {
  
}
```

By adding the `@RestController` annotation, Spring will recognize our controller and allow us to respond to URLs with methods in this class.
So...
Now we need a method called `findOnePost` in our class.
It will return a `Post` object (remember to import this from `com.ryantablada.entities.Post`).

```java
public Post findOnePost() {
    
}
```

Now let's create a new `Post` and set the `title` and `content` finally we will return this POJO post:

```java
public Post findOnePost() {
  Post post = new Post();
  post.setTitle("This is my first Post");
  post.setContent("Lorem lorem lorem");

  return post;
}
```

Before this will work, we need our new method to actually be used for API requests.
We can do this by adding an `@RequestMapping` annotation from 

```java
@RequestMapping(path = "/posts/1", method = RequestMethod.GET)
public Post findOnePost() {
  Post post = new Post();
  post.setTitle("This is my first Post");
  post.setContent("Lorem lorem lorem");

  return post;
}
```

> **NOTE** Remember to import the following: 
> * `org.springframework.web.bind.annotation.*;`

Now if we restart our server using `./gradlew buildRun`.
And then we can use cURL to request our new API resource:

```bash
curl --request GET \
  --url http://localhost:8080/posts/1 | python -m json.tool
```

And we have the output:

```json
{
    "content": "Lorem lorem lorem",
    "title": "This is my first Post"
}
```

This is good, but it's a far cry from the full JSON API spec.
We could manually build up objects or Maps for every response.
But, instead we can make a reusable set of classes to help us build this out.

## Serializing Single Records

To get started, we'll create a new `RootSerializer` to help us create the base `data` hash and `links` hashes.
This will be the entry point for our serialization.

In a new file called `src/main/java/com/ryantablada/serializers/RootSerializer.java`, we'll create the following class:

```java
package com.ryantablada.serializers;

import java.util.HashMap;
import java.util.List;
import java.util.stream.Collectors;

public class RootSerializer {

    private HashMap<String, Object> makeRoot(String resourceUrl) {
        HashMap<String, Object> result = new HashMap<>();
        HashMap<String, Object> links = new HashMap<>();

        links.put("self", resourceUrl);
        result.put("links", links);

        return result;
    }

    public HashMap<String, Object> serializeOne(String resourceUrl, HasId data, JsonDataSerializer serializer) {
        HashMap<String, Object> result = makeRoot(resourceUrl);
        result.put("data", serializer.serialize(data));

        return result;
    }

    public HashMap<String, Object> serializeMany(String resourceUrl, List<HasId> data, JsonDataSerializer serializer) {
        HashMap<String, Object> result = makeRoot(resourceUrl);
        result.put("data", data.stream().map((e) -> serializer.serialize(e)).collect(Collectors.toList()));

        return result;
    }
}
```

Here we have three methods:

* `serializeOne` - Will return a HashMap formatted to match the JSON API spec (roughly).
* `makeRoot` - Starts by adding links to the HashMap.
* `serializeMany` - This uses lambdas to map a list of items into an array of JSON API documents

If you read through this and paste the code above in your editor you'll notice two missing types:

* `HasId`
* `JsonDataSerializer`

We still need to create these, but first...
WHAT DO THESE MEAN???

Well, the `HasId` will be a simple interface that we will put on our Entities to make sure we have a `getId` function.
Then there is the `JsonDataSerializer` this will be the core of our serialization.

Let's start with the `HasId` interface in: `src/main/java/com/ryantablada/entities/HasId.java`

```java
package com.ryantablada.entities;

import java.io.Serializable;

public interface HasId extends Serializable {
  String getId();
}
```

We extend from `Serializable` just to make sure our entities are serializable to JSON without needing to add too many interfaces on our entity classes.
Speaking of that...
Let's add this interface and an `id` property to our `Post`:

```java
public class Post implements HasId {
  String title;
  String content;
  Integer id;

  public String getId() {
    return this.id.toString();
  }

  public void setId(Integer val) {
    this.id = val;
  }

  // Rest of class omitted...
}
```

## Abstract JsonDataSerializer

Now we need to work on that `JsonDataSerializer` in `src/main/java/com/ryantablada/serializers/RJsonDataSerializerootSerializer.java`:

```java
package com.ryantablada.serializers;

import java.util.Map;
import java.util.HashMap;
import java.util.stream.Collectors;
import com.ryantablada.entities.HasId;


public abstract class JsonDataSerializer {
    abstract Map<String, Object> getAttributes(HasId data);

    abstract Map<String, String> getRelationshipUrls();

    abstract String getType();

    public Map<String, Object> serialize(HasId data) {
        Map<String, Object> result = new HashMap<>();

        result.put("type", this.getType());
        result.put("id", data.getId());
        result.put("attributes", this.getAttributes(data));

        return result;
    }
}
```

This one is a bit bigger, but it has a few things.
First, we declare a few abstract methods:

* `getType` - This returns a string for the `type` in the JSON API document
* `getAttributes` - This will return a Map of properties given a single Entity (example our `Post`)
* `getRelationshipUrls` - This will be a map of the relationship urls (we'll work on relationship urls later)

Then we have the `serialize` function which splits up our `type`, `id`, and `attributes` into a valid JSON API document.
We do this by creating a simple `HashMap` with fields for `type`, `id`, and `attributes` calling our different abstract methods.

## Creating Our Post Serializer

Now that we have our interface and abstract serializer defined, we can create our `PostSerializer`.

```java
package com.ryantablada.serializers;

import java.util.HashMap;
import java.util.Map;

import com.ryantablada.entities.HasId;
import com.ryantablada.entities.Post;

public class PostSerializer extends JsonDataSerializer {
  
  public String getType() {
    return "posts";
  }

  public Map<String, Object> getAttributes(HasId entity) {
    Map<String, Object> result = new HashMap<>();
    Post post = (Post) entity;

    result.put("title", post.getTitle());
    result.put("content", post.getContent());

    return result;
  }

  public Map<String, String> getRelationshipUrls() {
    return new HashMap<String, String>();
  }
}
```

Here we setup a basic serializer.
We defined getters for type and attributes.
**Notice**, we typecasted our `HasId` into an instance of our `Post` entity.

## Setting Up Our Controller

Now our serializer is done (until we get relations).
Let's go to our controller and serialize our output.


We need to modify our `findOnePost` function.
First it should return a `Map<String, Object>`.
And we'll use our `RootSerializer`s `serializeOne` method to achieve our output:

```java
public Map<String, Object> findOnePost() {
  RootSerializer rootSerializer = new RootSerializer();
  PostSerializer postSerializer = new PostSerializer();

  Post post = new Post();
  post.setId(2);
  post.setTitle("This is my first Post");
  post.setContent("Lorem lorem lorem");

  return rootSerializer.serializeOne("/posts/1", post, postSerializer);
}
```

To the `serializeOne` function, we need to pass three arguments.

* First is our String of the current path in our URL.
* Second our data for a single Entity (remember this has to implement `HasId`)
* The serializer that will be used to define: type, how to get attributes, and eventually the relationship URLs.

Now let's restart the server and give it a go!

```bash
curl --request GET \
  --url http://localhost:8080/posts/1 | python -m json.tool
```

And there we go!!!
We have our data serialization.

```json
{
    "data": {
        "attributes": {
            "content": "Lorem lorem lorem",
            "title": "This is my first Post"
        },
        "id": "2",
        "relationships": {},
        "type": "posts"
    },
    "links": {
        "self": "/posts/1"
    }
}
```

## Serializing Lists

Since our API will need to serve Lists of entities, we should test out our code to serialize lists.
So, we'll make a new method in our controller to respond to `GET` requests to `/posts`:

```java
@RequestMapping(path = "/posts", method = RequestMethod.GET)
public Map<String, Object> findAllPost() {
  RootSerializer rootSerializer = new RootSerializer();
  PostSerializer postSerializer = new PostSerializer();

  Post post = new Post();
  post.setId(2);
  post.setTitle("This is my first Post");
  post.setContent("Lorem lorem lorem");

  Post post2 = new Post();
  post2.setId(15);
  post2.setTitle("This is my second Post");
  post2.setContent("More text goes here");

  List<HasId> posts = new ArrayList<>();
  posts.add(post);
  posts.add(post2);

  return rootSerializer.serializeMany("/posts", posts, postSerializer);
}
```

And we can restart the server and run curl:

```bash
curl --request GET \
  --url http://localhost:8080/posts | python -m json.tool
```

And we'll get the final result!

```json
{
    "data": [
        {
            "attributes": {
                "content": "Lorem lorem lorem",
                "title": "This is my first Post"
            },
            "id": "2",
            "relationships": {},
            "type": "posts"
        },
        {
            "attributes": {
                "content": "More text goes here",
                "title": "This is my second Post"
            },
            "id": "15",
            "relationships": {},
            "type": "posts"
        }
    ],
    "links": {
        "self": "/posts"
    }
}
```

Ok, so we have serialization going.
What about JSON API data coming in?
We'll cover that in the next section!
