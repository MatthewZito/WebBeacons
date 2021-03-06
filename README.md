# Web Beacons

This repository collates experiments concerned with establishing *low-resource* web-beacons, automating remote data-logging, and utilizing many of the more esoteric HTTP status codes and the myriad interesting things we can do with them.

To date, the following endeavors have endured trial successfully<sup>[1](#fn1)</sup>:
 - Embedding a counter beacon inside of image byte-code. 
 - Data harvesting via Beacon API
 - Data harvesting via pure JS tracking pixel (written from scratch; no pre-generated tokens)

## What is a Web Beacon?
A Web beacon is a HTML-embedded script used to monitor the behavior of the given user visiting the website or opening the email to which it has been applied.

The beacon is typically a resource (e.g. a 1x1 pixel GIF) which, when requested by the client, passes along to the server information such as the IP address of the machine that retrieved the resource, the time the beacon was viewed and for how long, the type of browser that retrieved the resource, et cetera.

## Embedding an Executable Binary Inside of a JPG (OS-agnostic)
This was a strange one. A friend asked me if there was a way to count how many times an image had been opened, to track its journey across the web. I endeavored to do this, knowing the solution would essentially involve tampering with embedded or compiled executables.

I've used pyinstaller here. This is a methodology commonly employed by malware developers; the original image is compiled with the executable (which simply makes a form POST to the campaign's Google Form document). macos by default does not display extensions, but I have applied the [right-to-left override](https://krebsonsecurity.com/tag/right-to-left-override/) to spoof the extension to jpg. I've tested various compilation configurations and have found something I am happy with, which works quietly and as designed on Windows, macOS, and Linux (tested on Ubuntu, Arch, and Kali thus far).

As you can see, every time the image is opened, the form is mysteriously updated...
![Demo](https://github.com/MatthewZito/WebBeacons/blob/master/embedded/imgbeacon.gif)

## Tracking Pixel
Tracking pixels have long been relied upon as a lightweight solution for analytics and data harvesting. The theory is quite simple, albeit clever: We take a standard GET request and manipulate it into a vehicle for delivering data *to* the resource server.

We utilize this 'GET conduit' by collecting the exfiltrated data into query parameters, which are then appended onto the `src` attribute/URI for a 1x1 pixel GIF (standard is a GIF given it is in this context essentially the smallest resource one can serve). The GIF is criticial here: we need to serve an `img` tag in our webpage (or email) so the client will request said resource.

The email or browser client will request the GIF resource; this request will include all of the harvested parameters, formatted and ready to be persisted into a database.

I've been using MongoDB across my tracking campaigns; a NOSQL data distribution seemed to me most robust for accommodating the types of data operations which will be conducted here (I am developing a client for assembling topological graphs of collected data, so this format is especially useful here). I typically assign my pixels a tracking ID; database collections are instantiated dynamically and correlate to this ID:

```
const upsertEntryQuaIpv4 = async (dbClient, ipv4Address, entryData, pixelIdentifier) => {
    const result = await dbClient.db(`tracking_pixel_${pixelIdentifier}`).collection("user_data")
        .updateOne(
            { ipv4_address: ipv4Address }, 
            { $set: entryData, $setOnInsert: { createdAt: new Date() }, $currentDate: { "lastModified": true }, $inc: { "pings": 1 } },
            { upsert: true }
        );
}
```
You can see that we will also set a few fields upon insert: date created (e.g. the first date our pixel has tracked *X* ipv4 address), date last modified (data we last encountered *X* ipv4 address), and pings, or the number of times *X* ipv4 address has requested the tracking pixel. 

So, what happens when the request is made? This is where tracking pixels and beacons are strange. We'll discuss the Beacon API later (making POST requests to which a 204 Status Code - and nothing else - is returned), but even in this particular instance, we are not especially concerned with handling errors (we just elect to 'pass', or do nothing, rather). Unlike the Beacon, we *do* need to respond with the requested 1x1 pixel GIF.

Most tracking pixel designs I've encountered actually serve a static GIF. In fact, given tracking pixels are almost exclusively proprietary technologies, you're perhaps looking at the only wholly open-source tracking pixel that isn't using an API-generated tracking script or token. 

Instead of serving a static GIF, we are going to generate our pixel ad hoc. That is, when the resource is requested, we create it server-side: 

First, we construct the hex distribution needed.
```
const imgData = [
    0x47,0x49, 0x46,0x38, 0x39,0x61, 0x01,0x00, 0x01,0x00, 0x80,0x00, 0x00,0xFF, 0xFF,0xFF,
    0x00,0x00, 0x00,0x21, 0xf9,0x04, 0x04,0x00, 0x00,0x00, 0x00,0x2c, 0x00,0x00, 0x00,0x00,
    0x01,0x00, 0x01,0x00, 0x00,0x02, 0x02,0x44, 0x01,0x00, 0x3b
    ];
```
Allocate hex data to instantiate new Buffer
```
const imgBuf = Buffer.from(imgData);
```
A tracking pixel of campaign :id is requested; we persist our data and serve the buffer
```
app.get("/:id/pixel.png", async (req, res) => {
        // collate data
        ...
        try {
            // persist to DB
            ...
            res.writeHead(200,{
                'Content-Type': 'image/gif',
                'Content-Length': imgData.length,
                });
        } 
        finally {
        res.end(imgBuf);
        }
    });
```
##### <a name="fn1">1</a>: What constitutes a *trial* (and *success* thereof) will be elucidated in an imminent update to this document.

Disclaimer: This software and all contents therein were created for research use only. I neither condone nor hold, in any capacity, responsibility for the actions of those who might intend to use this software in a manner malicious or otherwise illegal.
