+++
title = "Async Components, FileReader, and Angular"
date = "2020-05-03"
author = "Andrew Alvarez"
authorTwitter = "kweizarlog" #do not include @
cover = ""
tags = ["angular", "filereader", "async", "javascript"]
keywords = ["angular", "filereader", "async", "javascript"]
description = "The problem: User input can't be trusted. When a file is uploaded to the internet, you can check on its Mime type, but can you trust it?"
showFullContent = false
+++

The problem: User input can't be trusted. When a file is uploaded to the internet, you can check on its Mime type, but can you trust it?

While developing a drag and drop interface to upload media, my partner and I architected a series of processes to reduce the load on the server-side.

We built an app that took audio and video and sent it to a google API for further processing. We didn't want the server to perform file validation since that we would need processes to deal with garbage data. We thought it might be a better idea to validate our media on the front-end to only send the right type of files.

Let's say you upload a .html file and check on its type, you'll get "text/html"; when you upload a .mp3 file you get "audio/mpeg".
So what's the problem with this? The browser is reading correctly your files! But it actually isn't.

![Nani!?](https://media1.giphy.com/media/9DgcVlvwcAAjetmNrp/giphy.gif)

<figcaption>NANI!?</figcaption>

If I change my audio file's extension from .mp3 to .txt, yes you'll "break" your file but you'll also be able to fool the browser as it will scan it and output "text/plain as it's Mime type.
No one wants this to happen, we need to ensure the integrity of our app.
The solution: Our Angular component needs to read the file and determine its actual content by its Magic Numbers.

```javascript
/*
In my component, I have declared a method called validateMime, 
it takes a Blob type since its what we get when our files go online.
*/
export class DragAndDrop {
  validateMime(blob: Blob) {
    // Our validation implementation goes here
  }
  readAsync(blob: Blob) {
    // Our FileReader logic goes here
  }
}
```

Our tool to go is FileReader, a native JS object that allows us to read the file contents or the raw data buffer! You can read the specs of the FileReader object here.
To execute FileReader, you'll have to call one of its 5 methods. In this case, I'll be using .readAsBinaryString()

```javascript
reader.readAsBinaryString(blob.slice(0, 4));
reader.onload = (e) => {
  const mime = e.target.result
    .toString()
    .split("")
    .map((bit) => bit.codePointAt(0).toString(16).toUpperCase())
    .join("");
  console.log(` - ${mime} - `);
};
```

Before we proceed, I must note that FileReader's methods work asynchronously, everything that happens within onload() won't be accessible on the outer scopes. We will have to change some of the component's methods. This is where async/await comes to the rescue.

```javascript
readAsync(blob: Blob) {
  return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = (e) => {
        resolve(e.target.result.toString()
                               .split('')
                               .map(bit =>
        bit.codePointAt(0).toString(16).toUpperCase())
                                       .join('');
      };
    reader.onerror = () => {
        reject (new Error ('Unable to read..'));
    };
    reader.readAsBinaryString(blob.slice(0, 4));
  });
}
```

Our method returns a promise that will either execute its reject statement if for some reason it can't read the file or it will return the resolved value if it succeeds. The blob yields an ArrayBuffer type value which we'll just slice to get the four first bytes. These will tell us the real type of the file. The method chain will transform these bytes from Unicode into a string that represents the Magic Numbers of our file.

```javascript
async validateMime(blob: Blob) {
    try {
        const contentBuffer = await this.readAsync(blob);
        // Validation Process
        let isValid = false;
        acceptedMimeTypes.forEach(mimetype => {
        if ( contentBuffer === mimetype.magicNo) { isValid = true; }
    });
    return true;
    }
    catch (err) {
      console.log(err);
    }
}
```

As you can see, processFile() is an async method. It will await until readAsync returns (asynchronously) a value to assign it to contentBuffer, a variable I created to compare its value to a list of the accepted Mime Types for my app. If the Mime type shows up in my list it'll return true and it will accept my file!

I hope you liked this article, feel free to give me feedback or contact me if you have any questions. I will keep posting the challenges I keep encountering when developing apps and narrate how I solved them.
Thank you for reading!
