# Installation of go

default repository only supports golang1.6, but mp-quic requires 1.9.
```
sudo apt-get install software-properties-common python-software-properties
sudo add-apt-repository ppa:gophers/archive
sudo apt-get update
sudo apt-get install golang-1.9-go git

# set up golang environment
# add the following lines to ~/.bash_profile
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
export GOROOT=/usr/lib/go-1.9
export PATH=$GOROOT/bin:/home/bigmac-admin/youtube-dl:$PATH

# clone it
go get -t -u github.com/lucas-clemente/quic-go
go get -t -u github.com/bifurcation/mint
go get -t -u github.com/qdeconinck/mp-quic

# do some nasty rename
cd ~/go/src/github.com/lucas-clemente/
mv quic-go quic-go.old
cp /../../qdeconin/mp-quic ./quic-go -R


# revert to some older version git repo
cd ~/go/src/github.com/bifurcation/mint
git checkout 4230e37b9cfeac7eca62b6a9589601e5e59235ea # maybe I'm wrong

```
# VMs
- use lts0 and lts1, both with Ubuntu Xenial(16.04)

# Network
- The linux bridge is br-ctl0, vproxy0 and vproxy1 ( guest ens3, ens5 and ens7)

# original quic-go example programs

the original quic-go example requires (self-signed) certificates. For server side, the directory to the cert and privatekey can be specified by -certpath argument. For client side, you should copy the cert file to /usr/share/ca-certificates and run ``sudo update-ca-certificates``.


```bash
#server side
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout <private_key> -out <cert_file>
go run ./example/main.go -v -bind 0.0.0.0:6121 -certpath <your_path_or_default>

#client side, be sure to add the cert into trusted list.
copy self generated <cert_file> to /usr/share/ca-certificates/extra  make sure using ".crt" suffix
sudo update-ca-certificates
sudo dpkg-reconfigure ca-certificates   // enable the <cert_file>
sudo echo "<ip_addr_of_lts0> lts0" >> /ets/hosts
go run ./example/client/main.go https://lts0:6121

```
#convenience scripts
$HOME/bash  for running server client
$QUIC_HOME  for accessing quic home

# how to enable get mp-quic running

the -m (multipath) option doesn't work, I need to try to figure out what is happening under the hood.

the -m option is wrapped in a quic.Config struct, then passed to h2quic.RoundTripper, then registered as a Transport of http.Client

TODO:
Looking throught the code, the only possible way of using this boolean is the newClientSession() method in client.go

We can tell from the callstack that HTTP Get method will call a series of callbacks ( h2quic.RoundTripper.RoundTrip-> h2quic.client.RoundTrip-> quic.client.establishSecureConnection )

# example code and core API

## server side business logic
user code skeleton of an echo server looks like this:
```
quic.listenAddr -> server
server.Accept() -> session
session.AcceptStream() -> stream
stream.read/write
```
quic.listenAddr() listen on a UDP port, then use pconnManager (pcm.run()) thread to keep refreshing the physical configuration, handle timeout, etc.

after initial config was set up, call go server.serve() to run in background. The only important channel on server side is pconnManager.rcvRawPackets, it calls server.handlePacket() to perform logic. this pipe is written by pconnManager.listen() routine

server.Accept() is simple, it just pull a session from queue. the queue is filled by handlePacket()

h2quic is a bit more complicated because it registers callbacks to net.http. 

Server side uses h2quic.Server, it is a wrapper for quic.server class and handles TLS handshake ,forward stream to net.http.Handler. You usually don't see Accept() AcceptStream() calls in h2quic.Server class


## client side business logic
echo client looks like this:
```
session = quic.DialAddr(serverAddr,...);
stream = session.OpenStreamSync();
stream.Write()
io.ReadFull(stream,buf)
```

there's no quic.client but only some "free functions" in quic/client.go. so h2quic.client holds only a quic.session member.
P.S. mp-quic has shitty naming convention, I don't know why h2quic.client/h2quic.Server can even coexist.

h2quic.client uses RoundTripper (implements http.RoundTripper) class to wrap the h2quic.client class. when a request comes in, the roundtripper will spawn a new client call client.RoundTrip, and finally that calls quic.Stream.Read/Write.

call stack look like this:
```
- quic.Stream.Read //only appends to buffer, the background thread will handle link layer io
- client.RoundTripper // async Read/Write
- h2quic.RoundTripper // create (DialAddr) or find client
- net.http.Get() // get method
```

Notice: h2quic.RoundTripper itself is stateless (only lookup a global map) , but client needs to keep track of session, thus it needs to be stateful.

The semantic of RoundTripper is a routine ( blocking io ) that first write request then retrieve response. The example http2 client uses this blocking io fashion. 

The link layer (dirty) jobs are handled by pconnManager(?). The callstack look like this.
```
pconnManager.run() //in a new thread
pconnManager.setup()
quic.DialAddr //initialize and handshake
h2quic.RoundTripper // no client found, create one
```

the pathes are set up like this
```
pathManager.createPaths() // for M local addrs and N remote addrs set up M*N paths
pathManager.run()
pathManager.setup()
session.setup()
newClientSession
DailAddr
```


```
#0  .(*client).createNewSession (c=0xc42013a070, 
    negotiatedVersions= []/internal/protocol.VersionNumber, 
    conn=..., ~r2=...)
    at /client.go:394
#1  0x0000000000691128 in .(*client).establishSecureConnection
 (c=0xc42013a070, conn=..., ~r1=...)
    at /client.go:207
#2  0x00000000006909c9 in .DialNonFWSecure (pconn=..., 
    remoteAddr=..., host="quic.clemente.io:443", tlsConf=0x0, config=0xc420080280, 
    pconnMgrArg=0xc42013a000, ~r6=..., ~r7=...)
    at /client.go:137
#3  0x0000000000690d7d in .Dial (pconn=..., remoteAddr=..., 
    host="quic.clemente.io:443", tlsConf=0x0, config=0xc420080280, pconnMgrArg=0xc42013a000, 
    ~r6=..., ~r7=...)
    at /client.go:153
#4  0x0000000000690501 in .DialAddr (
    addr="quic.clemente.io:443", tlsConf=0x0, config=0xc420080280, ~r3=..., ~r4=...)
    at /client.go:60
#5  0x00000000006c4b85 in /h2quic.(*client).dial (
    c=0xc4200a2420, ~r0=...)
    at /h2quic/client.go:85
#6  0x00000000006c996f in /h2quic.(*client).RoundTrip.func1
    () at /h2quic/client.go:159
#7  0x0000000000460c6e in sync.(*Once).Do (o=0xc4200a2478, f={void (void)} 0xc42004f798)    tionID)                                                                                    │    at /usr/lib/go-1.9/src/sync/once.go:44
#8  0x00000000006c58d8 in /h2quic.(*client).RoundTrip (
    c=0xc4200a2420, req=0xc4200f2100, ~r1=0x14, ~r2=...)
    at /h2quic/client.go:158
#9  0x00000000006c8f33 in /h2quic.(*RoundTripper).RoundTripOpt
 (r=0xc4200112f0, req=0xc4200f2100, opt=..., ~r2=0x0, ~r3=...)
    at /h2quic/roundtrip.go:102
#10 0x00000000006c933a in /h2quic.(*RoundTripper).RoundTrip (
    r=0xc4200112f0, req=0xc4200f2100, ~r1=0xc4200112f0, ~r2=...)
    at /h2quic/roundtrip.go:107
#11 0x00000000005d31d9 in net/http.send (ireq=0xc4200f2100, rt=..., deadline=..., 
    resp=0xc42000e068, didTimeout={void (bool *)} 0xc42004fca0, err=...)
    at /usr/lib/go-1.9/src/net/http/client.go:249
#12 0x00000000005d2e6d in net/http.(*Client).send (c=0xc420011320, req=0xc4200f2100, 
    deadline=..., resp=0xc42000e068, didTimeout={void (bool *)} 0xc42004fd18, err=...)
    at /usr/lib/go-1.9/src/net/http/client.go:173
#13 0x00000000005d450d in net/http.(*Client).Do (c=0xc420011320, req=0xc4200f2100, 
    ~r1=0x7fffffffe75b, ~r2=...) at /usr/lib/go-1.9/src/net/http/client.go:602
#14 0x00000000005d4027 in net/http.(*Client).Get (c=0xc420011320, 
    url="https://quic.clemente.io", resp=0x0, err=...)
    at /usr/lib/go-1.9/src/net/http/client.go:393
#15 0x00000000006ca3ba in main.main.func1 (hclient=0xc420011320, &wg=0xc420012610, 
    addr="https://quic.clemente.io")
    at /example/client/main.go:54
#16 0x0000000000458a51 in runtime.goexit () at /usr/lib/go-1.9/src/runtime/asm_amd64.s:2337
#17 0x000000c420011320 in ?? ()
#18 0x000000c420012610 in ?? ()
```


