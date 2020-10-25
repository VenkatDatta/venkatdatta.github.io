---
layout: post
title:  "Build a Simple KV store"
date:   2020-10-10 09:29:20 +0700
categories: database c++
---
To understand Database better, I have started writing a simple KV store. 

The KV store will support Get, Insert operations. It will be using LevelDB as storage engine and GRPC as the network interface.

## Table of Contents
{:.no_toc}

* TOC
{:toc}

### Tools Required

**- Install GRPC:**
You can find the official instructions for setting up [here](https://grpc.io/docs/languages/cpp/quickstart/)

**- Install Glog & Gflags:**
     ```sudo apt-get install libgoogle-glog-dev libgflags-dev```

**- Install LevelDB:** 
    
```bash
    sudo apt-get install libsnappy-dev 
    wget https://leveldb.googlecode.com/files/leveldb-1.9.0.tar.gz
    tar -xzf leveldb-1.9.0.tar.gz
    cd leveldb-1.9.0
    make
    sudo mv libleveldb.* /usr/local/lib
    cd include
    sudo cp -R leveldb /usr/local/include
    sudo ldconfig
```
		
### The Design

<figure>
<img src="/assets/img/simplekv.png" alt="Design of Simple KV">
<figcaption>Fig 1. design.</figcaption>
</figure>


### Defining the Interface Definition Language
Interface Definition Language(IDL) is used to describe a software component's Application Programming Interface(API). In case of GRPC, both IDL and underlying message interchange format can be defined using Protocol Buffers.

**- Step 1:** Define the structure of request/response message and the operations supported in the **proto** file. 

```java
syntax = "proto3";
package PACKAGE_NAME;

//The Key Value service definition
service KVService {
  //Sends request for getting value for particual key.
  rpc Get(ReqKey) returns (RespValue) {}
  //Sends request to insert a key,value pair.
  rpc Insert(ReqKeyValue) returns (RespValue) {}
};

//The request message containing the Key
message ReqKey {
    int64 qid = 1;
    repeated string key = 2;
};

//The request message containing the Key,Value
message ReqKeyValue {
    int64 qid = 1;
    repeated string key = 2;
    repeated string value = 3;
};

//The response message containing the Value
message RespValue {
    int32 status = 1;
    repeated string value = 2;
};
```

**- Step 2:** Compiling the **proto** file to generate message and service classes.
This can be achieved by running the below commands - 
```
$ protoc -I ../../protos --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ./kvService.proto
$ protoc -I ../../protos --cpp_out=. ./protos/kvService.proto
```
Running these commands generates the following files in your current directory.
* **kvService.pb.h**, the header which declares your generated message classes
* **kvService.pb.cc**, which contains the implementation of your message classes
* **kvService.grpc.pb.h**, the header which declares your generated service classes
* **kvService.grpc.pb.cc**, which contains the implementation of your service classes

These contain:
* All the protocol buffer code to populate, serialize, and retrieve our request and response message types
* A class called **KeyValueService** that contains
    * a remote interface type (or stub) for clients to call with the methods defined in the KeyValue service.
    * two abstract interfaces for servers to implement, also with the methods defined in the KeyValue service.

Same can be achieved using a [Makefile](https://github.com/grpc/grpc/blob/v1.33.1/examples/cpp/route_guide/Makefile) that runs **protoc**

**- Step 3:** Write an Implementation for GET, INSERT operations

In this step, we need to add the handlers for GET, INSERT operations performed by the client. 

```cpp

class KVServiceImpl final : public PACKAGE_NAME::KVService::Service {
public:

  /**
   * Implementation for GET operation.
  **/
  Status Get(ServerContext* context, const ReqKey* request,
                  RespValue* reply) override {
    if (request->key_size() == 0) {
        LOG(ERROR) << "Reqid=" << request->qid() 
                   << " miss request key.";
        return Status::OK;
    }
    LOG(INFO) << "Reqid=" << request->qid() 
              << " key_size=" << request->key_size()
              << " Sample key=" << request->key(0);
    reply->clear_value();
    for (int i = 0; i < request->key_size(); ++i) {
        std::string value;

        //TODO - Define KeyValue Engine to perform GET operation.
        
    }
    return Status::OK;
  }
  /**
  * Implementation for Insert operation.
  **/
  Status Insert(ServerContext* context, 
                const ReqKeyValue* req_key_value,
                RespValue* relay) override {
      int key_size = req_key_value->key_size();
      int value_size = req_key_value->value_size();
      if (key_size != value_size) {
          LOG(WARNING) << "Insert cancelled for key value size not match [" 
                       << key_size << "/" << value_size << "]";
          return Status::CANCELLED;
      }
      LOG(INFO) << "Reqid=" << req_key_value->qid() 
                << " key_size=" << req_key_value->key_size()
                << " Sample key=" << req_key_value->key(0);
      
      //TODO - Define KeyValue Engine to perform INSERT operation.

      return Status::OK;
  }
};
```


### Building a Basic DB Engine
The main goal of the DB engine is to store/retrieve data in the database. To start with simplicity, I'm sharing a basic implmentation of a plain DB engine that uses Map for storing the Key,Value in memory.

```cpp
class PlainEngine : public BaseKVEngine {
    public:
        bool init() {
            return true;
        }
        int insert(const std::string& key, const std::string& value) {
            _map.insert(std::make_pair(key, value));
            return 0;
        }
        int insert(const std::pair<std::string, std::string>& kv) {
            _map.insert(kv);
            return 0;
        }
        bool has_key(const std::string& key) {
            if (_map.find(key) != _map.end()) {
                return true;
            }
            return false;
        } 
        std::string operator[] (const std::string& key) {
            return _map[key];
        } 
    private:
        std::unordered_map<std::string, std::string> _map;

};
```
### Integrating with LevelDB
Now, we need to extend the plain DB engine to use the put/get functionality provided by the LevelDB.
```cpp
class LevelDBEngine : public BaseKVEngine {
    public:
        using LevelOpRet = std::function<leveldb::Status(const leveldb::WriteOptions,
                           const std::string&, const std::string&)>;
        LevelDBEngine() {
        }
        bool init() {
            leveldb::Options options;
            options.create_if_missing = true;
            leveldb::DB* db;
            leveldb::Status status = leveldb::DB::Open(options, FLAGS_db_dir, &db);
            if (!status.ok()) {
                LOG(ERROR) << "open leveldb [" << FLAGS_db_dir << "] failed.";
                return false;
            }
            LOG(INFO) << "open leveldb [" << FLAGS_db_dir << "] complete.";
            _db.reset(db);
            return true;
        }
        int insert(const std::string& key, const std::string& value) {
            LevelOpRet f = std::bind(&leveldb::DB::Put, (leveldb::DB*)_db.get(),
                               std::placeholders::_1, std::placeholders::_2,
                               std::placeholders::_3); 
            LOG(INFO) << "kv insert [" << key << "] [" << value << "]";
            return leveldb_work(f, std::string("Put"), leveldb::WriteOptions(), key, value);
        }
        int insert(const std::pair<std::string, std::string>& kv) {
            return insert(kv.first, kv.second);
        }
        bool has_key(const std::string& key) {
            std::string value;
            leveldb::Status status = _db->Get(leveldb::ReadOptions(), key, &value);
            if (status.ok()) {
                return true;
            }
            if (status.IsNotFound()) {
                return false;
            }
            LOG(ERROR) << "some error when has_key [" << key << "]";
            return false;
        }
        std::string operator[] (const std::string& key) {
            std::string value;
            leveldb::Status status = _db->Get(leveldb::ReadOptions(), key, &value);
            if (!status.ok()) {
                LOG(ERROR) << "some error when Get [" << key << "]";
                return "";
            }
            return value;
        }
    private:
        std::unique_ptr<leveldb::DB> _db;
};
```

### Define Server
The role of the server is to listen on a particular port and pass the request recieved to GRPC implementation.
```cpp
int RunServer() {
  std::string server_address("0.0.0.0:50051");
  KVServiceImpl<LevelDBEngine>> *service =  new(std::nothrow)KVServiceImpl<<LevelDBEngine>>();

  if (!service->init()) {
    LOG(WARNING) << "KVServiceImpl init failed.";
    return -1;
  }

  ServerBuilder builder;
  builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
  builder.RegisterService(service);
  std::unique_ptr<Server> server(builder.BuildAndStart());
  LOG(INFO) << "Server listening on " << server_address;
  server->Wait();
  return 0;
}
```
### Start Playing!!
```
Final Project Structure looks like this.

├── cmake
├── CMakeLists.txt
├── CMakeScripts
│   ├── FindGflags.cmake
│   ├── FindGlog.cmake
│   └── FindLevelDB.cmake
└── src
    ├── db
    │   ├── client
    │   │   └── kv_client.cpp
    │   ├── db.cpp
    │   ├── engine
    │   │   ├── base_kv_engine.cpp
    │   │   ├── leveldb_engine.cpp
    │   │   └── plainkv_engine.cpp
    │   ├── interface
    │   │   └── service.proto
    │   └── server
    │       └── kv_server.cpp
    └── include
        ├── base_kv_engine.hpp
        ├── leveldb_engine.hpp
        └── plainkv_engine.hpp

```