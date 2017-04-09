# Relationship Input

Now let's start working on the API for creating **Comments**.

To start, we'll create a Comment entity that will only have a text field for `content`.
And we will have our `id`.

```java
package com.ryantablada.entities;

import javax.persistence.*;
import org.hibernate.annotations.GenericGenerator;

@Entity
public class Comment implements HasId {
  private static final long serialVersionUID = 1L;

  @Column(columnDefinition = "text")
  String content;

  @Id
  @GeneratedValue(generator="system-uuid")
  @GenericGenerator(name="system-uuid", strategy = "uuid")
  String id;

  public String getId() {
    return id;
  }

  public void setId(String val) {
    this.id = val;
  }

  public String getContent() {
    return this.content;
  }

  public void setContent(String val) {
    this.content = val;
  }
}
```

But, we're not done, we need to define that our **Comment** should point to the post we are commenting on (imagine a comment thread on a single blog post).
For this, we'll use the persistence `@ManyToOne` annotation, and we'll create setters and getters:

```java
@ManyToOne
Post post;

public Post getPost() {
  return post;
}

public void setPost(Post val) {
  this.post = val;
}
```

Finally, we need to be able set a post just based on the String `id` of the Post since we won't be able to send an actual Java **Post** object through the API.
To do this, we'll just make a temporary **Post** and, we'll set its `id` and pass to our regular setter:

```java
public void setPost(String id) {
  Post post = new Post();
  post.setId(id);

  this.setPost(post);
}
```

So our final class looks like:

```java
package com.ryantablada.entities;

import javax.persistence.*;
import org.hibernate.annotations.GenericGenerator;

@Entity
public class Comment implements HasId {
  private static final long serialVersionUID = 1L;

  @ManyToOne
  Post post;

  @Column(columnDefinition = "text")
  String content;

  @Id
  @GeneratedValue(generator="system-uuid")
  @GenericGenerator(name="system-uuid", strategy = "uuid")
  String id;

  public String getId() {
    return id;
  }

  public void setId(String val) {
    this.id = val;
  }

  public Post getPost() {
    return post;
  }

  public void setPost(Post val) {
    this.post = val;
  }

  public void setPost(String id) {
    Post post = new Post();
    post.setId(id);

    this.setPost(post);
  }

  public String getContent() {
    return this.content;
  }

  public void setContent(String val) {
    this.content = val;
  }
}
```

## Repository

Now we need to create our Comment CRUDRepository:

```java
package com.ryantablada.repositories;

import org.springframework.data.repository.CrudRepository;
import com.ryantablada.entities.Comment;

public interface CommentRepository extends CrudRepository<Comment, String> {

}
```

## Controller

Now we're ready to create a controller to work with Comments in `src/java/com/ryantablada/controllers/CommentController.java`:

```java
package com.ryantablada.controllers;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RestController;

import com.ryantablada.repositories.CommentRepository;

@RestController
public class CommentController {

  @Autowired
  CommentRepository comments;

}
```

Then we can make a function called `storeComment` that will respond to incoming `POST` requests to `/comments`.

```java
@RequestMapping(path  = "/comments", method = RequestMethod.POST)
public Map<String, Object> storeComment(@RequestBody RootParser<Comment> parser) {

}
```

## Serializing Data Out for Comments

Now we have our `storeComment` function started, but we need to think about how we can turn a `Post` into our `HashMap` output.
Luckily, we can extend from the `JsonDataSerializer` that we made in part 2, we just need to define the output for our `Comment`.
So, let's create a new file `src/java/com/ryantablada/serializers/CommentSerializer.java`.
Here we'll need to extend from `JsonDataSerializer` and define `getType`, `getAttributes`, and `getRelationshipUrls`:

```java
package com.ryantablada.serializers;

import java.util.HashMap;
import java.util.Map;

import com.ryantablada.entities.HasId;
import com.ryantablada.entities.Comment;

public class CommentSerializer extends JsonDataSerializer {

  public String getType() {
    return "comments";
  }

  public Map<String, Object> getAttributes(HasId entity) {
    Map<String, Object> result = new HashMap<>();
    Comment post = (Comment) entity;

    result.put("content", post.getContent());

    return result;
  }

  public Map<String, String> getRelationshipUrls() {
    return new HashMap<String, String>();
  }
}
```

Now we can go back to our controller and setup our serialization:

```java
RootSerializer rootSerializer;
CommentSerializer commentSerializer;

public CommentController() {
  this.rootSerializer = new RootSerializer();
  this.commentSerializer = new CommentSerializer();
}


@RequestMapping(path  = "/comments", method = RequestMethod.POST)
public HashMap<String, Object> storeComment(@RequestBody RootParser<Comment> parser) {
  Comment comment = parser.getData().getEntity();

  return this.rootSerializer.serializeOne("/comments/" + comment.getId(), comment, this.commentSerializer);
}
```

Then let's try this out.

```bash
curl --request POST \
  --url http://localhost:8080/comments \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{ "data": { "attributes": { "content": "Lorem lorem lorem" }, "type": "comments" } }' \
  | python -m json.tool
```

Good, but what about the **Post** relation?

Well from our input JSON the JSON API spec says that things should look a bit like this:

```json
{
  "data": {
    "type": "comments",
    "attributes": {
      "content": "Lorem lorem lorem"
    },
    "relationships": {
      "post": {
        "data": {
          "id": "12345",
          "type": "post"
        }
      }
    }
  }
}
```

We need a way to get the post id from this data, so we'll need to add a new feature to our JsonApiDataParser.
We'll start by defining a new property creatively called `relationships` that will be a Map of `<String, RootParser>` (this recursive nature of making this a recursive HashMap actually allows us to support embedded records in the future).
Then, we'll define a `getRelationshipId` function that grabs the `id` from a particular relation based on a String argument.

```java
public String getRelationshipId(String relationName) {
  return this.relationships.get(relationName).getData().getId();
}
```

## Using `getRelationshipId`

Now in our controller, we need to set the `user` for our new comment based on the incoming `id` for the `post` relation.
Let's use `System.out.println` to print the `id` from our **Comment's** related Post:

```java
@RequestMapping(path  = "/comments", method = RequestMethod.POST)
public HashMap<String, Object> storeComment(@RequestBody RootParser<Comment> parser) {
  Comment comment = parser.getData().getEntity();
  comment.setPost(parser.getData().getRelationshipId("post"));

  System.out.println("Post id: " + comment.getPost().getId());

  return this.rootSerializer.serializeOne("/comments/" + comment.getId(), comment, this.commentSerializer);
}
```

Now, let's try to **Comment** a new comment using cURL:

```bash
curl -X POST \
  http://localhost:8080/comments \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{ "data": { "type": "comments", "attributes": { "content": "Lorem lorem lorem" }, "relationships": { "post": { "data": { "id": "12345", "type": "post" } } } } }' \
  | python -m json.tool
```

This gives us the same JSON output as before, but let's look in our console where we're running `./gradlew bootRun`.
Here we should see a single line of output that says:

```
Post id: 12345
```

So, we have our post set on our comment.
Now, we just need to save this using our repository:

```java
public HashMap<String, Object> storeComment(@RequestBody RootParser<Comment> parser) {
  Comment comment = parser.getData().getEntity();
  comment.setPost(parser.getData().getRelationshipId("post"));

  comments.save(comment);

  return this.rootSerializer.serializeOne("/comments/" + comment.getId(), comment, this.commentSerializer);
}
```

Since our comments use a foreign key in our database, we can't just use `1234` as our **Post** id.
So to get one post, we need to get a result:

```bash
curl -X GET \
  http://localhost:8080/posts \
  | python -m json.tool
```

Then take the `id` from one of the **Posts** returned by the API.
If you don't see any posts in your results, look back at some of the cURL requests from the last lesson.
In my case, there is a Post with an id of `402880ec5b54b676015b54b948d50001` (yours will be different since it is a UUID).
So, now we can use this to create a new **Comment**:

```bash
curl -X POST \
  http://localhost:8080/comments \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{
  "data": {
    "type": "comments",
    "attributes": {
      "content": "Lorem lorem lorem"
    },
    "relationships": {
      "post": {
        "data": {
          "id": "402880ec5b54b676015b54b948d50001",
          "type": "post"
        }
      }
    }
  }
}' | python -m json.tool
```

The result will look something like this:

```json
{
  "data": {
    "attributes": {
      "content": "Lorem lorem lorem"
    },
    "id": "402880ec5b54b676015b54c9e3430002",
    "type": "comments"
  },
  "links": {
    "self": "/comments/402880ec5b54b676015b54c9e3430002"
  }
}
```

We'll have to use `psql` to check if this all worked:

```bash
psql blog-api -c "SELECT * FROM comment"
```

And here is the resulting table:

id                               | content           | post_id
---------------------------------|-------------------|---------------------------------
402880ec5b54b676015b54c9e3430002 | Lorem lorem lorem | 402880ec5b54b676015b54b948d50001

## Full Comment CRUD

Now that we have a **Comment** in the database, we can start filling out the rest of our CRUD routes (similar to what we did for **Post** CRUD):

```java
@RestController
public class CommentController {

  @Autowired
  CommentRepository comments;

  RootSerializer rootSerializer;
  CommentSerializer commentSerializer;

  public CommentController() {
    this.rootSerializer = new RootSerializer();
    this.commentSerializer = new CommentSerializer();
  }

  @RequestMapping(path = "/comments", method = RequestMethod.GET)
  public HashMap<String, Object> findAllComment() {
    Iterable<Comment> results = comments.findAll();

    return rootSerializer.serializeMany("/comments", results, commentSerializer);
  }

  @RequestMapping(path = "/comments/{id}", method = RequestMethod.GET)
  public HashMap<String, Object> findOneComment(@PathVariable("id") String id) {
    Comment comment = comments.findOne(id);

    return rootSerializer.serializeOne("/comments/" + comment.getId(), comment, commentSerializer);
  }


  @RequestMapping(path  = "/comments", method = RequestMethod.POST)
  public HashMap<String, Object> storeComment(@RequestBody RootParser<Comment> parser) {
    Comment comment = parser.getData().getEntity();
    comment.setPost(parser.getData().getRelationshipId("post"));

    comments.save(comment);

    return this.rootSerializer.serializeOne("/comments/" + comment.getId(), comment, this.commentSerializer);
  }

  @RequestMapping(path = "/comments/{id}", method = RequestMethod.PATCH)
  public HashMap<String, Object> updateComment(@PathVariable("id") String id, @RequestBody RootParser<Comment> parser) {
    Comment existingComment = comments.findOne(id);
    Comment input = parser.getData().getEntity();

    existingComment.setContent(input.getContent());
    existingComment.setPost(parser.getData().getRelationshipId("post"));

    comments.save(existingComment);

    return rootSerializer.serializeOne("/comments/" + existingComment.getId(), existingComment, commentSerializer);
  }

  @RequestMapping(path = "/comments/{id}", method = RequestMethod.DELETE)
  public void deleteComment(@PathVariable("id") String id, HttpServletResponse response) {
    comments.delete(id);

    response.setStatus(204);
  }
}
```

Now we have full CRUD for **Comments**.

## What about the Post?

We've looked into the database and know that our **Comments** are being "attached" to **Posts**.
But, this isn't exposed to our API.
While we would ideally support "sideloading" or "embedding" of relational data (which is supported by the JSON API spec), this introduces a large set of complexity.
Luckily, the JSON API spec allows for relative links.
This means we can use a url like `comments/12345/post` to get the **Post** for **Comment** `12345`.
But, we do need to make this URL known to end users.

For this, we'll add two methods to our `JsonDataSerializers`.
First, let's implement the `getRelationshipUrls` method in our `CommentSerializer`:

```java
public Map<String, String> getRelationshipUrls() {
  return new HashMap<String, String>() {{
      put("post", "/comments/{id}/post");
  }};
}
```

Here we're saying that the `post` relationship can be fetched by going to `/comments/{id}/posts` (we'll replace `{id}` with the actual `id` from our **Comment**).

While we're at it, let's define a similar URL in our `PostSerializer`:

```java
public Map<String, String> getRelationshipUrls() {
  return new HashMap<String, String>() {{
      put("comments", "/posts/{id}/comments");
  }};
}
```

Now, we need to write a method in our abstract `JsonDataSerializer` to take this and actually add it to the response JSON.
We'll call this `getRelationships`:

```java
private Map<String, Object> getRelationships(String id) {
    return this.getRelationshipUrls()
            .entrySet().stream()
            .collect(Collectors.toMap((entry) -> entry.getKey(),
                    (entry) -> {
                        Map<String, Object> result = new HashMap<>();
                        Map<String, Object> links = new HashMap<>();
                        String url = entry.getValue();

                        links.put("related", url.replace("{id}", id));

                        result.put("links", links);

                        return result;
                    }));
}
```

This looks like a lot and is the most complex function in our app so far.
Let's take this one step at a time:

1. We call `getRelationshipUrls` which will return our `Map<String, String>` defining the relationship url paths
2. We take the `Map` and turn it into a stream that we can modify
3. We call `Collectors.toMap` which allows us to create a new `Map` with two functions:
    1. The key of the `Map`
    2. The value of the `Map`

Mapping the key for the relationship is fairly simple: we just output the key from the original relationshipUrl `Map`.
Getting the formatted output is a bit harder.
To match the JSON API spec as closely as we can, we need to turn the string `/comments/{id}/post` into the following JSON (given a **Comment** with the id `12345`):

```json
{
  "links": {
    "related": "/comments/12345/post"
  }
}
```

This is what we're doing in the second Lambda expression.
We first create our `links` and result `HashMaps`.
Then we replace the `{id}` string with our actual resource `id` and put it in the `links` HashMap as `related`.
Then, we wrap it all up and return it.

There's one last step: we need to call our `getRelationships` method when we serialize a record:

```java
public Map<String, Object> serialize(HasId data) {
    Map<String, Object> result = new HashMap<>();

    result.put("type", this.getType());
    result.put("id", data.getId());
    result.put("attributes", this.getAttributes(data));
    result.put("relationships", this.getRelationships(data.getId()));

    return result;
}
```

Now let's try to look at our `/comments` API:

```bash
curl -X GET \
  http://localhost:8080/comments \
  | python -m json.tool
```

And now we have a result that looks a bit like this:

```json
{
  "data": [
    {
      "relationships": {
        "post": {
          "links": {
            "related": "/comments/402880ec5b54b676015b54d3cc400003/post"
          }
        }
      },
      "attributes": {
        "content": "Why"
      },
      "id": "402880ec5b54b676015b54d3cc400003",
      "type": "comments"
    }
  ],
  "links": {
    "self": "/comments"
  }
}
```

Notice that we now have the `relationships` data along side our `id`, `type`, and `attributes`.

In the next lesson, we'll actually implement the API resources that allow us to get the **Post** for a **Comment** with a url `/comments/12345/post`.
