All ceramic projects use ES6 Generators instead of callbacks for async. So you'd have to run wth node with the --harmony flag or use regenerator to transpile code to ES5.

### Installation
{% highlight sh %}
npm install ceramic
{% endhighlight %}

### Usage
{% highlight javascript %}

var Author, BlogPost;
var authorSchema, postSchema;

var Ceramic = require("ceramic");

//Constructor function for the Author Entity
Author = function(params) {
    if (params) {
        for(var key in params) {
            this[key] = params[key];
        }
    }
};

//authorSchema has ctor set to Author
//if ctor is omitted, a plain JS object is created
authorSchema = {
    name: 'author',
    ctor: Author,
    schema: {
        type: 'object',
        properties: {
            name: { type: 'string' },
            location: { type: 'string' },
            age: { type: 'number' }
        },
        required: ['name', 'location']
    }
};

//Constructor function for the BlogPost Entity
BlogPost = function(params) {
    if (params) {
        for(var key in params) {
            this[key] = params[key];
        }
    }
};

//postSchema has ctor set to BlogPost
//if ctor is omitted, a plain JS object is created
postSchema = {
    name: 'post',
    ctor: BlogPost,
    schema: {
        type: 'object',
        properties: {
            title: { type: 'string' },
            content: { type: 'string' },
            published: { type: 'string' },
            author: { $ref: 'author' }
        },
        required: ['title', 'content', 'author']
    }
};

var ceramic = new Ceramic();
var schemaCache = yield* ceramic.init([authorSchema, postSchema]);
var blogPostJSON = {
    title: "Busy Being Born",
    content: "The days keep dragging on, Those rats keep pushing on, ..",
    published: "yes",
    author: {
        name: "Middle Class Rut",
        location: "USA",
    }
};

var blogPost = yield* ceramic.constructEntity(blogPostJSON, postSchema);

console.log(blogPost instanceof BlogPost); //true
console.log(blogPost.author instanceof Author); //true

{% endhighlight %}



### Virtual Schemas

The beauty of document-based databases is that document structure is not necessarily rigid. We might use a single collection (or table) to store objects with differing schema, some of which may even be user-defined. For example, the Records collection might store objects of type text-posts, video-posts or short-stories.

Ceramic supports this through a concept called virtual-schemas. Virtual-schemas extend a base-schema with additional properties. To identify which virtual-schema to use, the base-schema must define a discriminator.

{% highlight javascript %}

// Note how the discriminator is defined.
// It uses the type property to find the virtual-schema
var songSchema = {
    name: 'ticket',
    discriminator: function*(obj, ceramic) {
        return yield* ceramic.getEntitySchema(obj.type);
    },
    schema: {
        type: 'object',
        properties: {
            title: { type: 'string' },
            artist: { type: 'string' },
            price: { type: 'number' },
            type: { type: 'string' }
        },
        required: ['title', 'artist']
    }
};

var mp3Schema = {
    name: "mp3",
    schema: {
        properties: {
            bitrate: { type: 'number' }
        },
        required: ['bitrate']
    }
};
var youtubeVideoSchema = {
    name: "youtube",
    schema: {
        properties: {
            url: { type: 'string' },
            highDef: { type: 'boolean' }
        },
        required: ['url', 'highDef']
    }
};

var ceramic = new Ceramic();

// ceramic.init(schemas, virtualSchemas)
// virtualSchemas is an array of virtualSchema definitions, as follows.
var schemaCache = yield* ceramic.init(
    [songSchema], //schemas
    [
        {
            entitySchemas: [mp3Schema, youtubeVideoSchema],
            baseEntitySchema: songSchema
        }
    ] //virtual-schemas
);

var mp3JSON = {
    "type": "mp3",
    "title": "Busy Being Born",
    "artist": "Middle Class Rut",
    "bitrate": 320
};

var mp3 = yield* ceramic.constructEntity(mp3JSON, songSchema, { validate: true });

{% endhighlight %}



### Validation
Ceramic does validation with JaySchema. For more information, see [https://github.com/natesilva/jayschema](https://github.com/natesilva/jayschema).

JaySchema supports extensive validation, including:

- Type mismatch (eg: schema says age is a number, but got string)
- Required fields
- Array max length and min length

{% highlight javascript %}

var ceramic = new Ceramic();
var typeCache = yield* ceramic.init([authorSchema, postSchema]);
var blogPost = new BlogPost({
    title: "Busy Being Born",
    content: "The days keep dragging on, Those rats keep pushing ..",
    published: "yes",
    author: new Author({
        name: "jeswin",
        location: "bangalore"
    })
});
var errors = yield* ceramic.validate(blogPost, postSchema);

console.log(errors.length); //Prints 0

{% endhighlight %}


### Dynamic Schemas (Advanced)

The simplest way to use Ceramic is to load all the schemas while calling ceramic.init(). But some apps might have dozens (or hundreds) of user-defined schemas stored on the disk that need not be preloaded during init() for performance reasons. Ceramic supports dynamic schemas to address this use case.

If Ceramic comes across a schema (or schema name) that it isn't aware, it tries to call a custom dynamic schema loader function. The schema loader function must be provided in Ceramic's constructor, and must return a valid schema when invoked.

{% highlight javascript %}

var songSchema = {
    name: 'song',
    schema: {
        type: 'object',
        properties: {
            title: { type: 'string' },
            artist: { type: 'string' },
            price: { type: 'number' },
            torrent: { $ref: 'torrent' }
        },
        required: ['title', 'artist']
    }
};

var torrentSchema = {
    name: "torrent",
    schema: {
        properties: {
            fileName: { type: 'string' },
            seeds: { type: 'number' },
            leeches: { type: 'number' }
        },
        required: ['fileName', 'seeds', 'leeches']
    }
};

var dynamicLoader = function*(name, dynamicResolutionContext) {
    switch(name) {
        case "torrent":
            return yield* ceramic.completeEntitySchema(torrentSchema);
    }
};

var ceramic = new Ceramic({
    fn: { getDynamicEntitySchema: dynamicLoader }
});

//song schema references torrent schema, but is not provided during init.
var schemaCache = yield* ceramic.init([songSchema]);

var songJSON = {
    title: "Busy Being Born",
    artist: "Middle Class Rut",
    price: 10,
    torrent: {
        fileName: "busy-being-born.mp3",
        seeds: 1000,
        leeches: 1100
    }
};

var mp3 = yield* ceramic.constructEntity(songJSON, songSchema, { validate: true });

console.log(mp3);

{% endhighlight %}


### Must Construct? (Advanced)

Ceramic will not try to construct an object against a schema if ceramic.mustConstruct() returns false. Instead the plain JS object is returned as is. You can override the default mustConstruct() behavior (which is true), by providing a custom mustConstruct while creating a Ceramic instance.

{% highlight javascript %}

var ceramic = new Ceramic({
    fn: {
        mustConstruct: function() {
            //...
        }
    }
});

{% endhighlight %}

### More

- Pull requests welcome. Just follow the same coding conventions.
- [Issues](https://github.com/jeswin/ceramic/issues)
- To run tests, 'mocha --harmony'
