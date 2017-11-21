---
  title: "GraphQL intro: from basic to PostgreSQL"
---
# GraphQL intro: from basic to PostgreSQL

In this post, I'll walk you through building a simple API using GraphQL that persists data on
PostgreSQL. For that, we'll use the `graphql-js` package and NodeJS. If you decide to jump right into
the code, you can take a look at the repository I created with the [final version of the code described here](https://github.com/nubunto/rickandmorty_friends).

I assume that the reader is somewhat familiar with GraphQL terminology (e.g. Queries, Mutations and Input Types)
and also has experience with both NodeJS (especially yarn) and a relational database.

If the reader is not familiar with GraphQL, I suggest reading [their website](https://graphql.org/learn) first.

Let's discuss and present some of the technologies we'll be using in this post.

# GraphQL

GraphQL stands as a typed query language for APIs. It is meant to be an improvement from REST,
although it is concise enough to sit together with an existing one or, even, be put in front of one.

GraphQL lets you make queries for data, and returns you specifically what you asked for. Nothing more, 
nothing less. Let's take a look:

```
{
  user {
    id
    name
    email
  }
}
```

This is a hypothetical GraphQL query, in a shorthand notation. An response from the API server could be something
along the lines of:

```
{
  "data": {
    "user": {
      "id": 1,
      "name": "John McCarthy",
      "email": "john@somewhere.com"
    }
  }
}
```

Plain old JSON, resembling very closely what we specified in our request.
There are other aspects of GraphQL which we'll get to know during the building of our application, but for now, this is all I'll show.

# NodeJS

NodeJS is well known in the programming large. It is a port of the Google Chrome V8 JavaScript engine, and is used by large companies worldwide.
With it, we'll be able to spawn a HTTP server that will handle our GraphQL requests and send responses.

We'll be using the latest version of node (v7.4.0 as of writing).

## Koa

I won't be sticking with express for HTTP, but koa. The reason being I believe koa to be more idiomatic and easier to follow in general
given it's generator-based nature, as opposed to callbacks. That being said, it shouldn't be hard to move from koa to express, since there
are packages that support GraphQL for express.

# PostgreSQL

A robust, rock-solid relational database. Nothing more to be said.

# What we'll be building

If you're not a fan of Rick & Morty, I expect you to walk out of this blog post as one. We'll be modelling their Universe (Multiverse, to be precise),
by creating a database of characters and their respective friends (Adult Swim, give me free stuff). We'll go from SQL statements, modelling our relational database, to the GraphQL spec. Hopefully, this will
give a feeling of how to write an application that does something real, and not (too much) contrived.

Let's get started, then!

# Our database models

I won't be covering PostgreSQL installation here, as that can be a bit lengthy and change from case-to-case, but I'll give you a hint: install [docker](https://docker.com)
and start a database using [the official PostgreSQL docker image](https://hub.docker.com/_/postgres). Then, you can have a PostgreSQL database up and running
by using this command: 

`$ docker run --name rickandmortypostgres -p 5432:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=rickandmorty -d postgres:9.6.1-alpine`

I'm using the 9.6.1-alpine version of Docker. The "alpine" part means that this container is running in a Alpine Linux OS, which actually counts towards a smaller image and, as a consequence,
faster to download. If you wish to install PostgreSQL locally, please stick to the 9.6 version, since that version is the latest as of this writing. I can't guarantee any of the statements to work
on newer versions, although they are very simple and you shouldn't have problems.

_Note: Our user **and** password will be `postgres`. Do **NOT** use this in production. Protect your database carefully._

Once we have a database setup and running, we can create the Rick & Morty database from the command line:
```
createdb -U postgres rickandmorty
```
_Note: you can skip the above if using Docker, as we provided `rickandmorty` to it as the default database._

If you have a running PostgreSQL instance on your machine, you already have the `psql` command line utility, so you can use that to connect to your instance.

If you have PostgreSQL running through Docker, you can run this command to get a `psql` running on your database:

```
$ docker run -it --rm --link rickandmortypostgres:postgres postgres:9.6.1-alpine psql -d rickandmorty -h rickandmorty -U postgres
```

We can now start by creating our tables:

```
CREATE TABLE character (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL
)

CREATE TABLE friend (
  id SERIAL PRIMARY KEY,
  character_id INTEGER REFERENCES character (id)
  friend_id INTEGER REFERENCES character (id)
)
```

The `character` table contains the ID and name of our characters. Every character has many friends, so on the `friend` table, we have an foreign key to
the respective character and its friends.

Let's take this model for a spin first, to validate our assumptions and to also provide some seed data:

```
INSERT INTO character (name)
  VALUES ('Rick Sanchez'), 
	 ('Morty Smith'),
	 ('Beth Sanchez'),
	 ('Jerry Smith'),
	 ('Summer Smith');

INSERT INTO friend (character_id, friend_id) 
VALUES (1, 2),
       (1, 3), 
       (1, 5), 
       (2, 3), 
       (2, 4), 
       (2, 5), 
       (3, 1), 
       (3, 2), 
       (3, 4), 
       (3, 5),
       (4, 2),
       (4, 3),
       (4, 5),
       (5, 1),
       (5, 2),
       (5, 3),
       (5, 4);
```

We modeled our relationship graph that looks like this:

Rick is friends with Morty, Beth and Summer. He despises Jerry.

Morty is friends with Rick, Beth, Jerry and Summer. After all, they are his family.

Beth is friends with Rick, Morty, Jerry and Summer.

Jerry is friends with Morty, Beth and Summer. He doesn't like Rick.

Great, now let's select all of Rick Sanchez friends:

```
SELECT
  c.name as 'Character Name', f.name as 'Friend name'
FROM
  character c
  JOIN friend ff ON c.id = ff.character_id
  JOIN character f ON f.id = ff.friend_id
WHERE
  name = 'Rick Sanchez'
```

And, sure enough, here are our results:
```
 Character Name | Friend name  
----------------+--------------
 Rick Sanchez   | Morty Smith
 Rick Sanchez   | Beth Sanchez
 Rick Sanchez   | Summer Smith
(3 rows)

```

Our schema is ready! Let's move on to the part where we _burp_ actually go through with this stuff, y'know.

# Building our GraphQL Schema

Let's take a moment to think about what our GraphQL schema will look like.
What I want to be able to do is something along the lines of:

```
{
  characters {
    name
    friends {
      name
    }
  }
}
```

And get something like this back:
```
{
  "data": {
    "characters": [
      {
	"name": "Rick Sanchez",
	"friends": [
	  { "name": "Morty Smith" },
	  { "name": "Beth Sanchez" },
	  { "name": "Summer Smith" }
	]
      },
      {
	"name": "Morty Smith",
	"friends": [
	  { "name": "Rick Sanchez" },
	  { "name": "Beth Sanchez" },
	  { "name": "Jerry Smith" },
	  { "name": "Summer Smith" }
	]
      },
      {
	"name": "Beth Sanchez",
	"friends": [
	  { "name": "Rick Sanchez" },
	  { "name": "Morty Smith" },
	  { "name": "Jerry Smith" },
	  { "name": "Summer Smith" }
	]
      },
      {
	"name": "Jerry Smith",
	"friends": [
	  { "name": "Morty Smith" },
	  { "name": "Beth Sanchez" },
	  { "name": "Summer Smith" }
	]
      },
      {
	"name": "Summer Smith",
	"friends": [
	  { "name": "Rick Sanchez" },
	  { "name": "Morty Smith" },
	  { "name": "Beth Sanchez" },
	  { "name": "Jerry Smith" },
	]
      }
    ]
  }
}
```

As it stands right now, we can start seeing the first real benefit of GraphQL.

In a REST API that was equivalent of this, we would probably have to make 1 request to get all the characters,
**and** we would also need to make N requests to fetch each friend of **each** character. In this case, the total
count of requests would be **6 requests** in order to fetch all of this data (1 for all + 1 for each character returned from the previous request).
Also, we would have to iterate the returned characters, and making a request for each character `id` we found. 

Something along the lines of
```
$ curl http://localhost:8080/characters
$ curl http://localhost:8080/characters/1/friends
$ curl http://localhost:8080/characters/2/friends
# and so on.
```

GraphQL can do all this in one bite.

Other real benefit is that by just looking at our request, we can make pretty good guesses at what our return will look like. With REST, we have
no idea what the server will return, having to resort at a (hopefully _existent_) documentation somewhere.

Other real benefit is that we don't have lingering fields we don't use. If we don't care, say, to a potential slow field, e.g. our `friends` field,
the server won't have to make any efforts to give us that.

In the end, in one simple request, we could highlight 3 real benefits of GraphQL over standard REST APIs. Let's continue.

We can also query for individual characters:

```
{
  character(id: 1) {
    name
  }
}
```

```
{
  "data": {
    "character": {
      "name": "Rick Sanchez"
    }
  }
}
```

And I think that's enough of our implementation for now.

Now that we have an idea of what we kinda want, let's begin the implementation!

# Project structure 

We will now create our project structure. Let's set that up right now:

```
$ mkdir friends && cd friends
$ touch index.js
$ mkdir src && touch src/schema.js && touch src/services.js
```

Let's start an application using `yarn init`. Our project tree should look like this:
```
.
├── index.js
├── package.json
└── src
    ├── schemas.js
    └── services.js
```

Let's start from the beginning, defining a simple GraphQL endpoint on `index.js` so we can get these tires kicking.

So fire up `index.js` on your favourite editor and let's dive in to the code:
```
const koa = require('koa')
const logger = require('koa-logger')
const graphqlHTTP = require('koa-graphql')
const mount = require('koa-mount')
const { buildSchema } = require('graphql')

const app = koa()

// let's start with a simple schema
const schema = buildSchema(`
	type Query {
		hello: String!
	}
`)

// let's define our root query object
const root = {
	hello: () => {
		return 'hello world'
	}
}

// mount and use stuff
app.use(logger())
app.use(mount('/graphql', graphqlHTTP({
	schema: schema,
	rootValue: root,
	graphiql: true
})))

app.listen(8080)
console.log('GraphQL app listening on 127.0.0.1:8080/graphql')
```

If you point your browser to `http://127.0.0.1:8080/graphql`, you'll see a GraphiQL (notice the "i"?) interface. You can type:
```
{
  hello
}
```

And get back:
```
{
  "data": {
    "hello": "hello world"
  }
}
```

Congratulations! You have your first GraphQL server up and running in no time!

Let's get to the serious business now.

# Designing our schema

We can't live on top of `buildSchema` all the time. It's time give our schema a face! Let's start by slightly modifying our `index.js`:
```
const koa = require('koa')
const logger = require('koa-logger')
const graphqlHTTP = require('koa-graphql')
const mount = require('koa-mount')
const { RickAndMortySchema } = require('./src/schemas.js')

const app = koa()

app.use(logger())
app.use(mount('/graphql', graphqlHTTP({
	schema: schema,
	graphiql: true
})))

app.listen(8080)
console.log('GraphQL app listening on 127.0.0.1:8080/graphql')
```

We removed the reference to `buildSchema` imported from `graphql` and replaced it with a `RickAndMortySchema` that we'll soon code. We also removed
the reference to a root object, since we will define one in `RickAndMortySchema` in a second.

With that in our plate, let's fire up `src/schemas.js` and start coding:
```
const {
	GraphQLInt,
	GraphQLString,
	GraphQLList,
	GraphQLObjectType,
	GraphQLSchema,
} = require('graphql')

const characterStore = [
	{
		id: 1,
		name: 'Rick'
	},
	{
		id: 2,
		name: 'Morty'
	},
	{
		id: 3,
		name: 'Summer'
	},
	{
		id: 4,
		name: 'Beth'
	},
	{
		id: 5,
		name: 'Jerry'
	},
]

const friendsStore = [
	[characterStore[1], characterStore[2], characterStore[3]],
	[characterStore[0], characterStore[2], characterStore[3], characterStore[4]],
	[characterStore[0], characterStore[1], characterStore[3], characterStore[4]],
	[characterStore[0], characterStore[1], characterStore[2], characterStore[4]],
	[characterStore[1], characterStore[2], characterStore[3]],
]

const Character = new GraphQLObjectType({
	name: 'CharacterSchema',
	description: 'Defines a Rick and Morty character',
	fields: () => ({
		id: { type: GraphQLInt }, 
		name: { type: GraphQLString },
		friends: {
			type: new GraphQLList(Character),
			resolve: (character) => {
				return friendsStore[character.id - 1]
			}
		}
	})
})

const Query = new GraphQLObjectType({
	name: 'RootQuery',
	fields: () => ({
		characters: {
			type: new GraphQLList(Character),
			resolve: () => {
				return characterStore
			}
		},
		character: {
			type: Character,
			args: {
				id: { type: GraphQLInt }
			},
			resolve: (root, { id }) => {
				return characterStore.find((character) => {
					return character.id === id
				})
			}
		}
	})
})

const Schema = new GraphQLSchema({
	query: Query
})

module.exports = {
	RickAndMortySchema: Schema,
}
```

There's quite a few things happening here, so let's shuffle those 1 on 1:

- We are building our types using GraphQL primitives: `GraphQLInt`, `GraphQLString`, `GraphQLObjectType`, etc.
- We have a root object, where we define our main queries, which return the types we defined above
- We have hardcoded the relationships between our characters, in order to show nesting. We will favour PostgreSQL for this in a moment.
- Each type can have an `resolve` function. Take notice of the `friends` resolve function. That will not run **unless** we specify it in a GraphQL query.

Allright, that works. Play around a little bit. Try finding friends of friends by nesting, for instance:
```
{
  character(id: 1) {
    name
    friends {
      name
      friends {
        name
      }
    }
  }
}
```

That should give you:
```
{
  "data": {
    "character": {
      "name": "Rick",
      "friends": [
        {
          "name": "Morty",
          "friends": [
            {
              "name": "Rick"
            },
            {
              "name": "Summer"
            },
            {
              "name": "Beth"
            },
            {
              "name": "Jerry"
            }
          ]
        },
        {
          "name": "Summer",
          "friends": [
            {
              "name": "Rick"
            },
            {
              "name": "Morty"
            },
            {
              "name": "Beth"
            },
            {
              "name": "Jerry"
            }
          ]
        },
        {
          "name": "Beth",
          "friends": [
            {
              "name": "Rick"
            },
            {
              "name": "Morty"
            },
            {
              "name": "Summer"
            },
            {
              "name": "Jerry"
            }
          ]
        }
      ]
    }
  }
}
```

Awesome! We're almost there. We just need to replace our `characterStore` and `friendsStore` with our tables in PostgreSQL. Let's _burp_ get to it!

# Postgres, at last!

We need to have a connection with PostgreSQL. That means we need to know:

- Who is the user
- What is the password
- Where the address of the database is
- What database we want

All of that is specified in the *connection string*. Since we want things to be as easy as possible, let's take it from an environment variable.

We'll call it `RM_PGSQL_CONNSTRING` (`RM` being short of `Rick and Morty`).

After that, all we gotta do is make a pooled connection to the PostgreSQL database. Since we're using koa and, as a consequence, generator functions
and not callbacks, we're gonna go with `co-pg` instead of `node-postgres`. It's a simple generator API wrapper over `node-postgres`.

First, let's make some changes to the `src/schema.js` file:
```
const { CharacterService } = require('./services.js')
const {
	GraphQLInt,
	GraphQLString,
	GraphQLList,
	GraphQLObjectType,
	GraphQLSchema,
} = require('graphql')

const Character = new GraphQLObjectType({
	name: 'CharacterSchema',
	description: 'Defines a Rick and Morty character',
	fields: () => ({
		id: { type: GraphQLInt }, 
		name: { type: GraphQLString },
		friends: {
			type: new GraphQLList(Character),
			resolve: (character) => {
				return CharacterService.getFriends(character.id)
			}
		}
	})
})

const Query = new GraphQLObjectType({
	name: 'RootQuery',
	fields: () => ({
		characters: {
			type: new GraphQLList(Character),
			resolve: () => {
				return CharacterService.getAll()
			}
		},
		character: {
			type: Character,
			args: {
				id: { type: GraphQLInt }
			},
			resolve: (root, { id }) => {
				return CharacterService.getOne(id)
			}
		}
	})
})

const Schema = new GraphQLSchema({
	query: Query
})

module.exports = {
	RickAndMortySchema: Schema,
}
```

We removed any reference to `characterStore` and `friendsStore` and imported a module called `CharacterService`.
As you'd expect, `CharacterService` is the service that will reach the underlying data source (in our case, PostgreSQL)
and retrieve data for us.

Let's install `co`, `co-pg` and `pg`:
```
$ yarn add co co-pg pg
```

Fire up `src/services.js` and write the following code:
```
const co = require('co')
const pg = require('co-pg')(require('pg'))

const config = {
	connectionString: process.env.RM_PGSQL_CONNSTRING,
}

function* queryDB(query, args) {
	const [ client, done ] = yield pg.connectPromise(config.connectionString)
	const result = yield client.queryPromise(query, args)
	done()
	return result
}

const CharacterService = {
	getAll() {
		return co(function* () {
			const result = yield co(
			  queryDB('SELECT id, name FROM character')
			)
			const characters = result.rows.map((character) => {
				return {
					id: character.id,
					name: character.name,
				}
			})
			return characters
		})
	},

	getOne(characterID) {
		return co(function* () {
			const result = yield co(
			  queryDB('SELECT id, name FROM character WHERE id=$1', [characterID])
			)
			const { id, name } = result.rows[0]
			return {
				id,
				name,
			}
		})
	},

	getFriends(characterID) {
		return co(function* () {
			const result = yield co(
			  queryDB('SELECT f.id, f.name FROM character c JOIN friend ff ON c.id = ff.character_id JOIN character f ON f.id = ff.friend_id WHERE c.id = $1',
	  			  [characterID])
			)
			const friends = result.rows.map((friend) => {
				return {
					id: friend.id,
					name: friend.name,
				}
			})
			return friends
		})
	},
}

module.exports = {
	CharacterService,
}
```

In every call to `queryDB`, we pick a client from the pool, run our query, and give back the result. Each function defined in `CharacterService` treats it
accordingly before returning.
Notice that we actually return an `co` function, which works like a Promise. GraphQL knows that it needs to wait on returned promises, which
alleviates our code writing.

# At last!
We're looking good! After all, we have this code that _burp_ talks to a PostgreSQL database, and fetches all of the relationships of Rick and Morty characters.

But there is no way to insert new characters, nor add relationships to existing characters. Let's fix that by learning what GraphQL mutations are!

# GraphQL Mutations

GraphQL is really simple. It exposes 2 main concepts: Queries and Mutations. Queries return stuff, which is what we've been doing all along: reading data from somewhere
and dumping that somewhere else.
Mutations, on the other hand, describes data that should be _persisted_ somewhere. It is in no way different than a Query, or different than any other object we've dealt with
so far. Let's take a look at a GraphQL Mutation example:

```
# this mutation adds Mr. PoopyButthole to our Character database.
mutation addMrPoopyButthole {
  mrPoopyButthole: createCharacter(
    character: {
      name: "Mr. PoopyButthole"
    }
  ) {
    id
    name
  }
}
```
See how we specify what character we want to create (by using the `character` argument) and then we specify what *data* should be returned to us after this mutation runs.
So we can create a resource, and cherry-pick any relevant fields from it right away. That's powerful.

An possible return is:
```
{
  "data": {
    "mrPoopyButthole": {
      "id": 30,
      "name": "Mr. PoopyButthole"
    }
  }
}
```

The `mrPoopyButthole: createCharacter` bit is what GraphQL calls an _alias_. It's just a new name for our returned field on the JSON.

OK, we know what a mutation looks like. Let's implement one for creating users!
Fire up `src/schemas.js` and after the definition of the `Query` object, add this: 

```
const CharacterInput = new GraphQLInputObjectType({
	name: 'CharacterInput',
	fields: () => ({
		name: { type: new GraphQLNonNull(GraphQLString) },
	})
})

const Mutation = new GraphQLObjectType({
	name: 'MutationSchema',
	fields: () => ({
		createCharacter: {
			type: Character,
			description: 'Creates a new character',
			args: {
				character: { type: CharacterInput },
			},
			resolve: (root, { character }) {
				return CharacterService.createCharacter(character)
			}
		}
	})
})
```

Don't forget to actually import `GraphQLInputObjectType` and `GraphQLNonNull`:
```
const {
	GraphQLInt,
	GraphQLString,
	GraphQLList,
	GraphQLObjectType,
	GraphQLInputObjectType,
	GraphQLSchema,
} = require('graphql')
```

We created a `Mutation` that is one of our old friends, a `GraphQLObjectType`. It has a few fields, which we are used to, and a `resolve` function,
which we are also familiar with.

Now, add the `Mutation` object to the schema:

```
const Schema = new GraphQLSchema({
	query: Query,
	mutation: Mutation,
})
```

And all that is left to finish this Mutation is actually creating the `CharacterService.createCharacter` function. So let's do that.

Open `src/services.js` and after the `getFriends` function, add this:

```
createCharacter(character) {
	return co(function* () {
		const result = yield co(queryDB('INSERT INTO character (name) VALUES ($1) RETURNING id', [character.name]))
		const { id } = result.rows[0]
		return {
			id,
			name: character.name,
		}
	})
},
```

Now let's really add Mr. PoopyButthole to our Character database:
```
mutation {
  createCharacter(
    character: {
      name: "Mr. PoopyButthole"
    }
  ) {
    id
    name
  }
}
```

We should get as a result:
```
{
  "data": {
    "createCharacter": {
      "id": 6,
      "name": "Mr. PoopyButthole"
    }
  }
}
```

Congrats! You officially just created your first GraphQL API!

# Next steps

- Most of Rick and Morty characters are aliens from other dimensions. Add [enums](http://graphql.org/learn/schema/#enumeration-types) to classify
them as humans, aliens, animals, gazorpazorpians, etc.
- Improve searching with a `search` endpoint. Allow the user to filter based on the categorization created in the step above.
- Create a mutation called `addFriendship` that, given a `characterID` and a `friendID` arguments, it persists the friendship on the database.
- Go watch a Rick and Morty episode, even if you're already a fan. _Adult Swim, give me free stuff_
- Ask Dan Harmon on Twitter about Rick and Morty Season 3

You can reach me on [Twitter](https://twitter.com/panuto_) if you have any doubts.

See ya!

