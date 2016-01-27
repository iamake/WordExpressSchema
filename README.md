# wordexpress-schema
This package provides a connection to a WordPress MYSQL database using Sequelize and provides a standard set of queries to use with GraphQL.

For a full example, check out the repo for [WordExpress.io](https://github.com/ramsaylanier/WordpressExpress), which was built using this package.

##Installation
```
npm install --save-dev wordexpress-schema
```

##WordExpressDatabase
The first part of WordExpress Schema is **WordExpressDatabase**. This class provides an easy connection to your WordPress database using a  Sequelize connection.

Below is the basic implementation:
```
import { WordExpressDatabase } from 'wordexpress-schema';
import { publicSettings, privateSettings } from '../settings/settings';

/*
  Example settings object:
  publicSettings: {
    uploads: "http://wordexpress.s3.amazonaws.com/",
    amazonS3: true
  },
  privateSettings: {
    wp_prefix: "wp_",
    database: {
      name: "wpexpress_dev",
      username: "root",
      password: "",
      host: "127.0.0.1"
    }
  }
*/

const { name, username, password, host } = privateSettings.database;
const { amazonS3, uploads } = publicSettings;

const connectionDetails = {
  name: name,
  username: username,
  password: password,
  host: host,
  amazonS3: amazonS3,
  uploadDirectory: uploads
}

const Database = new WordExpressDatabase(connectionDetails);
const ConnQueries = Database.queries;
export default ConnQueries;
```

###Connection Settings

In the above example, **WordExpressDatabase** is passed a connectionDetails object that contains some WordPress database settings. Name, username, password, and host are all self-explanatory. 

WordExpress will work with Amazon S3; passing in a truthy value for amazonS3 will alter the query for getting Post Thumbnail images. If you are using S3, you just need the include the base path to your S3 bucket (which means you should exclude the wp-content/uploads/ part of the path). If you are hosting images on your own server, include the full path to the uploads folder.

Lastly, you can modify the wordpress database prefix. Some people don't use the default "wp_" prefix for various reasons. If that's you, I got your back. 


###The Database Class

The Database class above contains the connectionDetails, the actual Sequelize connection, the database queries, and the database models. Really, all you need for GraphQL setup are the queries; however, if you'd like to extend queries with your own, the Database Models are exposed. 

####The Models
Here are the models and their definitions.  As you can see, for the Post model, not every column in the wp_posts table is included. I've included the most relevant columns; however because the Database class exposes the models, you can extend them to your liking.

```
Post: Conn.define(prefix + 'posts', {
  id: { type: Sequelize.INTEGER, primaryKey: true},
  post_author: { type: Sequelize.INTEGER },
  post_title: { type: Sequelize.STRING },
  post_content: { type: Sequelize.STRING },
  post_excerpt: { type: Sequelize.STRING },
  post_status:{ type: Sequelize.STRING },
  post_type:{ type: Sequelize.STRING },
  post_name:{ type: Sequelize.STRING},
  post_parent: { type: Sequelize.INTEGER},
  menu_order: { type: Sequelize.INTEGER}
}),
Postmeta: Conn.define(prefix + 'postmeta', {
  meta_id: { type: Sequelize.INTEGER, primaryKey: true, field: 'meta_id' },
  post_id: { type: Sequelize.INTEGER },
  meta_key: { type: Sequelize.STRING },
  meta_value: { type: Sequelize.INTEGER },
}),
Terms: Conn.define(prefix + 'terms', {
  term_id: { type: Sequelize.INTEGER, primaryKey: true },
  name: { type: Sequelize.STRING },
  slug: { type: Sequelize.STRING },
  term_group: { type: Sequelize.INTEGER },
}),
TermRelationships: Conn.define(prefix + 'term_relationships', {
  object_id: { type: Sequelize.INTEGER, primaryKey: true },
  term_taxonomy_id: { type: Sequelize.INTEGER },
  term_order: { type: Sequelize.INTEGER },
}),
TermTaxonomy: Conn.define(prefix + 'term_taxonomy', {
  term_taxonomy_id: { type: Sequelize.INTEGER, primaryKey: true },
  term_id: { type: Sequelize.INTEGER },
  taxonomy: { type: Sequelize.STRING },
  parent: { type: Sequelize.INTEGER },
  count: { type: Sequelize.INTEGER },
})
```

####The Queries

In the above example, ConnQueries will give you the following:

**getViewer()**

Necessary because GraphQL doesn't currently allow for nodes on the root query. It simply returns a User object with id:1 and name:Anonymous

**getPosts(args)**

args is an object; however, the only acceptable key in args is ```post_type```. This finds all published posts by post type. Returns a promised array.

**getPostById(postId)**

Accepts a post id and returns the corresponding post.

**getPostByName(name)**

Accepts a post name (AKA slug) and returns it. The post must be published.

**getPostThumbnail(postId)**

Accepts a post id and returns the thumbnail image. In the **connectionDetails** object above you'll notice that there is a setting for AmazonS3. Set to true if you want to use WordPress with AmazonS3.

**getPostmeta(postId, keys)**

Returns a posts' Postmeta. Keys is an array of valid meta_key values. See the below example for usage.

**getPostMetaById(metaId)**

Returns a single Postmeta by id. Probably not very useful right now. Instead, you'll want to use getPostmeta();

**getMenu(name)**

Returns a menu and all of its menu items where the name argument is the slug of your menu.

####Extending Queries

Extending the above example, it's possible to add your own queries (or even your own models). Here's an example:

```
...
const Database = new WordExpressDatabase(connectionDetails);
const Models = Database.models;
const ConnQueries = Database.queries;

const CustomPostsQuery = () => {
  return Models.Post.findAll({
    where: {
      post_type: 'my_custom_post_type',
      post_status: 'publish',
    }
  })
}

ConnQueries.getCustomPosts = CustomPostsQuery;
...
```

###Example Usage With GraphQL

This example requires working knowledge of GraphQL. If you aren't familiar, [start by reading the GraphQL documentation](http://graphql.org/docs/getting-started/).

If you'd like a full example, check out the repo for [WordExpress.io](https://github.com/ramsaylanier/WordpressExpress), which is built using this package.

```
const GraphQLUser = new GraphQLObjectType({
  name: "User",
  fields: {
    id: {type: new GraphQLNonNull(GraphQLID)} ,
    settings: {
      type: GraphQLSetting,
      resolve: ()=>{
        return publicSettings
      }
    },
    posts: {
      type: PostsConnection,
      args: {
        post_type: {
          type: GraphQLString,
          defaultValue: 'post'
        },
        ...connectionArgs
      },
      resolve(root, args) {
        return connectionFromPromisedArray( ConnQueries.getPosts(args), args );
      }
    },
    page: {
      type: GraphQLPost,
      args:{
        post_name:{ type: GraphQLString },
      },
      resolve(root, args){
        return ConnQueries.getPostByName(args.post_name);
      }
    },
    menus: {
      type: GraphQLMenu,
      args: {
        name: { type: GraphQLString }
      },
      resolve(root, args) {
        return ConnQueries.getMenu(args.name);
      }
    },
    postmeta: {
      type: PostmetaConnection,
      args: {
        post_id: {
          type: GraphQLInt
        },
        ...connectionArgs
      },
      resolve(root, args){
        return ConnQueries.getPostmeta(args.post_id)
      }
    }
  }
})

const GraphQLRoot = new GraphQLObjectType({
  name: 'Root',
  fields: {
    viewer: {
      type: GraphQLUser,
      resolve: () => {
        return ConnQueries.getViewer();
      }
    }
  }
});

const WordExpressSchema = new GraphQLSchema({
  query: GraphQLRoot
});
export default WordExpressSchema;

```

