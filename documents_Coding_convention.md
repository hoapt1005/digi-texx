## CQRS Architecture
### 1. Class Naming

#### Command

Command request format: **[Action]_[Object]_Command**

- CreateUserCommand
- UpdatePermissionOfUserCommand
- SaveRoleCommand

Command response format: **[Completed Action]_[Object]_Event**

- CreatedUserEvent
- UpdatedPermissionEvent

#### Query

Query request format: **[Action]__[Object]_[Params]_Query**

- FindUsersQuery
- FindUserByIdQuery
- FindUserByIdOfCompanyQuery
- GetUserByNameOrCodeQuery
- GetUserByEmailQuery
- LoadPermissionsByUserIdQuery

Query response format: **[Action]_[Object]_Result**

- FindUsersResult (for multiple items)
- GetUserItemResult (for single item) in result
- LoadPermissionsResult

#### External Request

External Service request format: **[Action]_[Object]_Message**

- SendEmailMessage
- SendSmsMessage
- RequestPaymentMessage

External Service response format: **[Action]_[Object]_Message**

- SentEmailMessage
- RequestedPaymentMessage

---

### Action naming (verbs)

| Name    | Description                                                                    |
| ------- | ------------------------------------------------------------------------------ |
| Find    | Can return null, empty                                                         |
| Create  | Create new item, return created item                                           |
| Delete  | Delete existing item, return deleted item                                      |
| Update  | Update existing item, return updated item                                      |
| ------- | ------------------------------------------------------------------------------ |
| Get     | Must return a value, throws exception if not found                             |
| Load    | is a synonym of Find or Get, but allow caching result and refreshable          |
| Insert  | is a synonym of Create, return inserted id of item                             |
| Save    | is a synonym of Create, return saved id of item, throws exception if not found |
| Remove  | is a synonym of Delete                                                         |
| Send    | call an external service without return value (void function)                  |
| Request | call an external service and wait for response                                 |

Object naming (nouns):

- Entity name in singular form for Command
  - User
  - Permission
  - Role
  - Company

- Entity name in singular form + "Item" or plural form for Query ( recommend plural form)
  - Users
  - Companies
  - Tasks
  - RoleItem (used for single item in result)
  - UserItem (used for single item in result)
  - TaskItem

Others:

- UsersQueryParams
- UsersRequestParams
- UsersConfig
- UserElement

---

Example declaring class

```ts
// wrong : too many params or missing suffix (Query)
// wrong2: too many class has similar feature
export class FindUsersByRealmAndUsernameAndCode {}
export class FindUsersByRealmAndUsernameAndActivated {}
export class FindUsersByRealmAndUsernameAndCodeQuery {}

// ok : consider 1 class with multiple function
export class FindUsersQuery {

  async execute(input:FindUserQueryParams):Promise<UserQueryResult>{

      const query = this.buildQuery(input); 
      const result = await this.repository.findMany(query);

      return plainToInstance(result,UserQueryResult.class);
  }

  buildQuery(input){
    //if...else anything to select function return query
    return this.findUsersByRealmAndUsername(input.realm,input.username)
  }

  findUsersByRealmAndUsernameAndCode(realm:string, username:string, code:string);
  findUsersByRealmAndUsername(realm:string, username:string, active = true);
}
```

```ts

// or create builder query params (recommended - query param may contains more metadata or updated value later without affect process query data)
export class FindUsersQuery {

  async execute(input:FindUserQueryParams):Promise<UserQueryResult>{

      const query = input.query; 
      const result = await this.repository.findMany(query);

      return plainToInstance(result,UserQueryResult.class);
  }
}

export class FindUserQueryParams{
  private mode;
  constructor({realm:string,username:name,code:string,active:boolean})

  get query(){
    switch(this.mode){
      case 1: return this.queryByRealmAndUsername();
      case ...
    }
  }

  queryByRealmAndUsername(){return {realm:this.realm,username:this.username}};
  queryByRealmAndUsernameOrCode(){return {realm:this.realm,username:this.username,code:this.code}};
}

// builder Result set
export class UserQueryResult{
  users:UserItem[]
}


```

### Function declaring

Simple function ( number of parameter < 4)

```ts
// wrong : too many params
findUsersByRealmAndUsernameAndCodeAndActivated(realm:string, username:string, code:string, active = true);

// accepted but not recommended (should reduce to 2 params)
findUsersByRealmAndUsernameAndCode(realm:string, username:string, code:string);
```

```ts
// wrong 
findUsersByRealmAndUsernameAndActivated(realm:string, username:string, active = true);

// ok : default value name does not need to be added to function name
findUsersByRealmAndUsername(realm:string, username:string, active = true);

// or combine adj to object
findActivatedUsersByRealmAndUsername(realm:string, username:string, active = true);
```

```ts
// ok but should create 2 function
findUsersByRealmAndUsernameAndCode(realm:string, username:string, code?:string);

// ok 
findUsersByRealmAndUsername(realm:string, username:string);
findUsersByRealmAndUsernameAndCode(realm:string, username:string, code:string); // this will require code is input, can throws exception if code is null
```

```ts

// wrong 
findUsers(realm:string, username:string, active = true);

// ok : naming with plural form should use Queryparams
findUsers(queryParams:UsersQueryParams);

```

### Pre-build infrastructure
Existing modules:
- EventNotificationModule: publish changed resources event to rabbitMq
- DatabaseModule: Provide singleton connection pool and definition of abstract entity & repository
- RedisModule: Access to external redis service
- ServiceRegistryModule: Use ServiceProxy provider to call remote function by name

All config of infra will be set at runtime env:
- MONGO_DB_URI : mongodb://localhost:27017
- QUEUE_URI  : amqp://localhost:5672
- REDIS_URI : redis://localhost:6379

**Example usage**:

Boostrap import:
```ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      cache: process.env.NODE_ENV !== 'development' || true,
      load: [], // additional configuration files which is not .env
    }),
    CqrsModule.forRoot(),
    DatabaseModule,
    EventNotificationModule.register({
      events: [ChangedResourceEvent, NotificationEvent]
    }),
    ServiceRegistryModule.registryClient({
      connectOptions: getConnectOptions(),
    }),
  ],
})
export class AppModule {}
```

DatabaseModule:
```ts

@Entity()
export class ExampleEntity { 
  @Column({ name: '_id', type: "String" })
  id?: EntityId | string;

  @Column('tenant_id')
  tenantId: string;

  @Column()
  name: string;

  @Column({ name: 'status', enumType: ActivityState })
  status: ActivityState;

  /**
  * Additional dynamic information data
  */
  @Column({ name: 'information', type: () => ProcessInformationVO })
  information: ProcessInformationVO;
}


@Repository({
  collectonNamePrefix: process.env.APP_NAME || '',
  collectionName: 'process',
  collectionNameSuffix: 'aggregation',
  entity: ExampleEntity
})
export class ProcessAggregationMongoRepository
  extends MongoCRUDRepository<ExampleEntity>{

  async findOne(query: { processId: string }): Promise<ExampleEntity> {
    const aggregationData: ExampleEntity = await super.findOne({
      process_id: query.processId,
    });

    return aggregationData;
  }
}

export declare class MongoCRUDRepository<T> extends MongoBaseRepository<T> {
    /**
    * Find data with searchDTO
    *  - Same action with findMany but input requires search models
    * @param search SearchDTO contain query/sort/paging
    * @param selectedColumns List of column returned
    * @returns Query result from db
    */
    search(search: QueryParams): Promise<T[]>;

    /**
     * Count data with searchDTO
     */
    count(counting: QueryParams): Promise<{
        count: number;
    }>;
    /**
     * Find data with searchDTO
     */
    findMany(search: QueryParams): Promise<T[]>;
    /**
   * Find data with default method of mongoose
   */
    find(query: object, projection?: {}, options?: {}): Promise<T[]>;
    /**
     * Find data with default method of mongoose
     * @param query
     * @param projection
     * @param options
     * @deprecated Renaming function Use method 'find' instead
     * @returns
     */
    findManyByQuery(query: object, projection?: {}, options?: {}): Promise<T[]>;
    /**
     * Count data with query object
     * @param query Query Object
     * @returns Count result
     */
    countByQuery(query: object): Promise<{
        count: number;
    }>;
    /**
     * Save input data
     * @param data
     * @returns Inserted record from db
     */
    create(data: T, options?: object): Promise<T>;
    /**
     * Save array as input data
     * @param data
     * @returns Inserted record from db
     */
    createMany(data: T[], options?: object): Promise<T>;
    /**
     * Find data by field and update value
     * @param id Id of data need to be updated
     * @param input Updating data
     * @returns Updated record from db
     */
    updateOneByField(field: string, value: any, input: any): Promise<T>;
    /**
     * Find data by many fields and update value
     * @param searchObj The object contain condition used to search such as : {"field1":"string","field2":"myString"}
     * @param input Updating data
     * @returns Updated record from db
     */
    updateOneByFields(searchObj: object, input: any): Promise<T>;
    /**
     * Find data by id and update value
     * @param id Id of data need to be updated
     * @param input Updating data
     * @param options Option when update
     * @returns Updated record from db
     */
    updateById(id: string, input: any, options?: object): Promise<T>;
    /**
     * Find data by id and delete
     * @param id Id of data need to be deleted
     * @returns Deleted record from db
     */
    deleteById(id: string, options?: object): Promise<T>;
    /**
     * Find data by primary key (id)
     * @param id Id of data
     * @param selectedColumns Returned selecting column
     * @returns Found data with selected columns if required
     */
    findById(id: string | Types.ObjectId, selectedColumns?: (keyof T)[] | string[]): Promise<T>;
    /**
     * Find data by multi primary key (id)
     * @param id Id of data
     * @param selectedColumns Returned selecting column
     * @returns
     */
    findByIds(ids: Array<Types.ObjectId> | Array<string>, selectedColumns?: (keyof T)[] | string[]): Promise<T[]>;
    /**
     * Find one result by 1 condition
     * @param field The field name used to search
     * @param value The value of field used to search
     * @param selectedColumns Returned selecting column
     * @returns Found record from db or null
     */
    findOneByField(field: string, value: any, selectedColumns?: (keyof T)[] | string[]): Promise<T>;
    /**
     * Query data by single where condition
     * @param field The field name used to search
     * @param value The value of field used to search
     * @param selectedColumns Returned selecting column
     * @returns Found records
     */
    findByField(field: string, value: any, selectedColumns?: (keyof T)[] | string[]): Promise<T[]>;
    /**
     * Find one result by 1 condition
     * @param searchObj The object contain condition used to search such as : {"field1":"string","field2":"myString"}
     * @param selectedColumns Returned selecting column
     * @returns Found record from db or null
     */
    findOneByFields(searchObj: object, selectedColumns?: (keyof T)[] | string[]): Promise<T>;
    /**
     * Find by id & push
     * @param id
     * @param data
     * @returns
     */
    findByIdAndPush(id: string | Types.ObjectId, data: any, option?: any): Promise<T>;
    /**
     * Find by id & pull
     * @param id
     * @param data
     * @returns
     */
    findByIdAndPull(id: string | Types.ObjectId, data: any, option?: any): Promise<T>;
    /**
    * Find by id by regex ( requires id is string)
    *  - Useful when search data relative data
    *  - example query: { _id: { $regex: new RegExp(`^${id}`) } }
    * @param id
    * @param query Additional query param
    * @returns
    */
    findByIdStartWith(id: string, query: any, selectedColumns?: (keyof T)[] | string[], option?: any): Promise<T[]>;
}
```

RedisModule
```ts
@Injectable()
export class RedisBaseService implements OnModuleInit {
    protected client: Redis = null;
    private readonly logger = new Logger(RedisBaseService.name);

    onModuleInit() {
        const connectConfig = process.env.REDIS_URI;
        if (!connectConfig) {
            this.logger.warn(`REDIS_URI is not defined`);
            return;
        }
        const appName = process.env.APP_NAME || process.env.npm_package_name
        this.init(connectConfig, appName);
    }

    private init(connectConfig: string, connectionName: string) {
        this.logger.log(`Connecting to Redis: ${connectConfig}`);
        this.client = new IORedis(connectConfig, { name: `${connectionName}_${Date.now()}` });
    }

    getClient(): Redis {
        if (!this.client) throw new Error('Redis not initialized');
        return this.client;
    }
}

@Injectable()
export class RedisPubSubService {
    constructor(
        private readonly redisBase: RedisBaseService,
        @Optional() private readonly serializer: ISerializer = new JsonSerializer()
    ) { }

    async publish(channel: string, data: any) {
        const payload = this.serializer.serialize(data);
        return this.redisBase.getClient().publish(channel, payload);
    }

    async subscribe(
        channel: string,
        callback: (message: any, pattern: string) => void
    ) {
        const client = this.redisBase.getClient();
        await client.psubscribe(channel);

        client.on('pmessage', (pattern, ch, message) => {
            const parsed = this.serializer.deserialize(message);
            callback(parsed, pattern);
        });
    }
}

@Module({
    providers: [
        RedisBaseService,
        RedisCrudService,
        RedisPubSubService,
        {
            provide: 'REDIS_SERIALIZER', // optional DI override
            useClass: JsonSerializer,
        },
    ],
    exports: [RedisCrudService, RedisPubSubService],
})
export class RedisModule { }
```

Base event : every CRUD module will be treated as a resource and publish changed resource event
```ts
export declare class ChangedUserEvent extends ChangedResourceEvent<ExampleEntity>{}

export declare class ChangedResourceEvent<T extends ResourceEntity = any> {
    action: ResourceAction;
    executor: RequestUserVO;
    resource: ResourceInfoVO;
    data: T | Partial<T>;
    time?: number;
    constructor(action: ResourceAction); // created/updated/deleted ...
}

```
Example json output

```json
{
  "time": 1762841553086,
  "action": "approved",
  "resource": { "name": "policies" },
  "data": {
    "policy_id": "572c6f54-540b-58ba-8505-6c8484179676",
    "policy_name": "access-app-administration",
    "users": ["u1"],
    "expired_date": "2025-03-17T04:10:41.638Z"
  },
  "executor": {"username":"approver1","realm":"default"},
  "id": "7c7f0a7d-51ec-4460-a983-4f23ff83658a"
}
```


ServiceRegistryModule:

```ts

@Injectable()
export class ServiceProxy {
    static context?: string = ServiceProxy.name;
    static logger = new Logger(ServiceProxy.name);

    /**
     * Send request to service and get err object to handle later instead of throw
     *  - Return object { err: }
     * Apply resilience pattern to manage request
     */
    static async request<Input = any, Output = any>(pattern: AllHandlers, data: Input): Promise<Output | { err: Error }>
    static async request<Input = any, Output = any>(pattern: string, data: Input): Promise<Output | { err: Error }>;

    @UseResilience(CircuitBreaker)
    static async request(pattern: AllHandlers | string, data: any) {
        const client = await ServiceDiscovery.instance.getClientByPattern(pattern as string);

        this.logFunction(pattern);

        return firstValueFrom(client.send(pattern, data)).catch(err => {
            this.logError(pattern, err);
            return {
                err: err
            }
        });
    }

    /**
     * Get connection and start request to microservice
     * Apply resilience pattern to manage request
     *  - Throw error if failed
     */
    static async send<Input = any, Output = any>(pattern: AllHandlers, data: Input): Promise<Output>
    static async send<Input = any, Output = any>(pattern: string, data: Input): Promise<Output>;

    @UseResilience(CircuitBreaker)
    static async send(pattern: AllHandlers | string, data: any) {
        const client = await ServiceDiscovery.instance.getClientByPattern(pattern);

        this.logFunction(pattern);

        return firstValueFrom(client.send(pattern, data)).catch(err => {
            this.logError(pattern, err);
            throw new Error(`Execute function failed - ${this.context}#${pattern}`);
        });
    }

    /**
     * Get connection and start request to microservice
     * Fire and forget event
     * Do not handle error
     */
    static async emit<Input = any, Output = any>(pattern: AllHandlers, data: Input): Promise<Output>
    static async emit<Input = any, Output = any>(pattern: string, data: Input): Promise<Output>;
    static async emit(pattern: AllHandlers | string, data: any) {
        const client = await ServiceDiscovery.instance.getClientByPattern(pattern);

        this.logFunction(pattern);

        return firstValueFrom(client.emit(pattern, data)).catch(err => {
            this.logError(pattern, err)
        });
    }
}

```
Example usage
```ts
  async test() {
    const dto = {
      users: ["user1"],
      resources: ["xxxx"]
    };

    await ServiceProxy.send(`shareUserOwnedResource`, dto)
  }
 

```