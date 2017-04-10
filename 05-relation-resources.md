## Relation resources

So far we have created full CRUD APIs for **Posts** and **Comments**.
We have serialized our JSON responses to be a near full subset of the JSON API specification including the `relationships` hash with links out to where to look up related resources.
However, we haven't actually created those resources for `/posts/{id}/comments` and `/comments/{id}/post`.
Now is the time we fix that.

## Fetching a Post for a Single Comment

Let's start with our `/comments/{id}/post` API.
Here we need to return a single **Post** record for a given comment.
Let's start by creating a RestController in `/src/java/com/ryantablada/controllers/comment/CommentPostController.java`.
Here we'll autowire our `CommentRepository`, but for serialization, we'll need to serialize **Posts** using the `PostSerializer`:

```java
package com.ryantablada.controllers.comment;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import com.ryantablada.repositories.CommentRepository;
import com.ryantablada.serializers.*;

@RestController
public class CommentPostController {
  @Autowired
  CommentRepository comments;

  RootSerializer rootSerializer;
  PostSerializer postSerializer;

  public CommentPostController() {
    rootSerializer = new RootSerializer();
    postSerializer = new PostSerializer();
  }
}
```

Next we will create a method called `findPost` that will map to `/comments/{id}/post`.
And, we will load the **Comment** based on the `id` from the URL.
Then we need to serialize the `post` for the comment that we have loaded:

```java
@RequestMapping(path = "/comments/{id}/post", method = RequestMethod.GET)
public Map<String, Object> getPostComment(@PathVariable("id") String id) {
  Comment comment = comments.findOne(id);

  return rootSerializer.serializeOne("/comments/" + comment.getId() + "/post", comment.getPost(), postSerializer);
}
```

## Defining a `@OneToMany` Relation

Now on our **Post** we will need to do something similar to grab all **Comments**.
But, first we need to define the `comments` relation on our **Post** entity.
This way we'll be able to fetch the `comments` from a loaded **Post**.

```java
@Entity
public class Post implements HasId {
  @OneToMany(mappedBy = "post")
  Set<Comment> comments;

  public Set<Comment> getComments() {
    return this.comments;
  }

  // other methods and properties omitted...
}
```

Here we used a `@OneToMany` relation and specified that the opposite end of this relation is the `post` property on a single **Comment** using the `mappedBy` argument.

## Loading Comments for Posts

Now that our entity is setup, let's create our controller in `/src/java/com/ryantablada/controllers/post/PostCommentController.java`:

```java
package com.ryantablada.controllers.post;


import java.util.Map;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import com.ryantablada.entities.Post;
import com.ryantablada.repositories.PostRepository;
import com.ryantablada.serializers.CommentSerializer;
import com.ryantablada.serializers.RootSerializer;

@RestController
public class PostCommentController {
  @Autowired
  PostRepository posts;

  RootSerializer rootSerializer;
  CommentSerializer commentSerializer;

  public PostCommentController() {
    rootSerializer = new RootSerializer();
    commentSerializer = new CommentSerializer();
  }

  @RequestMapping(path = "/posts/{id}/comments", method = RequestMethod.GET)
  public Map<String, Object> getCommentsPost(@PathVariable("id") String id) {
    Post post = posts.findOne(id);

    return rootSerializer.serializeMany("/posts/" + post.getId() + "/comments", post.getComments(), commentSerializer);
  }
}
```

Now let's try to give this a shot.
If we follow any relation link, we can navigating through our API.

For instance here is the results for GET - `/posts`:

```json
{
  "data": [
    {
      "relationships": {
        "comments": {
          "links": {
            "related": "/posts/402880ec5b54b676015b54b935710000/comments"
          }
        }
      },
      "attributes": {
        "title": "This is my first Post",
        "content": "Lorem lorem lorem"
      },
      "id": "402880ec5b54b676015b54b935710000",
      "type": "posts"
    },
    {
      "relationships": {
        "comments": {
          "links": {
            "related": "/posts/402880ec5b54b676015b54b948d50001/comments"
          }
        }
      },
      "attributes": {
        "title": "This is my first Post",
        "content": "Lorem lorem lorem"
      },
      "id": "402880ec5b54b676015b54b948d50001",
      "type": "posts"
    }
  ],
  "links": {
    "self": "/posts"
  }
}
```

Then following `/posts/402880ec5b54b676015b54b948d50001/comments` results in:

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
    "self": "/posts/402880ec5b54b676015b54b948d50001/comments"
  }
}
```

Then the post relation link can be followed to `/comments/402880ec5b54b676015b54d3cc400003/post`:

```json
{
  "data": {
    "relationships": {
      "comments": {
        "links": {
          "related": "/posts/402880ec5b54b676015b54b948d50001/comments"
        }
      }
    },
    "attributes": {
      "title": "This is my first Post",
      "content": "Lorem lorem lorem"
    },
    "id": "402880ec5b54b676015b54b948d50001",
    "type": "posts"
  },
  "links": {
    "self": "/comments/402880ec5b54b676015b54d3cc400003/post"
  }
}
```

This self describing set of links allows front-end applications to follow the links only when more data needs to be loaded (or even reloaded).

Now that we have our API working, we can add user registration and login next.
