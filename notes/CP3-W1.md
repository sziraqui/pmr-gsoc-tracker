# Coding Period 3 - Week 1
Coding period 3 begins but I have a lot of segfaults that crash my program often. Major work this week is to make a database of face embeddings from celebrity dataset (LFW).
## Solving segfaults
The program does not segfault always. 
This was the biggest hint but I did not pay attention to it. 
I had to find out which part of program does not always execute. I ran tests covering all branches in MatImage template class.
None of my log messages appeared when program crashed. 
What was I missing? The Destructor! Destructors are not always called. 

Something in the destructor could be causing segfaults.
```
Nodoface::MatImage<numericType>::~MatImage() {
    delete[] this->mat->data; // <- manually releasing memory
    delete this->mat; // <- this will cause Mat's destructor to be called
}
```
MatImage destructor is called by Node runtime whenever GC event is triggered. 
But Node V8 engine does not guarantee when it will trigger GC. It may not even trigger GC at all.
So whenever destructor is called code execution, I think, will follow this order:
- Image(MatImage on C++ side) instance goes out of scope
- V8's GC triggered destructing Image instance
- MatImage destructor called
- **Underlying cv::Mat's data pointer was manually deleted.**
- Underlying pointer to cv::Mat is deleted.
- cv::Mat's destructor is called just before program exists
- **cv::Mat's destructor attempts to delete data pointer which was already deleted**
- Segfault      

Because I used delete[] on Mat's data pointer, the Mat's destructor does not know data pointer is not valid.        
Hence delete[] is called again on a dangling pointer and that causes segfault.      
Releasing memory at runtime is important to handle videos. So MatImage destructor must exist.     

Re-implemented destructor:
```
template <class numericType>
Nodoface::MatImage<numericType>::~MatImage() {
    // std::cout<<"MatImage: Destructor called"<<std::endl;
    this->mat->release(); // Let Mat handle the data pointer
    delete this->mat;
}
```
## Another crash
This time it was because of `showImage()`. 
When showImage() receives Image object from Node, it has to convert the underlying Mat back to BGR (I stored Mat as RGB in Image class). 
Program crashes when calling cvtColor(). Not sure why it crashed but it has something to do with source Mat and destination Mat sizes. 
I fixed it by passing a clone of source as the destination mat in cvtColor() call.
This increases execution time slightly so I will have to find a way to avoid cloning the Mat.

## Task 1/2: Amazon Rekognition datatypes
There are quite a lot of data types defined by Amazon Rekognition API.
I separated them into basic types ie classes that do not depend on any other class and derived types which depend on basic types.

## Task 2/2: Database of embeddings
I first proposed to use MongoDB but after running some tests and benchmarks with PostgreSQL, I found it 4 times faster. PostgreSQL supports JSON so it works well with Node.
### Dev/Test/Prod databases
There will be 3 databases.
- pmrdev: Will be locally available for development.
- pmrtest: Will be hosted on AWS RDS for testing before deployment
- pmrprod: WIll be hosted on AWS RDS for production ready server.

I am using DBeaver Community Edition to manage the databases.

### Credentials and Db config
The server's 'config' folder present on root of the project has two files viz. secrets.json and dbconfig.json. 
These are not kept under git.
#### Sample secrets.json:
```
{
    "environment": "dev",
    "test": {
        "db": {
            "user": "pmrtest",
            "password": "password_here"
        }
    },
    "prod": {
        "db": {
            "user": "pmrprod",
            "password": "password_here"
        }
    },
    "dev": {
        "db": {
            "user": "pmradmin",
            "password": "password_here"
        }
    }
}
```
Actual file may contain more fields as the project progresses.
#### Sample dbconfig.json
```
{
    "test": {
        "host": "link-to-hosted-database",
        "port": 5432,
        "dbname": "pmrtest"
    },
    "prod": {
        "host": "link-to-hosted-database",
        "port": 5432,
        "dbname": "pmrdb"
    },
    "dev": {
        "host": "127.0.0.1",
        "port": 5432,
        "dbname": "pmrdev"
    }
}
```
Apart from these, to enable password based login, appropriate changes have to be made in `pg_hba.conf` file of PostreSQL program both locally and on hosted database.
### DB operations
I am using `pg` npm package to interact with Postgres from Node. A DbConnection class is to maintain DB connection.
It reads db credentials from secrets.json and other info like db name, host and port from dbconfig.json file. 
The database variant to use are controlled by "environment" key in secrets.json.
All queries are in Queries module with placeholders for arguments. `DbHelper` executes these queries by replacing $x with corresponding entry in an array passed as 2nd argument to pg.Pool insance.

## Next week work
I am delayed by 2 days now. Next week work will include writing end points for face detection and recognition. But before that, a simple classifier for finding face label from embedding have to be created. I forgot to include it in proposed timeline.

# References

[1] [Program crash on showImage(): 'corrupted size vs. prev_size'](https://stackoverflow.com/questions/49628615/understanding-corrupted-size-vs-prev-size-glibc-error)     
[2] [Program crash on showImage(): 'double or free corruption'](https://stackoverflow.com/questions/2902064/how-to-track-down-a-double-free-or-corruption-error)   
[3] [PostgresSQL login by password error](https://stackoverflow.com/questions/18664074/getting-error-peer-authentication-failed-for-user-postgres-when-trying-to-ge)    
[4] [Postgres basic tutorial on Medium](https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e)   
[5] [Postgres user management examples](https://www.ntchosting.com/encyclopedia/databases/postgresql/create-user/#Create_a_user_with_the_command_line)      
[6] [Using array type to store face embedding in Postgres](https://www.postgresql.org/docs/9.1/arrays.html)     
[7] [Nodejs Postgres tutorial](https://blog.logrocket.com/setting-up-a-restful-api-with-node-js-and-postgresql-d96d6fc892d8/)   
[8] [Passing array to Postgres query from Nodejs](https://stackoverflow.com/questions/52091628/pass-array-to-postgresql-query-nodejs)