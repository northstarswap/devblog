---
title: LAB sliders are basically cheating at color theory
tags:
  - art
  - technique
  - color-theory
---
This article will discuss my favorite way of picking colors for digital art: LAB sliders, and why I believe that after the initial learning curve they make choosing colors significantly easier.

# On HSB
Occasionally, I come across this image making fun of the complexity of choosing colors using an HSB hue cube versus the simplicity of using RGB sliders:

![[res/lab-sliders/rgb-versus-hsb.png]]

RGB sliders are—in my experience—rarely used as a primary way of picking colors when working in digital art due to the relative difficulty in controlling value. 

HSB however is the default for most digital painting software and likely what most people stick with. Curving the selection of colors on the HSB hue cube/color wheel and rotating the wheel is common advice in art communities to achieve more natural-looking colors.

# HSB and Color Theory
If you're not familiar with how ambient light interacts with light and shadow with color, I recommend watching these videos by Marco Bucci which give a fantastic overview of the topic: 

* [Understanding Shadow Colors (Ambient Light Part 2)](https://www.youtube.com/watch?v=gwLQ0cDb4cE)
* [The Power Of Light On Skin Colors](https://www.youtube.com/watch?v=gwLQ0cDb4cE)

To put it in the shortest terms: if you want light to look natural, you must control the **hue** of any given color relative to the others based on their **luminosity** and the color of the lights in your scene. 

This means that if the ambient light is orange, then shadows are blue. If the ambient light is blue, then the shadows are orange. This works for any complementary colors, but we'll be focusing on natural temperature in this article.

## How HSB handles temperature
**It doesn't.**
When using HSB, it's entirely up to you to mentally calculate colors based on the factors listed above. The Brightness (B) slider does not do this for you. For example, here's a cube painted by only changing the Brightness versus properly calculating temperature-aware colors:

![[res/lab-sliders/temperature-cubes.png]]

The luminosity-only palette looks distinctly digital because such a palette would only exist in a perfectly white room with 1 pure color of light, which doesn't happen often. This is why it's often recommended to move the picker in an arc in the hue cube and move around the color wheel—it's to compensate for temperature.

# Introducing LAB
**LAB** is a color space often used in photography for its ability to preserve values while changing temperature and overall hue versus HSL which is a simple transform on RGB. 

LAB separates color into 3 components: **Lightness**, **Green to Red (A)** and **Blue to Yellow (B)**. 

![[res/lab-sliders/lab-sliders.png]]

The key advantage of lab is that  **the L component isolates luminance from chromaticity**, giving you direct control over values without accidentally changing hues.

This also means that changing only the L component completely avoids perceived value distortion caused by chromaticity. This makes LAB very useful for painting in monochrome, as the hue won't shift around when changing the luminance.

## Color Temperature with LAB
You may have already noticed, but the **B** component is very close to a temperature slider. After we dial in our primary hue **we can just change L and B to create temperature-aware colors**. For warm light, increase the **B** value. For cold light, decrease the **B** value.

![[res/lab-sliders/lab-colors.gif]]

In this example, I use LAB sliders to automatically create roughly temperature-correct colors for a **deep blue** in a room with a **warm light**. Notice how we didn't have to fiddle with saturation as we approached peak brightness either. It's automatically solved for us.

## LAB colors in HSB
Now, let's look at what colors we picked—viewed through the HSB color wheel

![[res/lab-sliders/viewing-with-hsb.gif]]

It created the arc + twist automatically.

Additionally, to create more varied colors you can also fuzz the **A** value to make things look less perfect. 