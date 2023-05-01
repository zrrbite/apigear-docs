# Internal dev

## Architecture
Overview of ApiGear Architecture.



## Building and stuff

```
mkdir build
cd build
cmake ..
cmake --build .
```

What're we getting  here?

```
martin.kjeldsen@CDW-VII37VLKMRC D:\dev\apigear\objectlink-core-cpp.git\build
$ cmake --build .
MSBuild version 17.5.1+f6fdcf537 for .NET Framework

  1>Checking Build System
  1>Building Custom Rule D:/dev/apigear/objectlink-core-cpp.git/src/CMakeLists.txt
  basenode.cpp
  protocol.cpp
  types.cpp
  consolelogger.cpp
  clientnode.cpp
  clientregistry.cpp
  remotenode.cpp
  remoteregistry.cpp
  Generating Code...
  olink_core.vcxproj -> D:\dev\apigear\objectlink-core-cpp.git\build\src\Debug\olink_core.lib
  1>Building Custom Rule D:/dev/apigear/objectlink-core-cpp.git/tests/CMakeLists.txt
  test_main.cpp
  test_olink.cpp
  test_protocol.cpp
  test_client_registry.cpp
  test_client_node.cpp
  test_remote_registry.cpp
  test_uniqueidstorage.cpp
  test_remote_node.cpp
  Generating Code...
     Creating library D:/dev/apigear/objectlink-core-cpp.git/build/tests/Debug/tst_olink.lib and object D:/dev/apigear/objectlink-core-cpp.git/build/tests/Debug/tst_olin
  k.exp
  tst_olink.vcxproj -> D:\dev\apigear\objectlink-core-cpp.git\build\tests\Debug\tst_olink.exe
  1>Building Custom Rule D:/dev/apigear/objectlink-core-cpp.git/CMakeLists.txt

martin.kjeldsen@CDW-VII37VLKMRC D:\dev\apigear\objectlink-core-cpp.git\build
```

Now you've got all the things and can run the tests:

```
EPICGAMES+martin.kjeldsen@CDW-VII37VLKMRC MINGW64 /d/dev/apigear/objectlink-core-cpp.git/build/tests/Debug (main)
$ ./tst_olink.exe
[info   ] RemoteRegistry.addObjectSource: demo.Calc
[info   ] ClientRegistry.addSink: demo.Calc
[info   ] ClientNode.linkRemote: demo.Calc
[debug  ] writeMessage [10,"demo.Calc"]
[info   ] RemoteRegistry.getObjectSource: demo.Calc
[info   ] RemoteRegistry.linkRemoteNode: demo.Calc
linkeddemo.Calc[info   ] ClientNode.handleInit: demo.Calc{"total":1}
[info   ] ClientRegistry.getSink: demo.Calc
CalcSink.olinkOnInit: demo.Calc{"total":1}
[info   ] ClientRegistry.unsetNode: demo.Calc
[info   ] ClientRegistry.setNode: demo.Calc
[info   ] RemoteRegistry.removeObjectSource: demo.Calc
[info   ] ClientRegistry.removeSink: demo.Calc
[info   ] ClientRegistry.removeSink: demo.Calc
[info   ] RemoteRegistry.removeObjectSource: demo.Calc
invoke replydemo.Calc/add6===============================================================================
All tests passed (332 assertions in 10 test cases)

```

You can actually get a list of tests.
```
EPICGAMES+martin.kjeldsen@CDW-VII37VLKMRC MINGW64 /d/dev/apigear/objectlink-core-cpp.git/build/tests/Debug (main)
$ ./tst_olink.exe -l
All available test cases:
  link
  setProperty
  signal
  invoke
  protocol
  client registry
  Client Node
  server registry simple tests without threads
  storage
  Remote Node
10 test cases

```

## Logging

e.g. `emitLog(LogLevel::Warning, "Trying to add node, but it is already gone. Node NOT added.");`


# Errors

If client sends an invalid message, the server responds with `[ ERROR, 0, 0, "olink.error.invalid_message" ]`. Where does this happen on the server side?

For inspiration, we're probably trying to do something like this. In any case we're making kson array:

```
nlohmann::json Protocol::invokeMessage(int requestId, const std::string& methodId, const nlohmann::json& args)
{
    return nlohmann::json::array(
        { MsgType::Invoke, requestId, methodId, args }
    );
}
```

Something like this...

```
    return nlohmann::json::array(
        { MsgType::Invoke, requestId, methodId, args }
    );
```
=> `[ERROR , 0, 0, "olink.error.invalid_message" ]`. Does that mean that the IDs don't matter here? It should actually be something like, but what is an "invalid message" even?

```
    return nlohmann::json::array(
        { MsgType::ERROR, 0, 0, "olink.error.invalid_message" }
    );

```

The term "invokeMsg" is wrong. it actually just returns some json. It's the `emitWrite(msg);` that does it.

In `protocol.cpp` theres:

```
nlohmann::json Protocol::errorMessage(MsgType msgType, int requestId, const std::string& error)
{
    return nlohmann::json::array(
                { MsgType::Error, msgType, requestId, error }
                );
}
```

Which is only being used inside `test_protocol.cpp`.



# TODO
- Suggest internal documentation
- Suggest more code comments
- What is a remotenode?
- What is a clientnode?
- What is a sink?
- BaseNode::emitWrite and BaseNode::onWrite allow a remotenode (one connection) to
- Probably don't log before knowing whether or not it actually goes well. e.g.

```
void ClientNode::invokeRemote(const std::string& methodId, const nlohmann::json& args, InvokeReplyFunc func)
{
    emitLog(LogLevel::Info, "ClientNode.invokeRemote: " + methodId);
    int requestId = nextRequestId();
    std::unique_lock<std::mutex> lock(m_pendingInvokesMutex);
    m_invokesPending[requestId] = func;
    lock.unlock();
    nlohmann::json msg = Protocol::invokeMessage(requestId, methodId, args);
    emitWrite(msg);
}
```
- We definitely need to define these error texts somewhere. e.g. `olink.error.invalid_message`.
-
