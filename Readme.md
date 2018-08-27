### lnrpc

Source for the [lnrpc nuget package](https://www.nuget.org/packages/lnrpc/).

After installing the package, usage is as follows: 


```C#
string macaroonLocation = @"C:\<YOUR_LND_DIRECTORY>\data\admin.macaroon";
string tlsCertLocation = @"C:\Users\<YOUR_USER>\AppData\Local\Lnd\tls.cert";
string lndRPCLocation = "localhost:" + "<LND_GRPC_PORT>";
var lndPassword = Google.Protobuf.ByteString.CopyFromUtf8("<YOUR_LND_PASSWORD>");

byte[] macaroonBytes = File.ReadAllBytes(macaroonLocation);
var macaroon = BitConverter.ToString(macaroonBytes).Replace("-", ""); // hex format stripped of "-" chars
System.Environment.SetEnvironmentVariable("GRPC_SSL_CIPHER_SUITES", "HIGH+ECDSA");
var cert = File.ReadAllText(tlsCertLocation);
var sslCreds = new SslCredentials(cert);
var channel = new Grpc.Core.Channel(lndRPCLocation, sslCreds);

// unlock the lnd node using WalletUnlockerClient
var walletClient = new Lnrpc.WalletUnlocker.WalletUnlockerClient(channel);
var unlockRequest = new UnlockWalletRequest() { WalletPassword = lndPassword };
var unlockResponse = walletClient.UnlockWallet(unlockRequest, new Metadata() { new Metadata.Entry("macaroon", macaroon) });

// after unlock, call getinfo using LightningClient
var lightningClient = new Lnrpc.Lightning.LightningClient(channel);
var getinfoRequest = new GetInfoRequest();
var getinfoResponse = lightningClient.GetInfo(getinfoRequest, new Metadata() { new Metadata.Entry("macaroon", macaroon) });
```

