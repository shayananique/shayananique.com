---
title: TCP-based sockets on Windows Phone with support for large amounts of data
date: 2012-03-25T21:49:00+00:00
categories:
  - Programming
  - Windows Phone
---
Network communication through sockets is a welcome addition to the Windows Phone platform. Personally, I enjoy using those because of the tremendous performance gain compared to WCF services. One topic, however, seems to be rarely covered when it comes to TCP-based communication between the server (presumably running on a desktop machine) and the client (Windows Phone) = transmission of large amounts of data.

A lot of existing samples assume that the communication, although bi-directional, is done with small data units. For example, [here is a pretty good sample](http://msdn.microsoft.com/en-us/library/5w7b7x5f.aspx) that I used to build the asynchronous server. The reading callback looks like this:

<pre>public static void readCallback(IAsyncResult ar) {
    StateObject state = (StateObject) ar.AsyncState;
    Socket handler = state.WorkSocket;

    // Read data from the client socket.
    int read = handler.EndReceive(ar);

    // Data was read from the client socket.
    if (read &gt; 0) {
        state.sb.Append(
             Encoding.ASCII.GetString(state.buffer,0,read));
        handler.BeginReceive(state.buffer,0,
             StateObject.BufferSize, 0,
            new AsyncCallback(readCallback), state);
    } else {
        if (state.sb.Length &gt; 1) {
            // All the data has been read from the client;
            // display it on the console.
            string content = state.sb.ToString();
            Console.WriteLine(content);
        }
        handler.Close();
    }
}</pre>

It works perfectly fine on the desktop machine. I am also adding a terminator verification statement that allows me to check whether the whole data set was received:

<pre>public void ReadCallback(IAsyncResult ar)
{
StateObject state = (StateObject)ar.AsyncState;
Socket handler = state.WorkSocket;

int bytesRead = handler.EndReceive(ar);

if (bytesRead &gt; 0)
{
string contentString =

Encoding.UTF8.GetString(state.buffer, 0, bytesRead);

if (state.buffer[bytesRead - 1] != terminator[0])
{
state.ContentString.Append(contentString);
handler.BeginReceive(state.buffer, 0, StateObject.BUFFER_SIZE,

0, new AsyncCallback(ReadCallback), state);
}
else
{
contentString = contentString.Replace(terminator, "");
state.ContentString.Append(contentString);
}
}

}
</pre>

Although seemingly simple, the data receiving process on a Windows Phone is handled in a different manner. There is no direct implementation of[Socket.BeginReceive](http://msdn.microsoft.com/en-us/library/system.net.sockets.socket.beginreceive.aspx). Instead, developers work with [Socket.ReceiveAsync](http://msdn.microsoft.com/en-us/library/system.net.sockets.socket.receiveasync(v=vs.95).aspx). Many samples give you this piece of code:

<pre>public string Receive()
{
string response = "Operation Timeout";

if (_socket != null)
{
SocketAsyncEventArgs socketEventArg = new SocketAsyncEventArgs();
socketEventArg.RemoteEndPoint = _socket.RemoteEndPoint;
socketEventArg.SetBuffer(new Byte[MAX_BUFFER_SIZE],

0, MAX_BUFFER_SIZE);

socketEventArg.Completed +=
new EventHandler&lt;SocketAsyncEventArgs&gt;
(delegate(object s, SocketAsyncEventArgs e)
{
if (e.SocketError == SocketError.Success)
{
response = Encoding.UTF8.GetString(
e.Buffer, e.Offset, e.BytesTransferred);
response = response.Trim('\0');
}
else
{
response = e.SocketError.ToString();
}

clientDone.Set();
});

clientDone.Reset();
socket.ReceiveAsync(socketEventArg);
clientDone.WaitOne(TIMEOUT_MILLISECONDS);
}
else
{
response = "Socket is not initialized";
}

return response;
}
</pre>

Notice the problem with this snippet. Once the string is received, that will be it. The inbound data length will be limited to the size of the buffer, and the maximum chunk size. The buffer size cannot always be fixed, and it is safe to assume that the developer does not want to allocate too much space for that in one iteration. The default socket chuck size cannot always be accommodated either.

The resolution comes in a continuous invocation of [Socket.ReceiveAsync](http://msdn.microsoft.com/en-us/library/system.net.sockets.socket.receiveasync(v=vs.95).aspx):

<pre>private void ProcessReceive(SocketAsyncEventArgs e)
{
if (e.SocketError == SocketError.Success)
{
Socket sock = e.UserToken as Socket;
string dataFromServer =

Encoding.UTF8.GetString(e.Buffer, 0, e.BytesTransferred);

if (dataFromServer.EndsWith(terminator))
{
dataFromServer = dataFromServer.Replace(terminator, "");
builder.Append(dataFromServer);

sock.Shutdown(SocketShutdown.Send);
sock.Close();
clientDone.Set();

}
else
{
builder.Append(dataFromServer);
sock.ReceiveAsync(e);
}

}
else
{
clientDone.Set();
throw new SocketException((int)e.SocketError);
}
}
</pre>

Here is where the terminator character plays a major role. Look at** _if (dataFromServer.EndsWith(terminator)) –_** that is the flag that shows whether more data is coming or not. With a receiving structure like I showed above, it is possible to keep receiving data until the connection is closed (happens all the time) or the terminator character is detected.

> **NOTE:** If you are wondering what terminator character to use, try going with non-standard ones. For example, _**\u00BE**._ Otherwise, chances are that the actual “terminator” is not the terminating character.