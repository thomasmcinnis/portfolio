---
title: A dumb image slider with Javascript (or how I learnt to love modulo)
pubDate: 2024-03-22
description: Learning modulo by making something kind of stupid
tags: ['Javascript']
isDraft: true
slug: learn-to-love-modulo
---

Something that annoys me about developer blogs is they rarely show the *dumb* implementation of something they inevitably created before arriving at the nice and neat solution. I recently created an image slider with the aim of using the least HTML and CSS possible, and just a sprinkle of Javascript. 

Along the way I built something unnecessarily silly in order to continuously rotate through the images in the slider. So I want to share that dumb implementation, because it enabled me to reach the *good* implementation using modulo operator.

## What am I building?

Currently I am working on a small ecommerce page for selling only a handful of items. Product pages need a light-weight image slider with very few features.

Requirements:

- Return to the first image after reaching the end and can move both ways
- Images have consistent dimensions
- Allows for responsive design (not discussed in this post)
- Adding images should simply involve inserting them in the HTML (ie. no passing the quantity of images as an argument) 

The first thing I did was use some google-fu to search for "native JS CSS image carousel/slider" and of course there are *tonnes* of great guides.

Almost universally however I found myself thinking "this html looks a mess, and this css is overly long". 

## Start with the end in mind

I decided to start with what my ideal HTML would look like for a slider, and let that be my north star. 

*(Nb. For the sake of keeping this post short, I move navigation arrows outside the slider rather than over it, and responsiveness, accessibility considerations and touch interaction are ignored. A production implementation of this slider has more moving parts!)*

This is the most readable way I can think of to lay out an image slider with the fewest extraneous divs:

```html
<div class="slider">
    <button data-action="prev">←</button>
    <div class="frame">
	<div class="slides">
	    <img src="01.jpg" alt="" />
	    <img src="02.jpg" alt="" />
	    <!-- Add images here -->	
	</div>
    </div>
    <button data-action="next">→</button>
</div>
```

Given the simplicity of the HTML above, the CSS sort of becomes self-evident:

```css
.slider {
    display: flex;
}

.slider > button {
    background-color: transparent;
    border: none;
    cursor: pointer;
	/* only required as using unicode char for icons */
    font-size: 2rem; 
}

.frame {
    width: 600px;
    overflow: hidden;
}

.slides {
    display: flex;
    transition: transform 0.3s ease-in-out;
}
```

The overall size of the slider is determined by the width value we set to `.frame` and the height of the images in the html. (Making this responsive does not add significant complexity once we get the basics done!)

To change the position of the `.slides` element within the `.frame` we can now use something like `transform: translateX()`. By using a negative offset, we can push the slides div to the left. Because the `.slides` element is constrained by the `.frame` div, translating the slides by increments of 100% gets the job done.

But how do we handle going back to the first image (eg. 0% offset) after reaching the end? That is where we have to get clever with some Javascript for our navigation buttons.

## A wrong (but fun) approach with array shift and pop

My priority at this point is to make something naive, which works, and that I can easily *reason about*. Code that I cannot easily reason about will cost me more time. 

As a result, I arrived at a dumb but strangely charming solution. If you have some experience programming, you know what a more desirable implementation will be! Don't worry, we will get there.

I decided to collect the required offsets to display each image in an array. All we have to do is collect the index positions from the image nodes in the DOM, so the array would start out as something like `[0, 1, 2]`. To use that as a percent offset in our transform, we can multiply by 100 and make it a negative expression.

Instead of traversing the array for each offset, I figured I could set the 0th index to be the desired offset, and *rotate the values in the array*. On each button press of 'next' or 'previous' I could shift and push, or pop and unshift, such that the items that the 0th position is always the desired % offset.

This perpetually rotates the array values forward and back, meaning we naturally return to the first image after reaching the end (or the last after reaching the first with the previous button.) 

Stepping forward one image, our array is now `[1, 2, 0]`. Using the `Oth` value of one results in `transform: translateX(-100%)`.

> Rotating the offset amounts in the array, and always using the `0th` value brings me some deep and unreasonable amount of joy, even though I know it isn't the 'correct' approach.

More importantly once I had this highly visual proxy for image transforms in the array, finishing the implementation was trivial. 

To put this into practice, here is the full javascript implementation:

```js
const slides = document.querySelector('.slides');
const images = document.querySelectorAll('.slides img');

let positions = [];
for (let i = 0; i < images.length; i++) {
    positions.push(i);
}

function handleClick(event) {
    const eventTargetAction = event.target.closest('[data-action]');
    if (!eventTargetAction) return;

    const action = eventTargetAction.dataset.action;

	// rotate the translation positions array forward
    if (action === 'next') {
        const first = positions.shift();
        positions.push(first);
    }

	// rotate the translation positions array backward
    if (action === 'prev') {
        const last = positions.pop();
        positions.unshift(last);
    }

    // update the transform on the slides div with the new Oth value
    slides.style.transform = `translateX(-${positions[0] * 100}%)`;
}

document.addEventListener('click', handleClick);
```

## The better approach with modulo

I knew the modulo operator would come into play to solve rotating through the images. But reasoning about modulo in this context was not yet natural to me. Like a lot of things in code, things are hard until they aren't! Coming back to refactor was inevitable.

Modulo is one of those things which can be a real head-scratcher as you learn, then all of a sudden seems incredibly obvious.

For the un-initiated, the modulo operator (represented by the percent symbol) results in the remainder after a whole division. Eg, seven can be divided by two three *whole* times with a *remainder* of one.

- `5 % 6 = 5`: six cannot fit into five, so the remainder is five.
- `11 % 6 = 5`, six can be divided into 11 once, with remainder five.

Notice how five modulo six, and eleven modulo six returns the same value. This side effect is what we are interested in for rotation. And this value, is *always* a subset of six. It can never be anything outside the range 0 - 6.

Connecting the dots, with a set of *n* images, if we increment a value outside our *n* range, using the modulo operator puts us back in range. Sticking with six images, if we increment to six we return zero, convenient for us programmers who think that lists start at zero not one!

Let's go ahead and remove the positions array entirely, replacing it with a `currentImage` value and  `imageCount` derived from the length of the node-list of images. 

```js
const slides = document.querySelector('.slides');
const images = document.querySelectorAll('.slides img');

const imageCount = images.length;
let currentImage = 0;

function handleClick(event) {
    const eventTargetAction = event.target.closest('[data-action]');
    if (!eventTargetAction) return;

    const action = eventTargetAction.dataset.action;

    if (action === 'next') {
        currentImage = (currentImage + 1) % imageCount;
    }

    if (action === 'prev') {
        currentImage = (currentImage - 1 + imageCount) % imageCount;
    }

    // update the transform on the slides div with the new currImg value
    slides.style.transform = `translateX(-${currentImage * 100}%)`;
}

document.addEventListener('click', handleClick);
```

We can now increment and decrement the `currentImage` value and run the modulo operation on it.

With the negative case, we can add the total image number from the decremented value to ensure it never actually becomes a negative number. For example starting with `currentImage = 0` and  `imageCount = 6`:

- Decrement by one, plus imageCount, equals five
- Five modulo six, is five - the last image in the node list given it is zero indexed

This is a much less quirky solution, and requires fewer operations for each step forward and backward. While not significant in this context, remember that array shift and unshift methods are quite costly, having to move every element in an array. 

If you cannot yet reason through the modulo operator, then I don't think there is anything wrong with using some other proxy like I did initially. Working through my functioning implementation with the goal of using modulo really enabled me to internalise how modulo works. Discovering through experimentation and finding your own path to success, no matter how idiosyncratic, is an essential part of learning.
