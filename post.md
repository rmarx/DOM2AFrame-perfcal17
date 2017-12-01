

<style>
body 
{
    width: 650px;
}
.caption
{
	width: 100%;
	text-align: center;
	font-style: italic;
}

img
{
	max-width: 650px;
}
</style>

# DOM2AFrame : Putting the Web back in WebVR

## Browsing the Web in VR 

Earlier this year one of my students, [Sander Vanhove][lamasaurus], started his bachelor's thesis on WebVR. 
Virtual Reality is on the rise, but we felt that people are looking primarily at how to adapt games and video-content to this new platform: 
the many millions of webpages with existing interesting content seem to be forgotten and there are no best practices for bringing them into a VR context yet. 

[lamasaurus]: https://github.com/Lamasaurus

This is in my opinion made painfully clear by how modern top browsers support browsing in VR (if at all): they usually just map the page onto a very narrow ~2D plane,
acting like a floating screen/window in the 3D space. There is no immersion or depth, and worst of all: content specifically made for VR (like 360&deg; videos) isn't automatically integrated. 

** TODO : YOUTUBE VIDEO ** 

There is also nothing developers can do about this (yet): while CSS supports various 3D functionality, there is no CSS media-query for VR that would allow us to make a vr-version of a site (though support for this type of thing [is being worked on][custom-media-queries]). Even then, browsers would need additional directives to allow a full integration.  

[custom-media-queries]: https://drafts.csswg.org/mediaqueries-5/#script-custom-mq

Luckily, there is another standard in the works that might help with this: [WebVR][webvrrocks]. WebVR acts as a layer between the various Virtual Reality hardware setups and the 3D/WebGL capabilities of the HTML `<canvas>` element. Using these building blocks, we can indeed build a [fully immersive, VR-ready experience][webvrexamples]. The main problem is: **we cannot build this using our known tools of HTML, CSS and things like JavaScript click events**. Instead, we have to resort to the ever so slightly more complex shaders, 3D objects and raycasting: no more flexbox or rounded corners for you! This would result in having to maintain two very diverse versions of a website instead of a single codebase with specific directives per platform (as is the case with most mobile websites today). You would also possibly have to hire a different type of engineer to create this version. 

[webvrrocks]: https://webvr.rocks/
[webvrexamples]: https://medium.com/samsung-internet-dev/eleven-examples-of-how-webvr-is-being-used-today-cbcb214b816c


## Render HTML to a Texture

To solve this problem, we needed something that could render a typical webpage in 3D directly. A basic approach would be to first render the page to a 2D texture, and then map that texture onto a 3D object. If we create multiple 2D textures per page (e.g., one for the menu, one for the hero image, one for the contact form, etc.) and map them on separate 3D elements, we can transform them individually. Pseudocode could look something like this:

`let {tex2D, inputMap} = window.rasterize( DOMElement );`

Sadly, as it turns out, this type of _rasterization_ API is not available in JavaScript. The reasons for this are not fully clear to me (as this should be quite do-able, if non-trivial, to provide for browsers), but [the best reference I have found cites mainly security concerns][rasterize_security]. For example, Firefox does have something similar, but [you can only use it in privileged Firefox contexts][ff_rasterize] (what MDN refers to as chrome :)).

[ff_rasterize]: https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawWindow
[rasterize_security]: http://robert.ocallahan.org/2011/09/risks-of-exposing-web-page-pixel-data.html

Luckily, as usual, we were not the only ones thinking of this type of functionality. It turned out other people had already cracked this problem, using two very different workarounds! 

### Workaround 1: The `<foreignObject>` tag

The first method uses an obscure element of the Scalable Vector Graphis (SVG) format, namely the `<foreignObject>` [tag][foreignObject_spec]. Any HTML content placed within this tag will be rendered as if it was a normal page and then the result is injected back into the SVG. If we subsequently render the SVG to a texture with the existing `<canvas>` APIs, we can use it in 3D as well! 

However, there are several downsides: again due to security considerations, we need to jump through hoops to get acceptable results. For example, links to external resources (e.g., CSS, JS) are not supported so these need to be inlined/copied and images need to be base64 encoded. Das Surma has [a good write-up][dom2texture] on this method and the most mature library we have found that does this is [rasterizeHTML][rasterizeHTML].

A little additional work is needed to actually use these 3D textures in WebVR. 


You can see the results of this method using the rasterizeHTML and HTML2Three libraries on our testsite in Figure TODO.   

[foreignObject_spec]: https://developer.mozilla.org/en-US/docs/Web/SVG/Element/foreignObject
[dom2texture]: http://dassur.ma/things/dom2texture/
[rasterizeHTML]: https://cburgmer.github.io/rasterizeHTML.js/ 
[html2three]: https://github.com/marciot/html2three


## Transcode HTML to 3D objects and shaders 


---
**Thanks to Sander Vanhove and Wouter Vanmontfort for working with me on this topic.**

https://webvr.edm.uhasselt.be 

Last update: 04/12/2017  
Live version: https://github.com/rmarx/DOM2AFrame-perfcal17