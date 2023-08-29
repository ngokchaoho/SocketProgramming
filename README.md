# SocketProgramming
This is a OS course project implementing HTTP1.0 like protocol

# Project1 README file

This is Ho, Ngok Chao's  Readme file. Spring 2023 for Project 1 of CS6200 Graduate Intro to OS

## Warmup (Echo)
### Understanding

This is a typical Echo Server - Client pair, Client sends 15 bytes content to server and then server send the same content back. At the end, the client receive it and export to stdout.

### Implementation and Testing

Referencing beej's guide, in order to receive and send

- On server side, I called:
```c=
socket() // to get a socket file descriptor
setsockopt() // to allow reusing the port
bind() // bind to a port
listen() // set maximum pending connection 
accept() // get a new socket file descriptor
recv() // using the new socket fd to receive
send() // using the new socket fd to send
```
- On client side, I called:
```c=
socket() // to get a socket file descriptor
connect() // make a connection session
send()
recv()
```
`char buffer[16]` on stack is used as buffer to send and receive message. Since no loss is expected, it is assumed 15 bytes can be put into and read from buffer by calling send() and recv() one time.

Tested with `./echoclient -m messages`

## Warmup (Transferfile)

### Understanding
Server read a pre-defined file and send to client connecting to it. How many bytes each time the server process can read and send is not predetermined, hence I have to keep track of how much is read and sent each time, and continue from there.

### Implementation and Testing

Same kind of basic setup as previous echo exercise is needed.

On client side putting `recv()` and `fwrite()` into a while loop until number of bytes received is zero.

On server side putting `fread()` into a while loop until number of bytes available to read is zero; for each chunk of bytes read, they are further handled by `sendall()` which itself is a function sending via `send()` chunk by chunk until all bytes are sent. This is function is taken from *Beej's Guide to Network Programming*.

Tested with `./transferserver -f files` 

## Part 1 (gfclient)
### Understanding

Part 1 is about implementing a file transfer service according to the Getfile Protocol; for the client part, we expect to send request like `<scheme> <method> <path>\r\n\r\n` and receive `<scheme> <status> <length>\r\n\r\n<content>` The server part is expected to receive and send the opposite. Following the existing structure of `gfclient.c`, all parameters, connection state is saved in a `struct gfcrequest_t` so that a single source of truth is shared every where and no global variable is needed.

While the struct of `gfcrequest_t` for all info is create via `malloc` on heap which is unnecessary for a single threaded server, it make it easier to further extend to multi-threading given that the struct is on heap hence can be shared among threads.
### Flow of Control
The most important function in `gfclient.c`  is `gfc_perform` and hence I desmontrate its control flow as following
![](https://i.imgur.com/wQr3HIc.png)

### Implementation and Testing
```c=
#define BUFSIZE 2048
```
I set the buffer for recv() and send() to 2048 (not too small or too large), and hence the maximum size each chunk receive/send is 2048 bytes.

The longest valid server response header (from start until \r\n\r\n) is less than 50 bytes, hence if header is longer than 50 bytes, it is safe to return -1 (invalid response)
```
GETFILE OK 18446744073709551616\r\n\r\n
```

```c=
 struct gfcrequest_t
 {
  const char *server;
  const char *path;
  unsigned short port;
  void (*writefunc)(void *, size_t, void *);
  void (*headerfunc)(void *, size_t, void *);
  void *writearg, *headerarg;
  char buf[BUFSIZE];
  char header[BUFSIZE];
  size_t header_index;
  size_t bytes_received;
  size_t terminated;
  int sockfd;
 };
 ```
As mentioned in *understanding*, my implementation put all information into gfcrequest_t above as a single source of truth.

Following `gfclient_download.c`, we can see the usage of `gfclient`
- `gfc_global_init`: empty, no global var is used
- `gfc_create(gfcrequest_t **gfr)`: allocates memory for this struct on heap and then it is initialised via the following
- `gfc_set_port` : set the port
- `gfc_set_path` : set path
- `gfc_set_server`: set address for server
- `gfc_set_writefunc(&gfr, writecb)`: register a write down function pointer into this struct
- `gfc_set_writearg`: put arguemnts for writing function into this struct
- `gfc_perform`: use information mentioned above, to create sockfd and receive info into `header` (information before `\r\n\r\n`) and `buf`(content) and then use registered callback `writefunc` to export, and close sockfd when all done. Details in control flow graph above
- `gfc_cleanup`: free the memory for the struct on heap
- `gfc_global_cleanup`: empty, no global var is used.

**testing**
Using `gfserver_main_half_test` from Project 1 Interop Thread, my client can download file succesfully.

Using tests modified from 6200-tools, I ran the following test

- test 1: receive 8192 bytes from server side to check if the request is valid, SIGTERM the server to close connection and check if client exit 
- test 2: Server send the header, in two pieces. Then send 1 byte payload. Client should succeed.
- test 3: Server send the header without `\r\n\r\n`, and then close the sockfd, client should report gfc_perform return -1, received 0 bytes and status is INVALID
- test 4: prematurely closed connection during transfer of the message body
- test 5: non decimal file length in response, gfc_perform returns -1
- test 6: Send a file containing CR LF, received files with 2 bytes
- test 7: sned a Send a file containing CR LF CR LF, received files with 4 bytes
- test 8: send file with 2 ^ 31 - 1 bytes 
- test 9: Serve a random sized file with random-sized buffers of random bytes, and compare the SHA1 hash.

> Note: size_t, stroull (convert string to unsigned long long) is used hence the maximum file size supported should be **18,446,744,073,709,551,615**


## Part 1 (gfserver) 
### Understanding
The server part would answer to request `<scheme> <method> <path>\r\n\r\n` and send `<scheme> <status> <length>\r\n\r\n<content>` Following the existing structure of `gfserver.c`, server related info is stored in struct `gfserver_t` while connection specific information is stored in `gfcontext_t`.
    
Just like `gfcrequest_t` in the client part, `gfserver_t` is created on heap which is beneficial for sharing purpose if we decided to make some functions to be handled by multiple threads; Similarly `gfcontext_t` is also created on heap, later in part 2, it is shown how we have multiple threads working together, with a global queue of pointer to pointer to `gfcontext_t`.
    
From Piazza, `\r\n\r\n` is always expected, hence the server is calling recv() until we see `\r\n\r\n` instead of setting a timeout.
    
### Flow of Control
The most important function in `gfserver.c` is `gfs_serve` and hence I desmontrate its control flow as following
    
![](https://i.imgur.com/CUf2W2P.png)


### Implementation and Testing
```c=
struct gfserver_t
{
  unsigned short port;
  int max_npending;
  gfh_error_t (*handler_callback)(gfcontext_t **, const char *, void*);
  void* handlerarg;
  char buf[BUFSIZE];
};

struct gfcontext_t
{
  char header[BUFSIZE];
  char path[BUFSIZE];
  size_t header_index;
  int sockfd;
};
```
Per **understanding** server parameters are put into `gfserver_t`, per-request related information is put into `gfcontext_t`
    
Following `gfserver_main.c`, we can see the usage of `gfserver`
- content_init: Init content library
- gfserver_create: malloc gfserver_t
- gfserver_set_handler: set handling function, which would call `gfs_send` and `gfs_sendheader`
- gfserver_set_port: set listening port of the server
- gfserver_sex_maxpending: set maximum pending connection
- gfserver_set_handlerarg: set arg for handler function
- gfserver_serve: run the server, described in **Flow of Control**
    
**testing**
Using `gfclient_main_half_test` from Project 1 Interop Thread, my server can send file succesfully.
    
Using tests modified from 6200-tools, I ran the following test (particularly skipping those without `\r\n\r\n` is in request)
- test 1: Send an complete request, FILE_NOT_FOUND is expected
- test 2: Send invalid scheme and method. No handling is expected.
- test 3: Using 4096 bytes PATH, FILE_NOT_FOUND is expected
- test 4: request is sent in tiny pieces, FILE_NOT_FOUND is expected
- test 5: Path is / alone. FILE_NOT_FOUND is expected
- test 6: Path without /. `b'GETFILE INVALID\r\n\r\n'` is expected
- test 7: Send an complete request, and read in small pieces with delays.Ctrl+C to kill the client and see server give up this connection
- test 8: Send a complete request, but close before receiving the response. Server should keep running
- test 9: Send an incomplete request, then close. Server should keep running.




## Part 2 (multi-threaded gfserver)
    
### Understanding
This part of the project is to write `gfserver_main.c` using gflib above in multithreaded manner. A global variable of `steque_t` is used; The main thread (boss) would put incoming request (gfcontext_t) into this global steque_t, slave threads would take job from the `steque_t`.
    
A global mutex **gServerSharedMemoryLock** is used to protect the global `steque_t` shared among threads.

### Flow of Control
![](https://i.imgur.com/FkNIdvE.png)

    
    
    
### Implementation and Testing
Global variable used
```c=
steque_t server_steque;
pthread_mutex_t gServerSharedMemoryLock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t gServerReadPhase = PTHREAD_COND_INITIALIZER;
pthread_cond_t gServerWritePhase = PTHREAD_COND_INITIALIZER;
```
steque_t is a singly linked list implemented deque.
```c=

typedef struct steque_node_t{
  steque_item item;
  struct steque_node_t* next;
} steque_node_t;

typedef struct{
  steque_node_t* front;
  steque_node_t* back;
  int N;
}steque_t;
```

Mutex protected section for access global variable **server_steque**
```c=
pthread_mutex_lock(&gServerSharedMemoryLock);
		while (!steque_isempty(&server_steque)) {
			pthread_cond_wait(&gServerWritePhase,&gServerSharedMemoryLock);
		}
        ...
        steque_enqueue(&server_steque, item);
        ...
pthread_mutex_unlock(&gServerSharedMemoryLock);
```

Struct representing a request
```c=
typedef struct {
    gfcontext_t *ctx;
    char *path;
    void* arg; 
} handler_item;
```
The global variable shared we want to protect is **server_steque** whose member is pointer to a `handler_item` (information for incoming request).
    
For the main thread, mutex is used in `gfs_handler`, the predicate for granting access is when steque_isempty; when job taken by slave thread, slave thread would wake up main thread to check for this predicate and then enqueue if access granted.
    
On the other hand, the predicate for granting access for slave thread, is when steque is not empty i.e. we have job to take; if so, pop the steque and take the job and wake up the boss thread. 
    
Details of control flow is described in **Flow of Control**. 

**testing**
Using `gfclient_main_half_test` from Project 1 Interop Thread, my server can send file succesfully.
    
Using tests modified from 6200-tools, I ran the following test (particularly skipping those no ``\r\n\r\n` is in request)
- test 1: Send an complete request, FILE_NOT_FOUND is expected
- test 2: Send invalid scheme and method. No handling is expected.
- test 3: Using 4096 bytes PATH, FILE_NOT_FOUND is expected
- test 4: request is sent in tiny pieces, FILE_NOT_FOUND is expected
- test 5: Path is / alone. FILE_NOT_FOUND is expected
- test 6: Path without /. `b'GETFILE INVALID\r\n\r\n'` is expected
- test 7: Send an complete request, and read in small pieces with delays.Ctrl+C to kill the client and see server give up this connection
- test 8: Send a complete request, but close before receiving the response. Server should keep running
- test 9: Send an incomplete request, then close. Server should keep running.




## Part 2 (multi-threaded gfclient)
    
### Understanding
We need to implement in `gfclient_download.c` a multithreaded version usage of `gfclient`. Main thread would be the boss thread enqueuing download requests and slave threads would take job (callling gflib gfc_perform to send and recv) to request and download files.


###  Flow of Control

![](https://i.imgur.com/h2cpTXG.png)




### Implementation and Testing
```c=
unsigned short port = 10880;
char *server = "localhost";
int nrequests = 12;
int gfinished_requests = 0;
steque_t work_steque;
pthread_mutex_t gSharedMemoryLock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t gWritePhase = PTHREAD_COND_INITIALIZER;
pthread_cond_t gReadPhase = PTHREAD_COND_INITIALIZER;
```
The above global variable is used to facilitate multithreading design.

A global `steque_t` is used to keep the info for user input requests. slave threads are used to pop info out of steque_t and call gfc_perform in gflib to receive file. Sepereate condition variable and opposite predicate are used for boss thread and slave thread so that only one thread would have access to the global steque_t to avoid error. (enqueue and pop at the same time would lead to incorrect result due to they are not atomic operation for linked list).

The predicate for boss thread to enqueue is when steque_isempty is True; we wrap the mutex lock section in a for loop for the n requests we get from user.
```c=
for (int i = 0; i < nrequests; i++) {
    pthread_mutex_lock(&gSharedMemoryLock);
        while (!steque_isempty(&work_steque)) {
          pthread_cond_wait(&gWritePhase, &gSharedMemoryLock);
        }
        steque_enqueue(&work_steque, workload_get_path());
        pthread_cond_signal(&gReadPhase);
    pthread_mutex_unlock(&gSharedMemoryLock);
}
```
    
The predicate for slave thread to pop is when steque_isempty is not True and possible number of pending job to finish job >=0; For the slave thread, if there is no possible pending job to finish then we can just exit, otherwise call `gfc_perform`.
    
**testing**
Using `gfserver_main_half_test` from Project 1 Interop Thread, my client can download file succesfully.

Using tests modified from 6200-tools, I ran the following test

- test 1: receive 8192 bytes from server side to check if the request is valid, SIGTERM the server to close connection and check if client exit 
- test 2: Server send the header, in two pieces. Then send 1 byte payload. Client should succeed.
- test 3: Server send the header without \r\n\r\n, and then close the sockfd, client should report gfc_perform return -1, received 0 bytes and status is INVALID
- test 4: prematurely closed connection during transfer of the message body
- test 5: non decimal file length in response, gfc_perform returns -1
- test 6: Send a file containing CR LF, received files with 2 bytes
- test 7: sned a Send a file containing CR LF CR LF, received files with 4 bytes
- test 8: send file with 2 ^ 31 - 1 bytes 
- test 9: Serve a random sized file with random-sized buffers of random bytes, and compare the SHA1 hash.
    

## References

- Instructions to use vscode codespace https://github.com/ngokchaoho/GT_GIOS_ENV/blob/main/articles/codespace_instructions.md

- waitpid https://linux.die.net/man/2/waitpid

- getaddrinfo https://man7.org/linux/man-pages/man3/getaddrinfo.3.html

- Project 1 Interop Thread https://piazza.com/class/lco3fd6h1yo48k/post/79

- 6200-tools https://github.gatech.edu/cparaz3/6200-tools/

- Beej's Guide to Network Programming https://beej.us/guide/bgnet/  (function sendall is taken from here, and warmup for echo is done referencing this)

