# Java Spring JSON API with JWT Tutorial

This tutorial will guide you through how to create a Blog API with JSON API and JWTs.

The final code for this project can be found at: https://github.com/rtablada/java-blog-api

## Configuration

First, we will add `application-dev.properties` to our gitignore file to make sure our sensitive data and personal configuration will not be shared across git repos.

```bash
echo "**/application-dev.properties" >> .gitignore
```

Next, let's copy our existing `application.properties` file into `application-dev.properties`.
Add any personal customization (example your database username or secret API keys).

Finally, we need a way to tell spring to load configuration from the `dev` profile instead of the normal `application.properties`.

To run our app with a given profile we can set a `SPRING_PROFILES_ACTIVE` environment variable.
For instance to use our `dev` profile:

```bash
SPRING_PROFILES_ACTIVE=dev ./gradlew bootRun
```

But, this is easy to forget so let's create a small bash script for this:

```bash
echo "SPRING_PROFILES_ACTIVE=dev ./gradlew bootRun" > ./dev.sh
chmod 777 dev.sh
```

> **NOTE** If you add a property to `application-dev.properties` remember to tell your team about the new configuration property.
