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

  return this.rootSerializer.serializeOne("/comments" + comment.getId(), comment, this.commentSerializer);
}
```

Then let's try this out.

```bash
curl --request POST \
  --url http://localhost:8080/comments \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{ "data": { "attributes": { "content": "Lorem lorem lorem" }, "type": "comments" } }'
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

  System.out.println(comment.getPost().getId());

  return this.rootSerializer.serializeOne("/comments" + comment.getId(), comment, this.commentSerializer);
}
```
