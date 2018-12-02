## lnrpc

Source for the [lnrpc nuget package](https://www.nuget.org/packages/lnrpc/).

After installing the package, usage is as follows: 


```c#
string macaroonLocation = @"C:\<YOUR_LND_DIRECTORY>\data\chain\bitcoin\simnet\admin.macaroon";
string tlsCertLocation = @"C:\Users\<YOUR_USER>\AppData\Local\Lnd\tls.cert";
string lndRPCLocation = "localhost:" + "<LND_GRPC_PORT>"; // default is 10009
var lndPassword = Google.Protobuf.ByteString.CopyFromUtf8("<YOUR_LND_PASSWORD>");

byte[] macaroonBytes = File.ReadAllBytes(macaroonLocation);
var macaroon = BitConverter.ToString(macaroonBytes).Replace("-", ""); // hex format stripped of "-" chars
System.Environment.SetEnvironmentVariable("GRPC_SSL_CIPHER_SUITES", "HIGH+ECDSA");
var cert = File.ReadAllText(tlsCertLocation);
var sslCreds = new SslCredentials(cert);
var channel = new Grpc.Core.Channel(lndRPCLocation, sslCreds);

// unlock the lnd node using WalletUnlockerClient
// var walletClient = new Lnrpc.WalletUnlocker.WalletUnlockerClient(channel);
/// var unlockRequest = new UnlockWalletRequest() { WalletPassword = lndPassword };
/// var unlockResponse = walletClient.UnlockWallet(unlockRequest, new Metadata() { new Metadata.Entry("macaroon", macaroon) });

// call getinfo using LightningClient
var lightningClient = new Lnrpc.Lightning.LightningClient(channel);
var getinfoRequest = new GetInfoRequest();
var getinfoResponse = lightningClient.GetInfo(getinfoRequest, new Metadata() { new Metadata.Entry("macaroon", macaroon) });


// Response streaming example, listen for invoice received
var invoiceReceivedRequest = new InvoiceSubscription();
using (var call = lightningClient.SubscribeInvoices(invoiceReceivedRequest))
{
    while (await call.ResponseStream.MoveNext())
    {
        var invoice = call.ResponseStream.Current;
        Console.WriteLine(invoice.ToString());
    }
}


// bidirectional streaming, pay to a user every 2 secs
using (var call = lightningClient.SendPayment(new Metadata() { new Metadata.Entry("macaroon", macaroon) }))
{
    var responseReaderTask = Task.Run(async () =>
    {
        while (await call.ResponseStream.MoveNext())
        {
            var payment = call.ResponseStream.Current;
            Console.WriteLine(payment.ToString());
        }
    });

    foreach (SendRequest sendRequest in SendPayment())
    {
        await call.RequestStream.WriteAsync(sendRequest);
    }
    await call.RequestStream.CompleteAsync();
    await responseReaderTask;
}


IEnumerable<SendRequest> SendPayment()
{
    while (true)
    {
        SendRequest request = new SendRequest() {
            DestString = "<DEST_PUB_KEY>",
            Amt = 100,
            PaymentHashString = "<R_HASH>",
            FinalCltvDelta = <CLTV_DELTA>
        };
        yield return request;
        System.Threading.Thread.Sleep(2000);
    }
}



// Instead of adding macaroons to every request like we do above,
// we can add it to a channel using interceptors. 
Task AddMacaroon(AuthInterceptorContext context, Metadata metadata)
{
    metadata.Add(new Metadata.Entry("macaroon", macaroon));
    return Task.CompletedTask;
}
var macaroonInterceptor = new AsyncAuthInterceptor(AddMacaroon);
var combinedCreds = ChannelCredentials.Create(sslCreds, CallCredentials.FromInterceptor(macaroonInterceptor));

var channelWithMacaroon = new Grpc.Core.Channel(lndRPCLocation, combinedCreds);
var clientWithMacaroon = new Lnrpc.Lightning.LightningClient(channelWithMacaroon);

// now every call will be made with the macaroon already included
clientWithMacaroon.GetInfo(new GetInfoRequest())

```

