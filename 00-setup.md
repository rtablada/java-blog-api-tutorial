# Setting Up a Spring Server

For this application we'll be creating a [JSON API](http://jsonapi.org/) blog application with user authentication via [JWTs](https://jwt.io/).

> **NOTE** We won't be supporting the FULL specification for JSON API since this would be pretty intensive and involves dynamic eager-loading and a lot of serialization.
> Frameworks and tools like [Elide](http://elide.io/) cover much more of the specification but also add complexity.

## Understanding the problem

In this application, we'll be building a basic API for a blog platform.
The front-end will be an Ember App using Ember Data, but any front-end can consume the API we'll create.

For our server we will need three tables and JPA entities:

* User
* Post
* Comment

For this API, we will be exposing the following API end points:

* `/users`
  - GET - `/` - Get all **Users**
  - GET - `/:id` - Get one **User** by `:id`
  - GET - `/:id/posts` - Get the **Posts** created by the given **User**
  - POST - `/` - Create a new **User**
  - PATCH - `/:id` - Update a **User** by `:id` (Requires authentication and only update the current user)
* `/posts`
  - GET - `/` - Get all **Posts**
  - GET - `/:id` - Get one **Post** by `:id`
  - POST - `/` - Create a new **Post** (Requires authentication and sets the **User** based on authenticated **User**)
  - GET - `/:id/user` - Get the **User** who created the **Post** with a given `:id`
  - GET - `/:id/comments` - Get the **Comments** who created the **Post** with a given `:id`
  - PATCH - `/:id` - Update a **Post** by `:id` (Requires authentication and only allow update by the user who created it)
  - DELETE - `/:id` - Delete a **Post** by `:id` (Requires authentication and only allow update by the user who created it)
* `/comments`
  - GET - `/` - Get all **Comments**
  - GET - `/:id` - Get one **Comment** by `:id`
  - POST - `/` - Create a new **Comment** (Doesn't require authentication, but does require a specified **Post**)
  - GET - `/:id/post` - Get the **Post** for the **Comment** based on `/:id`

## Creating the Spring Boot App

To get started with our Spring App, we'll go to http://start.spring.io.

First, we need to Generate a `Gradle Project` with Spring Boot `1.5.2`.
From there we'll set our `Group` and `Artifact`:

* Group: `com.ryantablada`
* Artifact: `blog-api`

Then we need to setup our dependencies.

* Core: Security
* Core: DevTools
* Web: Web
* SQL: JPA
* SQL: PostgreSQL

Now we can hit the big "Generate Project" button.
This will download a `.zip` of our project that we need to unpack.

## Setting up Git

Before we get started, we'll use `git` to track our project changes.
After unzipping our project where we want it, we can use our terminal to `cd` into the project directory and run `git init`.

Then we'll commit the starting project so we have a starting reference.

```bash
git add .
git commit -m "Start project"
```

## Configuring the Database

Now we need to go into `/src/main/resources/application.properties` to setup the app configuration:

* `server.port = 8080` - Sets Spring and Tomcat to listen to port `8080`
* `spring.datasource.url=jdbc:postgresql://localhost:5432/blog-api` - Sets JPA to talk to a local PG database called `blog-api`
* `spring.datasource.username=` - Set this to your PostgreSQL username
* `spring.datasource.password=` - Set this to your PostgreSQL password
* `spring.jpa.hibernate.ddl-auto=update` - This will make JPA update our database to match our Entity Models

Finally we need to create a new database called `blog-api` (you can name this whatever as long as it matches the database name in our `spring.datasource.url` config).

```
createdb blog-api
```

## Starting the Server

We can start the server from our terminal with Gradle:

```bash
./gradlew bootRun
```

Now we can try to visit our API:

```bash
curl --request GET \
  --url http://localhost:8080/posts
```

This gives us back the following result: 

```json
{
  "timestamp":1491525064905,
  "status":401,
  "error":"Unauthorized",
  "message":"Full authentication is required to access this resource",
  "path":"/posts"
}
```

## Disabling Security (For now)

Since we'll be handling authentication and security in a later tutorial, we'll go to our `build.gradle` file and comment out the line that says: 

```
compile('org.springframework.boot:spring-boot-starter-security')
```

We'll need to restart our `./gradlew buildRun` command.
If we rerun the cURL command from earlier now we get:

```json
{
    "error": "Not Found",
    "message": "No message available",
    "path": "/posts",
    "status": 404,
    "timestamp": 1491525377066
}
```

That's what we expect since we haven't started writing any controllers yet.
So now we're good to get going!
