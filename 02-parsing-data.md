# Parsing Incoming Data

Ok, so we have the ability to format and serialize data leaving our API.
But what about the JSON coming into our application?
For this we'll need to do some more work.
But, first let's add a `storePost` method to our controller that responds to `POST` requests to `/posts`:

```java
@RequestMapping(path = "/posts", method = RequestMethod.POST)
public Map<String, Object> storePost() {

}
```

## Reading Request Information

To start reading the incoming data, we'll need to add an argument to our function annotated with `@RequestBody` and typed to our `Post` entity and then we'll pass this information to our serialization layer.

```java
@RequestMapping(path = "/posts", method = RequestMethod.POST)
public Map<String, Object> storePost(@RequestBody Post post) {
  RootSerializer rootSerializer = new RootSerializer();
  PostSerializer postSerializer = new PostSerializer();

  return rootSerializer.serializeOne(
    "/posts/" + (String) post.getId(),
    post,
    postSerializer);
}
```

Now, let's try to send some data to our API!

```bash
curl --request POST \
  --url http://localhost:8080/posts \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{"title": "This is my first Post", "content": "Lorem lorem lorem", "id": 2000}' \
  | python -m json.tool
```

Here we get back the following JSON:

```json
{
  "data": {
    "attributes": {
      "content": "Lorem lorem lorem",
      "title": "This is my first Post"
    },
    "id": "2000",
    "relationships": {},
    "type": "posts"
  },
  "links": {
    "self": "/posts/2000"
  }
}
```

While this is a big step forward and our output matches the JSON API Spec...
We can't say the same about our input.

So we'll need to work on that.

## Building a Parser

To start, we need a look at what our input is going to look like.
And, it turns out that our input is going to look the same as the output we serialized (except the `links`):

```json
{
  "data": {
    "attributes": {
      "content": "Lorem lorem lorem",
      "title": "This is my first Post"
    },
    "id": "2000",
    "relationships": {},
    "type": "posts"
  }
}
```

But let's break it down.

First, everything is inside of a `data` key in the JSON object.
Then, we have the `type` (which we'll later use for type validation).
The `id` is fairly straightforward, this can be used for our update route later on.
Finally, we have the `attributes` which is going to be very helpful to fill out all the properties on our `Post` (or any Entity in our app).

We'll create our new class in `src/java/com/ryantablada/parsers/RootParser.java`.
Since our Parser will need to parse different entities, we'll make our class take a Generic.

```java
package com.ryantablada.parsers;

public class RootParser<T> {

}
```

As the heart of our parser, we'll use a special `@JsonProperty` annotation from Jackson.
This will allow Spring to take our input and know how to fill out properties in our Java objects from the incoming JSON.
We'll start by making a constructor for our class, and we'll use `@JsonProperty` to bind the `data` attribute from JSON to a new class called `JsonApiDataParser` (and we'll make a getter for this data too).
Notice that our `JsonApiDataParser` will also use our generic.

```java
package com.ryantablada.parsers;

public class RootParser<T> {
  JsonApiDataParser<T> data;

  public RootParser(
    @JsonProperty("data") JsonApiDataParser<T> data) {
    this.data = data;
  }

  public JsonApiDataParser<T> getData() {
    return this.data;
  }
}
```

Ok, so this `RootParser` unravels the `data` from our incoming JSON, but we still need to get to the `id`, `type`, and `attributes` within that `data` object.
This is where our `JsonApiDataParser` comes in:

```java
package com.ryantablada.parsers;

public class JsonApiDataParser<T> {

}
```

Here we'll once again use `@JsonProperty` to pull the `id`, `type`, and `attributes` out of the JSON payload.
BUT!
This is where our Generic will start to do some work.
By using the Generic as the type for `attributes`, we can make our parsing classes parse any Entity we want.
This prevents having to make a new parser for every Entity in our system.

```java
package com.ryantablada.parsers;

import com.fasterxml.jackson.annotation.JsonProperty;

public class JsonApiDataParser<T> {
  String type;
  String id;
  T entity;

  public JsonApiDataParser(
    @JsonProperty("type") String type,
    @JsonProperty("id") String id,
    @JsonProperty("attributes") T entity
    ) {
      this.type = type;
      this.id = id;
      this.entity = entity;
  }

  public String getType() {
    return this.type;
  }

  public String getId() {
    return this.id;
  }

  public T getEntity() {
    return this.entity;
  }
}
```

## Using Our Parser

Now we can use our parser in our controller.
We'll update our `storePost` method:

```java
@RequestMapping(path = "/posts", method = RequestMethod.POST)
public Map<String, Object> storePost(@RequestBody RootParser<Post> parser) {
  RootSerializer rootSerializer = new RootSerializer();
  PostSerializer postSerializer = new PostSerializer();
  Post post = parser.getData().getEntity();

  Integer id = Integer.parseInt(parser.getData().getId());

  post.setId(id);

  return rootSerializer.serializeOne(
    "/posts/" + (String) post.getId(),
    post,
    postSerializer);
}
```

Since we're only using `attributes` from the incoming JSON to fill in the properties of our `Post`, we need to set the `id` separately (and we need to cast it to an `Integer`).
Now let's try things out:

```java
curl --request POST \
  --url http://localhost:8080/posts \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{ "data": { "attributes": { "content": "Lorem lorem lorem", "title": "This is my first Post" }, "id": "2000", "relationships": {}, "type": "posts" } }' \
  | python -m json.tool
```

And this gives us the same JSON out, but with added `links`.

```json
{
  "data": {
    "attributes": {
      "content": "Lorem lorem lorem",
      "title": "This is my first Post"
    },
    "id": "2000",
    "relationships": {},
    "type": "posts"
  },
  "links": {
    "self": "/posts/2000"
  }
}
```

Ok, so we have our parsing and serialization good to go!
Let's start putting things in the database!!!
