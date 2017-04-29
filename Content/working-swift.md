
# Working with the Swift Object Storage Service
After starting succesfully the Swift Object Storage Service, it is possible to store and retrive data in containers using the CLI Swift client.

Source the user credentials and query the service using the CLI client
```
# source /root/keystonerc_admin
# swift stat
        Account: AUTH_a6c6d1f171e649ba8af7a7a6cf63c3b9
     Containers: 0
        Objects: 0
          Bytes: 0
X-Put-Timestamp: 1421939769.00333
    X-Timestamp: 1421939769.00333
     X-Trans-Id: tx0df39bb7df404386aa13b-0054c11438
   Content-Type: text/plain; charset=utf-8

```

Create a containers and upload some content
```
# swift post container01
# swift upload container01 somedata.file
```

View the list and content of containers
```
# swift list
container01
# swift list container01
somedata.file
# swift stat container01
         Account: AUTH_22bdc5a0210e4a96add0cea90a6137ed
       Container: container01
         Objects: 1
           Bytes: 3354194
        Read ACL:
       Write ACL:
         Sync To:
        Sync Key:
   Accept-Ranges: bytes
      X-Trans-Id: txd6672b6f11024f97a953a-0056f54424
X-Storage-Policy: Policy-0
     X-Timestamp: 1458906767.31714
    Content-Type: text/plain; charset=utf-8
```

Download the content from a container
```
# swift list test
somedata.file
# swift download test somedata.file
somedata.file [auth 0.259s, headers 0.314s, total 0.335s, 44.082 MB/s]
```

To access objects via REST API make the container public readable
```
# swift post -r '.r:*' test
# swift stat test -v
             URL: http://controller:8080/v1/AUTH_22bdc5a0210e4a96add0cea90a6137ed/test
      Auth Token: 6fa1a82383d244a9adeccbcb7d9db3fc
         Account: AUTH_22bdc5a0210e4a96add0cea90a6137ed
       Container: test
         Objects: 1
           Bytes: 3354194
        Read ACL: .r:*
       Write ACL:
         Sync To:
        Sync Key:
   Accept-Ranges: bytes
      X-Trans-Id: txaa934da2d12e4bff9896a-0056f5475f
X-Storage-Policy: Policy-0
     X-Timestamp: 1458906767.38224
    Content-Type: text/plain; charset=utf-8
```

Access objects via its public URL. The public URL that is the HTTP endpoint from where access the Object Storage. It includes the Object Storage API version number and the account name
```
# wget http://controller:8080/v1/AUTH_22bdc5a0210e4a96add0cea90a6137ed/test/somedata.file
Connecting to controller:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3354194 (3.2M) [application/text]
Saving to: ‘somedata.file.1’
100%[=================================>] 3,354,194   --.-K/s   in 0.04s
2016-03-25 15:14:52 (91.2 MB/s) - ‘somedata.file.1’ saved [3354194/3354194]
```

Delete the container and its content
```
# swift delete container01
somedata.file
```
