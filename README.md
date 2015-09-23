# Migrating from Mongoid 4.x.x to Mongoid 5.x.x in Ruby on Rails app

## Config changes in `mongoid.yml`
All configuration options can be found on the [MongoDB Docs](http://docs.mongodb.org/ecosystem/tutorial/ruby-mongoid-tutorial/#anatomy-of-a-mongoid-config)

### Rename `sessions` to `clients`
Was
```yml
production:
  sessions:
    default:
```
    
Became
```yml
production:
  clients:
    default:
```

### Configure authentification with username and password
Was
```yml
production:
  clients:
    default:
      username: www
      password: pa$$worD
```

Became one level deeper, change option name for username and add auth method
```yml
production:
  clients:
    default:
      options:
        user: www
        password: pa$$worD
        auth_mech: :mongodb_cr  # this option for MongoDB 2.8 or greater with Auth Schema 3
        # auth_mech: :scram # for MongoDB 3.x with (Auth Schema 5)
        # auth_mech: :plain # for MongoDB 2.4, 2.6 (Auth Schema 3)
        # auth_source: admin # DB name for authorization, this is default value
```

If you see such error `no such cmd: saslStart (59)` with `auth_mech: :plain`, so you should use `auth_mech: :mongodb_cr`

### Change `pool_size` option name
Was
```yml
options:
  pool_size: 96
```

Became
```yml
options:
  max_pool_size: 96
```

### Change `read` option
Was
```yml
options:
  read: primary
```

Became
```yml
options:
  read: 
    mode: primary
```

If you see such error `NoMethodError - undefined method 'each_pair' for "primary":String` in `bson-3.2.4/lib/bson/document.rb:82` this is solution for you.


## Change LOGGER level or WILL DIE
I believe that this time bomb will be replaced in further versions. My current version is `gem mongo v2.1.0`, `gem mongoid v5.0.0`.

Logging level is DEBUG by default, which writes each query to app logfile. So you may get several GB per hour. 
When running out free disk space your application will die.     

Available logging levels: DEBUG, INFO, WARN, ERROR, FATAL.

Just changing logging options in the environment files. For example, add to `config/environments/production.rb`:
```ruby
Mongo::Logger.logger.level = Logger::WARN
```


## Model changes

### Rename `session` to `client`
Was
```ruby
class Artist
  include Mongoid::Document
  store_in session: 'stat'
```

Became
```ruby
class Artist
  include Mongoid::Document
  store_in client: 'stat'
```

If you use 3rd party gems with mongoid 4 models, which can not be modified, you may use 
this patch. Just create following file `config/initializers/mongoid5_patch_store_in.rb`:

```ruby
Mongoid::Clients::StorageOptions::ClassMethods.module_eval do
  def store_in(options)
    options[:client] = options.delete(:session) if options && options[:session]

    Mongoid::Clients::Validators::Storage.validate(self, options)
    storage_options.merge!(options)
  end
end
```

### Custom Fields types with `mongoize` and `demongoize`
Yay, nothing to do. All works perfectly.


## Query Changes

### select() -> projection()
Was
```ruby
collection.find.select(_id: 1, price: 1)
```
Became
```ruby
collection.find.projection(_id: 1, price: 1)
```

### upsert() -> update_one()
Was
```ruby
collection.where(code: 123).upsert(:$inc => inc)
```

Became
```ruby
collection.where(code: 123).update_one({:$inc => inc}, {:upsert => true})
```


### aggregate({}, {}, {})  -> aggregate([{}, {}, {}])
Was
```ruby
collection.aggregate(
      {'$match' => ... },
      {'$project' => ... }
)
```

Became
```ruby
collection.aggregate(
   [
      {'$match' => ... },
      {'$project' => ... }
   ]
)
```      

### Creating new collection
Was
```ruby
Moped::Collection.new(database, collection_name)
```

Became
```ruby
client = Mongo::Client.new([ '127.0.0.1:27017' ], :database => 'music')
Collection = client[:artists]
# or
Collection = SomeExistingModel.collection.client[:artists]
```

## Disclaimer
All the changes are made at your own risk.

This is not a complete list of all changes. 
If you want add some additional information, please make a pull request or open an issue.

Community will be very happy if get clearly instruction for migration.
Thank you.
