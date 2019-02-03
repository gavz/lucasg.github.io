---
layout: post
title: "Discovering Nvidia NvBackend endpoint"
date: 2019-02-01
---

I regularly look at what's installed on my home computers, especially when it's come to third-party editors. I trust OS companies (Apple, Google, Microsoft) to ship relatively secure software, but much less for the others. In this state of mind, I looked at which processes bind server on my machine :

![NvBackendNetstat](/assets/NvBackendNetstat.png)

There is an unknown process which opens a listening socket on `127.0.0.1:23401`. Since it's bound on localhost, you can't actually access it from a host on the same LAN, only on the machine itself. However it might be possible to forge cross-origin requests from the browser if the server is not configured properly.

`NvBackend.exe` is part of files installed by Nvidia, and it's description is `NVIDIA Update Backend` which initially triggers my interest. An Update backend server that listen to socket requests : that's a recipe for disaster.

`C:\Program Files (x86)\NVIDIA Corporation\Update Core\NvBackend.exe` is a 32-bit application which is written mainly in C++ and has ~5000 functions detected by IDA. You just can't static reverse your way to the server handler here, which is an error I regularly see among juniors RE people. Before renaming every functions found, we need to have a better understanding of how we can interact with it (usually done in blackbox).

**TLDR : Nvidia expose it's updater APIs via XML-RPC over HTTP (on localhost) which sounds really bad, but thanks to `CORS` policy being rolled out by default in modern web browsers we avoid the worst.**

<!--more-->

## Reverse

Let's try to ping it first :

{%highlight Python %}
>>> import requests
>>> r = requests.get("http://localhost:23401")
Traceback (most recent call last):
  File "C:/Python36/lib/site-packages/urllib3/connectionpool.py", line 600, in urlopen
    chunked=chunked)
  File "C:/Python36/lib/site-packages/urllib3/connectionpool.py", line 384, in _make_request
    six.raise_from(e, None)
  File "<string>", line 2, in raise_from
  File "C:/Python36/lib/site-packages/urllib3/connectionpool.py", line 380, in _make_request
    httplib_response = conn.getresponse()
  File "C:/Python36/lib/http/client.py", line 1331, in getresponse
    response.begin()
  File "C:/Python36/lib/http/client.py", line 297, in begin
    version, status, reason = self._read_status()
  File "C:/Python36/lib/http/client.py", line 279, in _read_status
    raise BadStatusLine(line)
http.client.BadStatusLine: HTTP 400 Bad request
{%endhighlight Python %}




Tough luck, let's try POST requests then :

{%highlight Python %}
>>> import requests
>>> r = requests.post("http://localhost:23401")
>>> r
<Response [200]>
>>> r.headers
{'Server': 'NVIDIA Stream HTTP server/2008', 'Content-Type': 'text/xml', 'Content-Length': '533'}
>>> print(r.text)
"""
<?xml version="1.0"?>
<methodResponse>
   <fault>
      <value>
         <struct>
            <member>
               <name>faultCode</name>
               <value><int>12</int></value>
               </member>
            <member>
               <name>faultString</name>
               <value><string>data invalid or corrupted in 'RPC::PoppedDataConvertion; broken XML header'
IO error [#0] while [Sock::RecvTm]
</string></value>
               </member>
            </struct>
         </value>
      </fault>
   </methodResponse>
"""
{%endhighlight Python %}

Yes ! We get an answer, and better this is a custom error message. It looks like we have a XML-RPC server endpoint on the other end. Now we search for a matching string in the binary :

![NvBackendErrorMessage](/assets/NvBackendErrorMessage.PNG)

Now we have an entry point where we can set up a breakpoint and explore dynamically using a debugger (windbg in my case, but whatever userland debugger will fit the bill). Walking back the stack trace on IDA , we end up on this class (method names are my own) :
```
rdata:00557A40                 dd offset ??_R4BaseUserData@updtSTREAM@@6B@ ; const updtSTREAM::BaseUserData::`RTTI Complete Object Locator'
.rdata:00557A44 ; const updtSTREAM::BaseUserData::`vftable'
.rdata:00557A44 ??_7BaseUserData@updtSTREAM@@6B@ dd offset sub_471C9B
.rdata:00557A44                                         ; DATA XREF: sub_4775CF:loc_471C94↑o
.rdata:00557A44                                         ; sub_471C9B+A↑o ...
.rdata:00557A48                 dd offset ??_R4updtSTREAMRPC@@6B@ ; const updtSTREAMRPC::`RTTI Complete Object Locator'
.rdata:00557A4C ; const updtSTREAMRPC::`vftable'
.rdata:00557A4C ??_7updtSTREAMRPC@@6B@ dd offset __xml_rpc_push_data_convert
.rdata:00557A4C                                         ; DATA XREF: __xml_rpc_free+11↑o
.rdata:00557A4C                                         ; sub_471CFD+21↑o ...
.rdata:00557A50                 dd offset __xml_rpc_pop_data_convert
.rdata:00557A54                 dd offset sub_432631
.rdata:00557A58                 dd offset __xml_rpc_free
.rdata:00557A5C                 dd offset __xml_rpc_parse_data
.rdata:00557A60                 dd offset __xml_rpc_write_answer
.rdata:00557A64                 dd offset sub_476C56
.rdata:00557A68                 dd offset sub_472803
.rdata:00557A6C                 dd offset sub_471D64
.rdata:00557A70                 dd offset sub_471DB5
.rdata:00557A74                 dd offset sub_471DBF
.rdata:00557A78                 dd offset sub_477B93
.rdata:00557A7C                 dd offset sub_472B78
.rdata:00557A80                 dd offset sub_479BB6
.rdata:00557A84                 align 8
.rdata:00557A88 ; wchar_t aCDvsP4BuildSwR
```

Our breakpoint is triggered inside `__xml_rpc_write_answer`, but the method right before is much more interesting `__xml_rpc_parse_data`. Setting a breakpoint on this function allows us to break the application on new client requests
 :

{%highlight C %}
signed int __thiscall _xml_rpc_parse_data(_DWORD *this, int a2, wchar_t *a3, int a4)
{
  // [...]
  v10 = (_DWORD *)(40 * v6[32] + v9 + 36);
  if ( !*v10 )
  {
    // [...]
    if ( *(_DWORD *)(*(_DWORD *)a2 + 20) < 0x10u )
      v17 = *(const char **)a2;
    else
      v17 = *(const char **)v16;
    v18 = strstr(&v17[*((_DWORD *)v16 + 7)], "</methodName>");
    if ( !v18 )
    {
      v37 = L"RPC::PoppedDataConvertion; broken XML header";
      goto LABEL_41;
    }
    v20 = *(const char **)a2;
    if ( *(_DWORD *)(*(_DWORD *)a2 + 20) < 0x10u )
      v21 = *(const char **)a2;
    else
      v21 = *(const char **)v20;
    __methodName_anchor = strstr(&v21[*((_DWORD *)v20 + 7)], "<methodName>");
    v40 = __methodName_anchor;
    if ( !__methodName_anchor || v18 <= __methodName_anchor )
    {
      v37 = L"Error in request, queue error";
      _send_error_message(v35, (int)v36, v37);
      return 0;
    }
  // [...]
  return result;
}
{%endhighlight C %}

Just looking at strings inside the function, it's pretty clear it expect a `<methodName>` xml node. Let's try it :

{%highlight Python %}
>>> import requests
>>>  r = requests.get("http://localhost:23401", data='<?xml version="1.0"?>\n<methodName>AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA</methodName>')
                                                                                        # well we might get lucky, init ?
>>> r
<Response [200]>
>>> print(r.text)
"""
<?xml version="1.0"?>
<methodResponse>
   <fault>
      <value>
         <struct>
            <member>
               <name>faultCode</name>
               <value><int>8</int></value>
               </member>
            <member>
               <name>faultString</name>
               <value><string>element not found [XMLRPC's Secret header in HTTP]
</string></value>
               </member>
            </struct>
         </value>
      </fault>
</methodResponse>
"""
{%endhighlight Python %}

The error has changed : we've advanced in our reversing ! The error message is pretty interesting since it imply there is a secret seed/token the client must set in its request header. Unfortunately, we don't even know the custom header we need to use. That's where it's important to be efficient using a debugger. I've located a `strcmp` and set a breakpoint that dump every string compared :

```
0:006> bp NvBackend+0x70f3 ".printf /D \"%mu=%mu \\n\", ecx, eax;g" // this is a pretty fugly one-liner, but it does the job.
0:006> g
23401=23401 
23401=23401 
/crossdomain.xml=/ 
/=/crossdomain.xml 
/=/ 
/=/ 
X-NVRPC-SECRET=Host 
X-NVRPC-SECRET=User-Agent 
X-NVRPC-SECRET=Accept-Encoding 
X-NVRPC-SECRET=Accept 
X-NVRPC-SECRET=Connection 
X-NVRPC-SECRET=Content-Length 
DF1FCCCC6288WG92ABEE8GG0062DB6=
```

Interestingly, the xml-rpc server checks the path is either equal to `/` or `/crossdomain.xml`. The latter value smells funny, unfortunately a GET/POST requests does not return anything at all.

Anyway we found our secret custom header `X-NVRPC-SECRET`, and also it's 120-bit value `DF1FCCCC6288WG92ABEE8GG0062DB6`. let's try it out :

{%highlight Python %}
>>> import requests
>>> r = requests.get("http://localhost:23401", data='<?xml version="1.0"?>\n<methodName>myMethodName</methodName>', headers={'X-NVRPC-SECRET':"DF1FCCCC6288WG92ABEE8GG0062DB6"})
>>> r
<Response [200]>
>>> print(r.text)
"""
<?xml version="1.0"?>
<methodResponse>
   <fault>
      <value>
         <struct>
            <member>
               <name>faultCode</name>
               <value><int>9</int></value>
               </member>
            <member>
               <name>faultString</name>
               <value><string>Unknown Function name</string></value>
               </member>
            </struct>
         </value>
      </fault>
   </methodResponse>
"""
{%endhighlight Python %}

The returned error changed again ! We understanbly did gave a correct function name since we don't know which ones are valid. However, the `strcmp` breakpoint has dumped another interesting strings :

```
23401=23401 
23401=23401 
/crossdomain.xml=/ 
/=/crossdomain.xml 
/=/ 
/=/ 
X-NVRPC-SECRET=Host 
X-NVRPC-SECRET=User-Agent 
X-NVRPC-SECRET=Accept-Encoding 
X-NVRPC-SECRET=Accept 
X-NVRPC-SECRET=Connection 
X-NVRPC-SECRET=X-NVRPC-SECRET 
DF1FCCCC6288WG92ABEE8GG0062DB6=DF1FCCCC6288WG92ABEE8GG0062DB6 
myMethodName=GetApplicationList 
myMethodName=GetVersion 
myMethodName=SearchPaths_Get 
myMethodName=Streaming_GetOpsValues 
myMethodName=VOPS_GetPath 
myMethodName=VOPS_GetStatus 
```

Now we have access to certain function names exposed by the server. `GetVersion` seems pretty straightforward, let's try to call it :

{%highlight Python %}
>>> import requests
>>> r = requests.get("http://localhost:23401", data='<?xml version="1.0"?>\n<methodName>GetVersion</methodName>', headers={'X-NVRPC-SECRET':"DF1FCCCC6288WG92ABEE8GG0062DB6"})
>>> r
<Response [200]>
>>> print(r.text)
"""
<?xml version="1.0"?>
<methodResponse>
   <fault>
      <value>
         <struct>
            <member>
               <name>faultCode</name>
               <value><int>23</int></value>
               </member>
            <member>
               <name>faultString</name>
               <value><string>XML-RPC request invalid: wrong header
</string></value>
               </member>
            </struct>
         </value>
      </fault>
   </methodResponse>
"""
{%endhighlight Python %}

Huh, that didn't work. Before spending hours tracing calls and stepping instructions, let's see how an xml-rpc call is spec'ed (http://xmlrpc.scripting.com/spec.html):

![NvBackendXmlRpc](/assets/NvBackendXmlRpc.png)

Well, we better wrap our `methodName` xml node in a `methodCall` one :

{%highlight Python %}
>>> import requests
>>> r = requests.get("http://localhost:23401", data='<?xml version="1.0"?>\n<methodCall><methodName>GetVersion</methodName><params></params></methodCall>', headers={'X-NVRPC-SECRET':"DF1FCCCC6288WG92ABEE8GG0062DB6"})
>>> print(r.text)
"""
<?xml version='1.0' encoding='utf-8' ?>
<methodResponse>
  <params>
    <param>
      <value>
        <struct>
          <member>
            <name>Major</name>
            <value>
              <i4>10</i4>
            </value>
          </member>
          <member>
            <name>Minor</name>
            <value>
              <i4>4</i4>
            </value>
          </member>
          <member>
            <name>BuildNumber</name>
            <value>
              <i4>0</i4>
            </value>
          </member>
          <member>
            <name>Description</name>
            <value>
              <string>NVIDIA Update Backend</string>
            </value>
          </member>
          <member>
            <name>ReleaseDate</name>
            <value>
              <dateTime.iso8601>2015-06-29T00:00:00Z</dateTime.iso8601>
            </value>
          </member>
        </struct>
      </value>
    </param>
  </params>
</methodResponse>
"""
{%endhighlight Python %}

Success ! We managed to make a successful call to this interface endpoint. Now let's look at endpoint security.

## Security

This xml-rpc server is listening on a unknown port (I didn't managed to locate the port init routine in IDA) and expose "dangerous" API like `GetHardwareInfo` or `Downloader_AddURL`, which might be accessible from a remote attacker.

Firstly, it only listen to `localhost` so these API are not exposed to an attacker with an access to the same LAN. However it can be vunlerable to `CSRF` if not configured correctly. Fortunately, there is two things that mitigate this attack vector :

* [`CORS`](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) Web browsers companies clearly identify this attack vector and worked to limit it. Their hypothesis revolve more around CSRF between two websites/remote servers, but it also work to block local-only attacks :

![NvBackendCorb](/assets/NvBackendCorb.PNG)

The xml-rpc server only listen to HTTP requests (not HTTPS) and since the new big push to Let's Encrypt, there not many HTTP-only websites left (you can't make HTTP requests from an HTTPS page due to mixed content policy). Fortunately, there are still some web security experts which keep on delivering source code over HTTP :p.

The browser almost comptely shut down the `CSRF` attack vector, since we have to fallback on non-cors requests which are severly constrained.

* `X-NVRPC-SECRET` : this a custom HTTP request header implemented by the XML-RPC server which holds a "secret" key needed to access the xml-rpc APIs. By chance, the use of this header combined with CORS policy completly destroyed my `CSRF` approach since `no-cors` requests (the only one not blocked by the browser) can't set up custom headers in their request. Additionnaly, this secret is a GUID generated at process's runtime using `ole32!CoCreateGuid` and written down in a local section `.\Local\{AA45D379-DBB1-4AF4-833B-39E697EDCCA4}` which is not accessible from a remote attacker.

All in all, there is a string of complications which make the xml-rpc endpoint not exploitable from a remote scenario, but it's pretty tenuous security in my opinion. Since this endpoint is clearly meant to be accessible only from a local machine, it would be better to use an appropriate link layer, like a `NamedPipe` or a `ncalpc` RPC server.

### APIs

| methodName       |
| ------------- |
| AddFeedback |
| Application_ApplyOPS |
| Application_GetLastSettingsAction |
| Application_GetSettings |
| Application_HasSettings |
| Application_RevertOPS |
| ApplicationScanningEnabled |
| AutoApplyEnabled |
| CheckUpdatesNow |
| ClaimLicense |
| Downloader_AddURL |
| Downloader_GetList |
| Downloader_Pause |
| Downloader_Proceed |
| Downloader_Remove |
| EnableApplicationScanning |
| EnableAutoApply |
| EnableEventLogging |
| EventLoggingEnabled |
| EventLoggingEnabled |
| FRLGetState |
| FRLSetState |
| GetApplicationList |
| GetCheckFrequency |
| GetEnableAutomaticDriverDownload |
| GetEnableUpdates |
| GetEnableUpdateType |
| GetHardwareInformation |
| GetLastCheckTime |
| GetLastGeolocation |
| GetLastOPSChangeTime |
| GetLastOPSStatus |
| GetPackageList |
| GetSearchBetaVersions |
| GetSHIM |
| GetSUGAR |
| GetSupportedApplications |
| GetTranslation |
| GetUpdatesList |
| GetVersion |
| Installer_EnableDRS |
| Installer_EnableGFE |
| Installer_EnableNotifius |
| Installer_EnableUpdatus |
| IsFRLSupported |
| IsUpdateTypeSupported |
| MarkOOTB |
| NeedOOTB |
| NewPollInfo |
| ScrubApplications |
| SearchPaths_Add |
| SearchPaths_Get |
| SearchPaths_Remove |
| SetCheckFrequency |
| SetEnableAutomaticDriverDownload |
| SetEnableUpdates |
| SetEnableUpdateType |
| SetSearchBetaVersions |
| Streaming_GetApplicationDataPath |
| Streaming_GetOpsValues |
| Streaming_SetOps |
| Streaming_UnsetOps |
| TCSaveInformation |
| VOPS_GetPath |
| VOPS_GetStatus |