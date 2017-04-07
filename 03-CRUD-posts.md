# CRUD Posts

Ok, we have all of our data translation layers.
Let's add JPA to the mix.
We'll start by adding persistence annotations and a `serialVersionUID` to our `Post` entity.
Here we'll also use `GeneratedValue` and `GenericGenerator` annotations to auto generate UUIDs for our Posts.

```java
@Entity
public class Post implements HasId {
  private static final long serialVersionUID = 1L;

  @Column
  String title;

  @Column(columnDefinition = "text")
  String content;

  @Id
  @GeneratedValue(generator="system-uuid")
  @GenericGenerator(name="system-uuid", strategy = "uuid")
  String id;

  // getters and setters omitted
}
```

## Post Repository

In order to persist or fetch data records from the database, we'll need a `PostRepository`.
Let's make a new `src/java/com/ryantablada/repositories/PostRepository.java` that extends `CRUDRepository`:

```java
package com.ryantablada.repositories;

import org.springframework.data.repository.CrudRepository;
import com.ryantablada.entities.Post;

public interface PostRepository extends CrudRepository<Post, Integer> {

}
```

## Saving Posts

Ok, so we have our Entity and our `CrudRepository`.
Let's try to save a post!

We'll need to update our `PostController`.
First, we need to add an `@Autowired` version of our `PostRepository`:

```java
@RestController
public class PostController {

  @Autowired
  PostRepository posts;

  // methods omitted...

}
```

Then let's update the `storePost` method to save our input to the database:

```java
@RequestMapping(path = "/posts", method = RequestMethod.POST)
public Map<String, Object> storePost(@RequestBody RootParser<Post> parser) {
  RootSerializer rootSerializer = new RootSerializer();
  PostSerializer postSerializer = new PostSerializer();
  Post post = parser.getData().getEntity();

  posts.save(post);

  return rootSerializer.serializeOne(
    "/posts/" + (String) post.getId(),
    post,
    postSerializer);
}
```

Notice that we don't have to set our `id` property since JPA and hibernate will auto generate this value when our post is saved to the database!
Let's try it out:

```bash
curl --request POST \
  --url http://localhost:8080/posts \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{ "data": { "attributes": { "content": "Lorem lorem lorem", "title": "This is my first Post" }, "type": "posts" } }' \
  | python -m json.tool
```

And from this, we get the following output:

```json
{
  "data": {
    "attributes": {
      "content": "Lorem lorem lorem",
      "title": "This is my first Post"
    },
    "id": "402880e95b4883d2015b488c878d0004",
    "type": "posts"
  },
  "links": {
    "self": "/posts/402880e95b4883d2015b488c878d0004"
  }
}
```

> **NOTE** Since UUIDs are always unique, your `id` and `self` link will be different than mine.

If we run `psql blog-api` we can inspect the databae to check that our post was actually saved.
Here we'll run `SELECT * FROM post`:


id                               | content           | title
---------------------------------|-------------------|----------------------
402880e95b4883d2015b488768b60003 | Lorem lorem lorem | This is my first Post

Before moving on, let's save one more post so that we have something to work with:

```java
curl --request POST \
  --url http://localhost:8080/posts \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{ "data": { "attributes": { "content": "Something interesting", "title": "Another Post" }, "type": "posts" } }' \
  | python -m json.tool
```

## Listing All Posts

Now that we have a few posts in our database, let's work on listing all posts.
Here we'll move our our hard coded Posts from the `findAllPost` method and replace them with a call to `posts.findAll`.

> **NOTE** Since `findAll` returns an `Iteratable` instead of a `List`, we have to do a bit of extra work...

```java
@RequestMapping(path = "/posts", method = RequestMethod.GET)
  public Map<String, Object> findAllPost() {
    RootSerializer rootSerializer = new RootSerializer();
    PostSerializer postSerializer = new PostSerializer();

    Iterator<Post> results = posts.findAll().iterator();

    List<HasId> resultsList = new ArrayList<>();

    results.forEachRemaining(resultsList::add);

    return rootSerializer.serializeMany("/posts", resultsList, postSerializer);
  }
```

Now let's try to fetch our data:

```bash
curl --request GET \
  --url http://localhost:8080/posts \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  | python -m json.tool
```

And our result:

```json
{
  "data": [
    {
      "attributes": {
        "content": "Lorem lorem lorem",
        "title": "This is my first Post"
      },
      "id": "402880e95b4883d2015b488768b60003",
      "type": "posts"
    },
    {
      "attributes": {
        "content": "Something interesting",
        "title": "Another Post"
      },
      "id": "402880e95b4883d2015b4890594b0005",
      "type": "posts"
    }
  ],
  "links": {
    "self": "/posts"
  }
}
```

## Reducing Complexity

Since we'll be using Hibernate's `findAll` to work with many of our data results, we'll need to work with this iterator pattern a lot in our controllers.
But instead, we can overload our `serializeMany` method to accept an `Iterable` and turn it into our array list.

```java
public HashMap<String, Object> serializeMany(String resourceUrl, Iterable<?> data, JsonDataSerializer serializer) {
  Iterable<HasId> results = (Iterable<HasId>) data;
  List<HasId> resultsList = new ArrayList<>();

  results.iterator().forEachRemaining(resultsList::add);

  return this.serializeMany(resourceUrl, resultsList, serializer);
}
```

Now we can update our controller.

```java
@RequestMapping(path = "/posts", method = RequestMethod.GET)
public Map<String, Object> findAllPost() {
  RootSerializer rootSerializer = new RootSerializer();
  PostSerializer postSerializer = new PostSerializer();

  Iterable<HasId> results = posts.findAll();

  return rootSerializer.serializeMany("/posts", results, postSerializer);
}
```
