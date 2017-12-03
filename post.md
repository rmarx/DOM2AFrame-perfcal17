

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

This post is about the [DOM2AFrame proof-of-concept library][d2a_github], which **transcodes typical HTML/CSS webpages to WebVR compatible UIs** on-the-fly, while maintaining support for full interaction and animation at good enough&trade; framerates. This fun project has helped us get deeper insights in CSS handling, the browser's rendering pipeline, supported and missing JavaScript APIs and the performance of WebGL. 

Some of the main takeaways:
- WebGL is amazingly fast and even relatively unoptimized JS code can get you good results 
- JavaScript cannot (easily) capture all state changes on a webpage (and MutationObserver isn't fantastic in this regard)
- CSS overflow:hidden and browser font rendering are magical things 
- TODO

TODO: add sander's repo to the README of my repo 

TODO: add result video here 


[d2a_github]: https://github.com/rmarx/DOM2AFrame

## Browsing the Web in VR 

For my PhD in web performance, I had primarily been looking into network performance, with HTTP/2 and QUIC. 
To get a bit more acquainted with browser-side performance, together with bachelor student [Sander Vanhove][lamasaurus], we wanted to do something with the hot new kid on the block: [WebVR][webvrrocks]! The topic clickly turned into the question **(How) can we render webpages in 3D/VR**? And related: can we create UIs for games/3D experiences in plain HTML/CSS/JS? 

[lamasaurus]: https://github.com/Lamasaurus
[webvrrocks]: https://webvr.rocks/

It quickly became apparent that while some modern browsers do offer a _browse in VR_ option, this is often limited. As can be seen in the video below, they usually just map the page onto a very narrow ~2D plane, acting like a floating screen/window in the 3D space. There is no immersion or depth, and worst of all: content specifically made for VR (like 360&deg; videos) isn't automatically integrated. 

** TODO : YOUTUBE VIDEO ** 

Additionally, here is also nothing developers can do about this (yet): while CSS supports various 3D-related properties, there is no CSS media-query for VR that would allow us to make a vr-version of a site (though support for this type of thing [is being worked on][custom-media-queries]). Even then, browsers would need additional directives to allow a full integration.  

[custom-media-queries]: https://drafts.csswg.org/mediaqueries-5/#script-custom-mq

WebVR itself also can't help us here directly: it primarily acts as a layer between the various Virtual Reality hardware setups and the 3D/WebGL capabilities of the HTML `<canvas>` element. This means we can build [fully immersive, VR-ready 3D experiences][webvrexamples], but that we **have to use lower-level tech like shaders, 3D objects and raycasting, instead of our beloved HTML/CSS/.onclick**. For many, this would mean investing in a new skillset or even hiring new engineers to create a VR webpage and then they would still be stuck having to maintain two very different codebases. 

So we set out to find a way to **render standard HTML/CSS webpages into a 3D/VR context**. 

[comment]: <> (Luckily, there is another standard in the works that might help with this: [WebVR][webvrrocks]. WebVR acts as a layer between the various Virtual Reality hardware setups and the 3D/WebGL capabilities of the HTML `<canvas>` element. Using these building blocks, we can indeed build a [fully immersive, VR-ready experience][webvrexamples]. The main problem is: **we cannot build this using our known tools of HTML, CSS and things like JavaScript click events**. Instead, we have to resort to the ever so slightly more complex shaders, 3D objects and raycasting: no more flexbox or rounded corners for you! This would result in having to maintain two very diverse versions of a website instead of a single codebase with specific directives per platform (as is the case with most mobile websites today). You would also possibly have to hire a different type of engineer to create this version.)

[webvrexamples]: https://medium.com/samsung-internet-dev/eleven-examples-of-how-webvr-is-being-used-today-cbcb214b816c


## Render HTML to a Texture

A basic approach to our problem would be to render individual elements/containers (the menu, hero image, contact form, ...) to a 2D texture and then map those on separate 3D elements/quads. A pseudocode API for this could look something like in Figure 1. 

[comment]: <> (A basic approach to our problem would be to first render the page to a 2D texture, and then map that texture onto a 3D object. If we create multiple 2D textures per page (e.g., one for the menu, one for the hero image, one for the contact form, etc.) and map them on separate 3D elements, we can transform them individually. Pseudocode could look something like this:)

![Rasterization example](images/1_rasterization.png)
<div class="caption">Figure 1: Rasterization API example (Bullet template from [spyropress][spyropress])</div>

[spyropress]: http://spyropress.com/themes/bullet-multipurpose-vertical-menu-wp-theme.html

Sadly, as it turns out, this type of _rasterization_ API is not available in JavaScript. The reasons for this are not fully clear to me (as this should be quite do-able, if non-trivial, to provide for browsers), but [the best reference I have found cites mainly security concerns][rasterize_security]. For example, Firefox does have something similar, but [you can only use it in privileged Firefox contexts][ff_rasterize] (what MDN refers to as chrome :)).

[ff_rasterize]: https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawWindow
[rasterize_security]: http://robert.ocallahan.org/2011/09/risks-of-exposing-web-page-pixel-data.html

Luckily, as usual, we were not the only ones thinking of this type of functionality. It turned out other people had already cracked this problem, using two very different workarounds! 

### Workaround 1: The `<foreignObject>` tag

The first method uses the obscure `<foreignObject>` [tag][foreignObject_spec] of the Scalable Vector Graphis (SVG) format. Any HTML content placed within this tag will be rendered as if it was a normal page and then the result is injected back into the SVG. If we subsequently render the SVG to a texture with the existing `<canvas>` APIs, we can use it in 3D as well! 

However, again due to security considerations, links to external resources (e.g., CSS, JS) are not supported so these need to be inlined/copied and images need to be base64 encoded. Das Surma has [a good write-up][dom2texture] on this method and the most mature library we have found that does this is [rasterizeHTML][rasterizeHTML].

Getting these textures into 3D is then [relatively straightforward][dom2texture2canvas], but adding interaction with the 3D result is a bit more involved. A good approach is to just let the underlying DOM version of the page handle the intricacies and "simply" re-render the updated elements when needed. To alert the DOM that something happened, we can for example map mouse coordinates to texture UV coordinates and then use those through [document.elementFromPoint] to find the associated DOM element. Using [MutationObservers][mutationObserver] we can then trigger an appropriate re-render. [HTML2Three][html2three] is a nice POC implementation of these concepts that uses [rasterizeHTML][rasterizeHTML] interally.

You can see the results of this method using the rasterizeHTML and HTML2Three libraries on our testsite in Figure 3. Note that it is possible to get better results, but this requires substantially more setup and overhead.   

[foreignObject_spec]: https://developer.mozilla.org/en-US/docs/Web/SVG/Element/foreignObject
[dom2texture]: http://dassur.ma/things/dom2texture/
[dom2texture2canvas]: http://dassur.ma/things/dom2texture/#step-2-drawing-it-to-canvas
[rasterizeHTML]: https://cburgmer.github.io/rasterizeHTML.js/ 
[html2three]: https://github.com/marciot/html2three
[elementFromPoint]: https://developer.mozilla.org/en-US/docs/Web/API/Document/elementFromPoint
[mutationObserver]: https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver

### Workaround 2: Re-invent the browser's rendering implementation

This method sounds a bit crazy: instead of letting the browser render the HTML/CSS, let's write a custom renderer and draw the elements to a texture ourselves! This is possible because the `<canvas>` element also provides a mature and extensive [2D drawing API][context2D]. It is also possible because we can use JavaScript to obtain the results of the browser's "layouting" calculations and _only_ need to re-implement the actual "styling/painting" logic, see Figure 2. For example, the results of complex positioning setups, such as with flexbox, can be queried using [`getBoundingClientRect`][getBoundingClientRect]. Other results of CSS parsing are available using [`getComputedStyle`][getComputedStyle]. 

[context2D]: https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D
[getBoundingClientRect]: https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect
[getComputedStyle]: https://developer.mozilla.org/en-US/docs/Web/API/Window/getComputedStyle


![CSS Layouting and Styling](images/2_css_layoutvsstyle.png)
<div class="caption">Figure 2: CSS layouting and styling (simplified: [CSS obviously does a lot more][CSSpipeline])</div>

[CSSpipeline]: https://developers.google.com/web/fundamentals/performance/rendering/

Still, even "only" implementing this part is quite daunting, though it can make for an [interesting learning experience][kosamari]. We decided to use the excellent [HTML2Canvas][html2canvas] library, which is able to get extremely impressive results (almost picture perfect) in our tests, see Figure 3. 

[html2canvas]: https://html2canvas.hertzen.com/
[kosamari]: https://twitter.com/kosamari/status/930558641706500097

Since the end-result of this method is very similar to Workaround 1 (2D textures), the way we deal with user input and updates is also similar. The [HTML-GL][htmlgl] library wraps [HTML2Canvas][html2canvas] (and is in the progress of supporting other "backends") to support 3D. 

[htmlgl]: https://github.com/PixelsCommander/HTML-GL

![Method comparison](images/3_testsite_comparison.png)
<div class="caption">Figure 3: Method comparison for three methods (b-d) and the browser's ground truth (a)</div>


### Performance problems

So, great! In [HTML2Canvas][html2canvas] we have found an excellent renderer! Now we just need to use it on our individual elements and done! Sadly, this is a post (at least partly) about performance and we haven't talked about performance yet... 

As it turns out, neither of the two workarounds offer satisfactory performance. While Das Surma [quoted per-frame times of 10ms for Workaround 1][dom2texture3] on a 2017 macbook pro, in our tests it was closer to 275ms per-frame on a good gaming PC (i7, 16GB RAM, NVidia GTX 1070). Workaround 2 was even worse, with average times of 380ms. This was tested on our test page (see Figure 3) that triggers various animations and DOM changes over about 30s. This translates to 2.5 - 3.5 FPS, a long stretch from the [60 FPS target][RAIL]. 

Further testing revealed that this is mainly due to the overhead of creating new 2D texture contexts, not how much you actually end up drawing to them. This is why we observed that the two libraries that add VR/3D options, [HTML2Three][html2three] and [HTML-GL][htmlgl], both take a different approach to limit the amount of generated textures: HTML2Three prefers a texture per high-level `<div>` container, while HTML-GL tries to optimize by putting elements that have 3D CSS transform properties on separate "layers", see Figure 4 ([Note that the browser does something similar internally to optimize for animations etc.][Layers]). Most examples that we've seen stay at about 5 separate textures. This means performance can be kept usable in some cases, but severely limits our options for animating and interacting with individual elements. To be fair, is not "the fault" of rasterizeHTML or html2canvas as they were probably never made with high performance in mind, but that doesn't change our problem. 

![Method comparison](images/4_layers.png)
<div class="caption">Figure 4: Generation of individual textures/3D elements (b-d) vs the DOM representation (a). Element x has a CSS 3D transform set.</div>

[dom2texture3]: http://dassur.ma/things/dom2texture/#performance
[RAIL]: https://developers.google.com/web/fundamentals/performance/rail
[Layers]: https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count


## Transcode HTML to 3D objects and shaders 

In order to get better performance and thus support large amounts of animated objects, we decided to create our own solution. Seeing as the creation and maintenance of 2D textures was deemed the biggest bottleneck, we decided to try to go directly from DOM to 3D, without the intermediate step. For this, our high-level approach is very similar to that of HTML2Canvas in Figure 2: we re-purpose the browser's layouting calculations but instead of using the 2D drawing APIs, we directly create **3D equivalents for our 2D DOM elements for use in WebGL**.

We quickly deemed working directly with WebGL too daunting. The excellent [Three.js library][threejs] is a better option, but at that time it didn't offer good WebVR support out-of-the-box. Then we came accross [A-Frame, a project made specifically for WebVR by Mozilla][aframe]. A-Frame uses Three.js internally but adds a lot of goodies on top, both for interacting with WebVR and easily creating 3D scenes. For example, A-Frame sports its own HTML-alike syntax, see Figure 5. However, even though it looks and feels like HTML, this syntax doesn't take CSS into account and you cannot use typical HTML tags like `<div>` or `<img>`. 


![Method comparison](images/5_aframe.png)
<div class="caption">Figure 5: Simple DOM2AFrame example (not taking into account positioning/layout)</div>

[threejs]: https://threejs.org/
[aframe]: https://aframe.io/

Thus, [**DOM2AFrame**][d2a_github] was born! We simply loop over all DOM elements in a target `<div>` and create equivalent A-Frame elements for each. In most cases, this is a simple Quad (3D object constrained by 4 points) to represent the item's bounding rectangle. Items containing text are a combination of a background Quad (A-Frame text objects don't allow background colors) and a text mesh on top. Only more complex elements like `<select>` require a more intricate approach. Figure 5 shows a simple example of how we transcode HTML/CSS to equivalent A-Frame syntax. 

Using this setup, we can maintain a direct mapping between DOM elements and their A-Frame 3D equivalents, since both simply exist in JS. This leads to much less overhead when propagating input events and makes it easier to figure out which 3D objects should be updated when something changes in the DOM, when compared to the render-to-texture-first approach. 

TODO: custom CSS. "This works well for rendering elements onto a 2D plane, but if we want to be able to position individual elements in 3D using CSS, we need to go further. CSS does have 3D transforms, but no VR media query. Would mean that site also applies 3D transforms without VR, which we want to avoid. For now: custom properties! in future: custom media query and use existing properties!"

This setup has a good performance in theory... in practice there were several issues that make this a bit more difficult. 

### Performance aspects

- MutationObserver doesn't catch everything
	- changes triggered don't propagate through other elementst -> custom check subtrees (custom caching layer)
- eventlisteners
- requestAnimationFrame (TODO: check if still called!)
- grouping layout queries (but not sure if that's needed here, since we never directly write to DOM)
- CSS overflow 

### Unsupported
- Hover and pseudo-classes 
- auto-detect: * { margin: 15px; }







---
**Thanks to Sander Vanhove and Wouter Vanmontfort for working with me on this topic.**

*Custom figures were made with https://draw.io, Visual Studio Code and PowerPoint*

https://webvr.edm.uhasselt.be 

Last update: 04/12/2017  
Live version: https://github.com/rmarx/DOM2AFrame-perfcal17