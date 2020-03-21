---
title: "Repository Pattern for MongDB in C#"
date:   2020-03-21 17:15:00 -0400
categories: mongodb-for-dotnet-developers
---

Even though MongoDB has been around for a while, most developers, espcially those who are new to the whole document storage thing, are still struggling with bringing in this relatively new 
concept into the world of layered architecture they have grown familiar with. One pattern in particular that we've grown to love is the repository pattern for accessing database objects. 
The repository works great for most, if not all, of the architectural patterns out there. So, it is only right that we learn how to use it for accessing MongoDB objects as well.

The goal of this blog post is not to educate us on MongoDB nor is it to make argument for the repository pattern, but rather it is to walk you through how to create a very simple, yet production worthy, repository class for effectively managing MongoDB documents. I will highlight all the required dependencies, create a DbContext and finally create the repository class.

If you will like to follow along with a working piece of code, here is the Github repository for this tutorial - [https://github.com/NexTekk/MongoDbTutorial](https://github.com/NexTekk/MongoDbTutorial).


### Project Setup and Dependencies
For the purpose of this tutorial, we'll be creating a repository class, `UserRepository.cs`, for running basic CRUD operations on the user collections of 
our `MongoDB` database. Follow the following steps to get started:
1. Install `MongoDB` server if you haven't done that already.
2. Get a tool to view your databases and interact with your server. `Robo 3T` is a good one that you can download for free [here](https://robomongo.org/download)
3. Create a new .NET solution, if you don't already have one.
4. Add a new `Class Library` project to the solution.
5. Add the following `NuGet` packages to your new `Class Library` project.
    1. `MongoDB.Driver`
    2. `MongoDB.Bson`
6. Add a `UserDocument` class to your project. This class is the schema definition for user documents that will be stored in the database users collection. 
Add all the needed properties to the class and feel free to use the class below as a guide:

{% highlight cs %}
public class UserDocument
{
    public Guid Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
{% endhighlight %}

### DbContext
Entity Framework (EF) introduced the concept of `DbContext` that has worked really well with the repository pattern. 
Simply put, `DbContext` is just a class that helps centralize data access.  
Our `DbContext` class will help establish a connection to the database and create a `MongoDatabase` instance that we can work with. 
See the `DbContext` code below for all that needs to go into the class. 
Note that this is only for the users collection. Add more collections to the class as needed.

'DbSettings` is a custom object we’re injecting into the DbContext to help abstract all the database settings.
You can initialize and set the properties however you want, but for this tutorial, those values are coming from appsettings.json in the console application - 
check out the [GitHub repo](https://github.com/NexTekk/MongoDbTutorial) for details.

{% highlight cs %}
using System;
using System.Security.Authentication;
using MongoDB.Bson;
using MongoDB.Driver;

namespace Tutorial.Data
{
    public class DbContext
    {
        private const string UsersCollectionName = “users”;

        private readonly Lazy<IMongoDatabase> _database;
        private readonly DbSettings _dbSettings;

        public DbContext(DbSettings dbSettings)
        {
            _dbSettings = dbSettings;
            _database = new Lazy<IMongoDatabase>(GetDatabase, true);
        }

        public IMongoCollection<UserDocument> Users => GetCollection<UserDocument>(UsersCollectionName);

        private IMongoCollection<T> GetCollection<T>(string collectionName)
        {
            // set this for GUIDs to work as expected for different programming languages
            var collectionSettings = new MongoCollectionSettings
            {
                GuidRepresentation = GuidRepresentation.Standard
            };

            return _database.Value.GetCollection<T>(collectionName, collectionSettings);
        }

        private IMongoDatabase GetDatabase()
        {
            var connectionUrl = new MongoUrl(_dbSettings.DbServer);
            var settings = MongoClientSettings.FromUrl(connectionUrl);

            settings.SslSettings = new SslSettings
            {
                EnabledSslProtocols = SslProtocols.Tls12,
            };

            var client = new MongoClient(settings);

            return client.GetDatabase(_dbSettings.DbName);
        }
    }
}
{% endhighlight %}

{% highlight cs %}
public class DbSettings
{
    public string DbServer { get; set; }
    public string DbName { get; set; }
}
{% endhighlight %}


### Repository Class
Next up is the UserRepository class. The code below shows both asynchronous and synchronous methods for the most common basic CRUD operations. 
Feel free to delete all the synchronous method if you'd rather have all asynchrnous methods. 

{% highlight cs %}
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using MongoDB.Driver;

namespace Tutorial.Data
{
    public class UserRepository : IUserRepository
    {
        private readonly DbContext _dbContext;

        public UserRepository(DbContext dbContext)
        {
            _dbContext = dbContext;
        }

        public Guid Add(UserDocument document)
        {
            _dbContext.Users.InsertOne(document);

            return document.Id;
        }

        public async Task<Guid> AddAsync(UserDocument document)
        {
            await _dbContext.Users.InsertOneAsync(document);

            return document.Id;
        }

        public UserDocument Get(Guid userId)
        {
            var cursor = _dbContext.Users.Find(x => x.Id == userId);

            return cursor.Single();
        }

        public async Task<UserDocument> GetAsync(Guid userId)
        {
            var cursor = await _dbContext.Users.FindAsync(x => x.Id == userId);

            return await cursor.SingleOrDefaultAsync();
        }

        public IList<UserDocument> GetAll()
        {
            var filter = Builders<UserDocument>.Filter.Empty;
            var cursor = _dbContext.Users.Find(filter);

            return cursor.ToList();
        }

        public async Task<IList<UserDocument>> GetAllAsync()
        {
            var filter = Builders<UserDocument>.Filter.Empty;
            var cursor = await _dbContext.Users.FindAsync(filter);

            return await cursor.ToListAsync();
        }

        public void Upsert(UserDocument document)
        {
            var options = new FindOneAndReplaceOptions<UserDocument, UserDocument>
            {
                IsUpsert = true
            };

            _dbContext.Users.FindOneAndReplace<UserDocument>(d => d.Id == document.Id, document, options);
        }

        public async Task UpsertAsync(UserDocument document)
        {
            var options = new FindOneAndReplaceOptions<UserDocument, UserDocument>
            {
                IsUpsert = true
            };

            await _dbContext.Users.FindOneAndReplaceAsync<UserDocument>(d => d.Id == document.Id, document, options);
        }

        public void Delete(Guid userId)
        {
            _dbContext.Users.DeleteOne(u => u.Id == userId);
        }

        public async Task DeleteAsync(Guid userId)
        {
            await _dbContext.Users.DeleteOneAsync(u => u.Id == userId);
        }
    }
}
{% endhighlight %}

### Conclusion
We have seen how easy it is to set up a repository class for MongoDB databases. On my next post, I'll go over how to create a generic repository
that takes care of most of the common database operations so you don't have to rewrite the code for every repository. That'll help keep our code `DRY`.

One last thing: don't forget to take quiz quiz below to test your knowledge of MongoDB.

{% include embed.html link="https://nexquiz.com/quizzes/1mq3l2/mongodb-quiz/embed?user=mongo%20tutorial%20user" %}