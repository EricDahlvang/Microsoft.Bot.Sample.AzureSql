# Implementing a Custom State Client in Azure SQL

C# Prerequisites: 
* [Free Azure Trial account](https://azure.microsoft.com/)
* [Visual Studio 2017 or 2015 Community](https://www.visualstudio.com/downloads/) (update all extensions)
* [Bot Application Template](http://aka.ms/bf-bc-vstemplate) (in your Project Templates folder)

## Summary

The bot **State Service** was created to ensure that bots built using the **Microsoft Bot Framework** are restful and stateless.  User data, conversation data, and private conversation data are all managed by a **State Client**.  A default implementation is automatically provided by the Node.js and .NET SDKs.  These SDKs are open source, and can be found on GitHub: [https://github.com/Microsoft/BotBuilder](https://github.com/Microsoft/BotBuilder)  

The default **State Service** is not intended for production applications.  Bots should use one of the Azure Extensions available on GitHub: [https://github.com/Microsoft/BotBuilder-Azure](https://github.com/Microsoft/BotBuilder-Azure).  Or, impelement a custom **State Client** using whatever data storage platform you choose.  At the time of this writing, implementations can be found in:
* **MongoDB:** [Manacola/msbotframework-mongo-middlelayer](https://github.com/Manacola/msbotframework-mongo-middlelayer)
* **Redis:** [Microsoft-Bot-Framework-Use-Redis-to-store-conversation-state](https://ankitbko.github.io/2016/10/Microsoft-Bot-Framework-Use-Redis-to-store-conversation-state)
* **Table Storage** and **DocumentDb:** [https://github.com/Microsoft/BotBuilder-Azure](https://github.com/Microsoft/BotBuilder-Azure) (in both Node.js and .NET)  

This article demonstrates how to implement ```IBotDataStore<BotData> ``` using .NET and Azure Sql.

## More about the Default State Client

The **Bot Builder** Node.js and .NET sdks provide a default **State Client**, as mentioned previously.  The url for the **State Service** used by the default **State Client** is "https://state.botframework.com". 

This can be seen in the .NET sdk here: [https://github.com/Microsoft/BotBuilder/../StateAPI/StateClient.cs#L268](https://github.com/Microsoft/BotBuilder/blob/3a98a6b3d15962a57b5454bfb3f730d3588de3ef/CSharp/Library/Microsoft.Bot.Connector.Shared/StateAPI/StateClient.cs#L268)

```cs
private void Initialize()
{
    this.BotState = new BotState(this);
    this.BaseUri = new Uri("https://state.botframework.com");

    [...]
}
```

In the Node.js sdk here:
[https://github.com/Microsoft/BotBuilder/../ChatConnector.ts#L94](https://github.com/Microsoft/BotBuilder/blob/master/Node/core/src/bots/ChatConnector.ts#L94)

```js
constructor(protected settings: IChatConnectorSettings = {}) {
        if (!this.settings.endpoint) {
            this.settings.endpoint = {
                [...],
                stateEndpoint: this.settings.stateEndpoint || 'https://state.botframework.com'
            }
        }
        ...
    }
```

This default implementation can be overridden by supplying a custom ```IBotDataStore<BotData> ``` implementation in .NET or ```IStorageClient``` in Node.js. 

```cs
public interface IBotDataStore<T>
{
    Task<bool> FlushAsync(IAddress key, CancellationToken cancellationToken);
    
    Task<T> LoadAsync(IAddress key, BotStoreType botStoreType, CancellationToken cancellationToken);
    
    Task SaveAsync(IAddress key, BotStoreType botStoreType, T data, CancellationToken cancellationToken);
}
```

```js
export interface IStorageClient {
    initialize(callback: (error: any) => void): void;
    insertOrReplace(partitionKey: string, rowKey: string, entity: any, isCompressed: boolean, callback: (error: any, etag: any, response: IHttpResponse) => void): void;
    retrieve(partitionKey: string, rowKey: string, callback: (error: any, entity: IBotEntity, response: IHttpResponse) => void): void;
}
```

## Why use a Custom State Client

Here are more reasons to use custom state storage:
*	higher state api throughput (more control over performance)
*	low-latency for geo-distrubtion 
*	control over where the data is stored
*	store more than 32kb 

## IBotDataStore With Azure Sql

In this example we use the **Entity Framework** to map BotData objects to SqlBotDataEntity objects saved in an Sql Server table.  Below is the ```IBotDataStore<BotData>``` implementation.  You'll notice that the constructor is expecting a connection string name.  This should be the name of a standard **System.Data.SqlClient** connection string in the web.config of the bot project.

```xml
<connectionStrings>
    <add name="BotDataContextConnectionString" 
        providerName="System.Data.SqlClient" 
        connectionString="Server=tcp:[AzureDatabaseServerName].database.windows.net,1433;
                        Initial Catalog=[SqlDatabaseName];
                        Persist Security Info=False;
                        User ID=[SqlDatabaseUserName];
                        Password=[SqlDatabasePassword];
                        MultipleActiveResultSets=False;
                        Encrypt=True;
                        TrustServerCertificate=False;
                        Connection Timeout=30;" />
</connectionStrings>
```

The **SqlBotDataStore** class is the ```IBotDataStore<BotData>``` implementation responsible for loading and persisting the **SqlBotDataEntity** objects.  It uses an instance of the **SqlBotDataContext**, which is an **Entity Framework** DbContext (also shown below).

```cs
public class SqlBotDataStore : IBotDataStore<BotData>
{
    string _connectionStringName { get; set; }
    public SqlBotDataStore(string connectionStringName)
    {
        _connectionStringName = connectionStringName;
    }

    async Task<BotData> IBotDataStore<BotData>.LoadAsync(IAddress key, BotStoreType botStoreType, CancellationToken cancellationToken)
    {
        using (var context = new SqlBotDataContext(_connectionStringName))
        {
            try
            {
                SqlBotDataEntity entity = SqlBotDataEntity.GetSqlBotDataEntity(key, botStoreType, context);

                if (entity == null)
                    return new BotData(eTag: String.Empty, data: null);
                
                return new BotData(entity.ETag, entity.GetData());
            }               
            catch (Exception ex)
            {
                throw new HttpException((int)HttpStatusCode.InternalServerError, ex.Message);
            }
        }
    }

    async Task IBotDataStore<BotData>.SaveAsync(IAddress key, BotStoreType botStoreType, BotData botData, CancellationToken cancellationToken)
    {
        SqlBotDataEntity entity = new SqlBotDataEntity(botStoreType, key.BotId, key.ChannelId, key.ConversationId, key.UserId, botData.Data)
        {
            ETag = botData.ETag,
            ServiceUrl = key.ServiceUrl
        };

        using (var context = new SqlBotDataContext(_connectionStringName))
        {
            try
            {
                if (String.IsNullOrEmpty(entity.ETag))
                {
                    context.BotData.Add(entity);
                }
                else if (entity.ETag == "*")
                {
                    var foundData = SqlBotDataEntity.GetSqlBotDataEntity(key, botStoreType, context);
                    if (botData.Data != null)
                    {
                        if (foundData == null)
                            context.BotData.Add(entity);
                        else
                        {
                            foundData.Data = entity.Data;
                            foundData.ServiceUrl = entity.ServiceUrl;
                        }
                    }
                    else
                    {
                        if (foundData != null)
                            context.BotData.Remove(foundData);
                    }
                }
                else
                {
                    var foundData = SqlBotDataEntity.GetSqlBotDataEntity(key, botStoreType, context);
                    if (botData.Data != null)
                    {
                        if (foundData == null)
                            context.BotData.Add(entity);
                        else
                        {
                            foundData.Data = entity.Data;
                            foundData.ServiceUrl = entity.ServiceUrl;
                            foundData.ETag = entity.ETag;
                        }
                    }
                    else
                    {
                        if (foundData != null)
                            context.BotData.Remove(foundData);
                    }
                }
                context.SaveChanges();
            }
            catch (System.Data.SqlClient.SqlException err)
            {
                throw new HttpException((int)HttpStatusCode.InternalServerError, err.Message);
            }
        }
    }

    Task<bool> IBotDataStore<BotData>.FlushAsync(IAddress key, CancellationToken cancellationToken)
    {
        // Since our implementation saves everything in SaveAsync,
        // there is nothing to do here
        return Task.FromResult(true);
    }
}
```

The **SqlBotDataContext** inherits from DbContext and has the BotData **DbSet** of **SqlBotDataEntity** objects.

```cs
public class SqlBotDataContext : DbContext
{
    public SqlBotDataContext()
        : this("BotDataContextConnectionString")
    {
    }
    public SqlBotDataContext(string connectionStringName)
        : base(connectionStringName)
    {
    }
    public DbSet<SqlBotDataEntity> BotData { get; set; }
}
```

The **SqlBotDataEntity** class inherits from ```IAddress``` and contains the fields stored in the database.  It also has methods for serializing and deserializing the data field.

```cs
public class SqlBotDataEntity : IAddress
{
    private static readonly JsonSerializerSettings serializationSettings = new JsonSerializerSettings()
                                {
                                    Formatting = Formatting.None,
                                    NullValueHandling = NullValueHandling.Ignore
                                };
    internal SqlBotDataEntity() { }
    internal SqlBotDataEntity(BotStoreType botStoreType, string botId, string channelId, string conversationId, string userId, object data)
    {
        this.BotStoreType = botStoreType;
        this.BotId = botId;
        this.ChannelId = channelId;
        this.ConversationId = conversationId;
        this.UserId = userId;
        this.Data = Serialize(data);
    }


    #region Fields

    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    [Index("idxStoreChannelUser", 1)]
    [Index("idxStoreChannelConversation", 1)]
    [Index("idxStoreChannelConversationUser", 1)]
    public BotStoreType BotStoreType { get; set; }
    public string BotId { get; set; }
    [Index("idxStoreChannelConversation", 2)]
    [Index("idxStoreChannelUser", 2)]
    [Index("idxStoreChannelConversationUser", 2)]
    [MaxLength(200)]
    public string ChannelId { get; set; }
    [Index("idxStoreChannelConversation", 3)]
    [Index("idxStoreChannelConversationUser", 3)]
    [MaxLength(200)]
    public string ConversationId { get; set; }
    [Index("idxStoreChannelUser", 3)]
    [Index("idxStoreChannelConversationUser", 4)]
    [MaxLength(200)]
    public string UserId { get; set; }
    public byte[] Data { get; set; }        
    public string ETag { get; set; }
    public string ServiceUrl { get; set; }
    [Required, DatabaseGenerated(DatabaseGeneratedOption.Computed)]
    public DateTimeOffset Timestamp { get; set; }

    #endregion Fields

    #region Methods

    private static byte[] Serialize(object data)
    {
        using (var cmpStream = new MemoryStream())
        using (var stream = new GZipStream(cmpStream, CompressionMode.Compress))
        using (var streamWriter = new StreamWriter(stream))
        {
            var serializedJSon = JsonConvert.SerializeObject(data, serializationSettings);
            streamWriter.Write(serializedJSon);
            streamWriter.Close();
            stream.Close();
            return cmpStream.ToArray();
        }
    }

    private static object Deserialize(byte[] bytes)
    {
        using (var stream = new MemoryStream(bytes))
        using (var gz = new GZipStream(stream, CompressionMode.Decompress))
        using (var streamReader = new StreamReader(gz))
        {
            return JsonConvert.DeserializeObject(streamReader.ReadToEnd());
        }
    }

    internal ObjectT GetData<ObjectT>()
    {
        return ((JObject)Deserialize(this.Data)).ToObject<ObjectT>();
    }

    internal object GetData()
    {
        return Deserialize(this.Data);
    }
    internal static SqlBotDataEntity GetSqlBotDataEntity(IAddress key, BotStoreType botStoreType, SqlBotDataContext context)
    {
        SqlBotDataEntity entity = null;
        var query = context.BotData.OrderByDescending(d => d.Timestamp);
        switch (botStoreType)
        {
            case BotStoreType.BotConversationData:
                entity = query.FirstOrDefault(d => d.BotStoreType == botStoreType
                                                && d.ChannelId == key.ChannelId
                                                && d.ConversationId == key.ConversationId);
                break;
            case BotStoreType.BotUserData:
                entity = query.FirstOrDefault(d => d.BotStoreType == botStoreType
                                                && d.ChannelId == key.ChannelId
                                                && d.UserId == key.UserId);
                break;
            case BotStoreType.BotPrivateConversationData:
                entity = query.FirstOrDefault(d => d.BotStoreType == botStoreType
                                                && d.ChannelId == key.ChannelId
                                                && d.ConversationId == key.ConversationId
                                                && d.UserId == key.UserId);
                break;
            default:
                throw new ArgumentException("Unsupported bot store type!");
        }

        return entity;
    }
    #endregion
}
```

The **Bot Builder** .NET sdk uses **Autofac** for dependency injection.  In order to override the default ```IBotDataStore```, and use our custom implementation, modify the project's ```Global.asax.cs``` to match the following:

```cs
protected void Application_Start()
{
    GlobalConfiguration.Configure(WebApiConfig.Register);

    var builder = new ContainerBuilder();
    
    builder.RegisterModule(new DialogModule());
    
    var store = new SqlBotDataStore("BotDataContextConnectionString");

    builder.Register(c => new CachingBotDataStore(store, CachingBotDataStoreConsistencyPolicy.LastWriteWins))
        .As<IBotDataStore<BotData>>()
        .AsSelf()
        .InstancePerLifetimeScope();

    builder.Update(Conversation.Container);            
}
```

Once these classes are added to your project, the **Entity Framework** nuget package has been referenced, and the **Connection String** added to the web.config, open the **Package Manager Console** window and enter the following two commands:

```cs
PM> enable-migrations
PM> add-migration "Initial Setup"
```

This will generate the ```Configuration.cs``` and an ```InitialSetup``` DbMigration class that contains the following:

```cs
public partial class InitialSetup : DbMigration
{
    public override void Up()
    {
        CreateTable(
            "dbo.SqlBotDataEntities",
            c => new
                {
                    Id = c.Int(nullable: false, identity: true),
                    BotStoreType = c.Int(nullable: false),
                    BotId = c.String(),
                    ChannelId = c.String(maxLength: 200),
                    ConversationId = c.String(maxLength: 200),
                    UserId = c.String(maxLength: 200),
                    Data = c.Binary(),
                    ETag = c.String(),
                    ServiceUrl = c.String(),
                    Timestamp = c.DateTimeOffset(nullable: false, precision: 7),
                })
            .PrimaryKey(t => t.Id)
            .Index(t => new { t.BotStoreType, t.ChannelId, t.ConversationId }, name: "idxStoreChannelConversation")
            .Index(t => new { t.BotStoreType, t.ChannelId, t.ConversationId, t.UserId }, name: "idxStoreChannelConversationUser")
            .Index(t => new { t.BotStoreType, t.ChannelId, t.UserId }, name: "idxStoreChannelUser");
        
    }
    
    public override void Down()
    {
        DropIndex("dbo.SqlBotDataEntities", "idxStoreChannelUser");
        DropIndex("dbo.SqlBotDataEntities", "idxStoreChannelConversationUser");
        DropIndex("dbo.SqlBotDataEntities", "idxStoreChannelConversation");
        DropTable("dbo.SqlBotDataEntities");
    }
}
```

Note the Timestamp field.  We want it to have a default value of the current utc date.  So, modify that line like this:

```cs
Timestamp = c.DateTimeOffset(nullable: false, precision: 7, defaultValueSql: "GETUTCDATE()"),
```

Finally, execute ```update-database``` at the Nuget Package Manager Console:

```cs
PM> update-database
```

This will execute the migration against the database, creating the **SqlBotDataEntities** table.  Once the table is created, you can run the project.  However, nothing specific will be stored in the three bot data bags (PrivateConversationData, ConversationData and UserData) because we haven't added code to the bot that uses the data bags.

Modify the ```RootDialog``` so the MessageReceivedAsync method is like this:

```cs
private async Task MessageReceivedAsync(IDialogContext context, IAwaitable<object> result)
{
    //retrieve the info objects, incrementing the count for each
    var privateData = context.PrivateConversationData;
    var conversationData = context.ConversationData;
    var userData = context.UserData;
    var privateConversationInfo = IncrementInfoCount(privateData, BotStoreType.BotPrivateConversationData.ToString());    
    var conversationInfo = IncrementInfoCount(conversationData, BotStoreType.BotConversationData.ToString());
    var userInfo = IncrementInfoCount(userData, BotStoreType.BotUserData.ToString());

    var activity = await result as Activity;

    // calculate something for us to return
    int length = (activity.Text ?? string.Empty).Length;

    // return our reply to the user, showing the three counts
    await context.PostAsync($"You sent {activity.Text} which was {length} characters. \n\nPrivate Conversation message count: {privateConversationInfo.Count}. \n\nConversation message count: {conversationInfo.Count}.\n\nUser message count: {userInfo.Count}.");

    //persist the three info objects to the store
    privateData.SetValue(BotStoreType.BotPrivateConversationData.ToString(), privateConversationInfo);
    conversationData.SetValue(BotStoreType.BotConversationData.ToString(), conversationInfo);
    userData.SetValue(BotStoreType.BotUserData.ToString(), userInfo);

    context.Wait(MessageReceivedAsync);
}
```

You'll also need the following simple class and method:

```cs
public class BotDataInfo
{
    public int Count { get; set; }
}

private BotDataInfo IncrementInfoCount(IBotDataBag botdata, string key)
{
    BotDataInfo info = null;
    if (botdata.ContainsKey(key))
    {
        info = botdata.GetValue<BotDataInfo>(key);
        info.Count++;
    }
    else
        info = new BotDataInfo() { Count = 1 };

    return info;
}
```

## The Table

You should now be able to run the bot, connect to it from the emulator, and see something similar to the following:

![Bot](/images/Bot.PNG)

If you look at the database, you'll see data:

![Stored Entities](/images/SqlBotDataEntities.PNG)

## References
*	[docs.microsoft.com/.../bot-framework-rest-state](https://docs.microsoft.com/en-us/bot-framework/rest-api/bot-framework-rest-state)
*	[docs.microsoft.com/.../bot-builder-dotnet-state](https://docs.microsoft.com/en-us/bot-framework/dotnet/bot-builder-dotnet-state)
*	[docs.microsoft.com/.../bot-builder-nodejs-state](https://docs.microsoft.com/en-us/bot-framework/nodejs/bot-builder-nodejs-state)

