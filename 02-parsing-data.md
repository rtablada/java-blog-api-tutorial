# Parsing Incoming Data

Ok, so we have the ability to format and serialize data leaving our API.
But what about the JSON coming into our application?
For this we'll need to do some more work.
But, first let's add a `storePost` method to our controller that responds to `POST` requests to `/posts`:

```java
@RequestMapping(path = "/post", method = RequestMethod.POST)
public Map<String, Object> storePost() {

}
```

## Reading Request Information

To start reading the incoming data, we'll need to add an argument to our function annotated with `@RequestBody` and typed to our `Post` entity and then we'll pass this information to our serialization layer.

```java
@RequestMapping(path = "/post", method = RequestMethod.POST)
public Map<String, Object> storePost(@RequestBody Post post) {
  RootSerializer rootSerializer = new RootSerializer();
  PostSerializer postSerializer = new PostSerializer();

  return rootSerializer.serializeOne(
    "/posts/" + (String) post.getId().toString(),
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
