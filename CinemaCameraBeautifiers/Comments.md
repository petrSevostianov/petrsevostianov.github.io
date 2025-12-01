So, if I get this right, you're saying there's a hidden color transform?
How does your product interact with this?

Yes, camera manufacturers apply a non-linear gamut transform. I assume it is done to make images look "better" sacrificing linearity.
Don't get me wrong, I'm aware of artistic color grading, it is absolutely necessary and important. However, artistic color grading should be applied at the end of the pipeline, after all technical corrections are done.
It is ok to have non-linear color grading inside the camera, as long as camera footage is the end of the pipeline or followed by artistic color grading only.
But it is not the case in virtual production. 

There linearity is crucial.
1. LED wall calibration.
The goal here is to be able to send a specific color to the LED wall and get the same color captured by the camera.
We want to be able to do so for any color.
If the camera is linear and the LED wall is linear, the only thing we need to do is to capture 3 color patches (R, G, B) and calculate a 3x3 matrix transform.
Described [here](https://dev.epicgames.com/documentation/en-us/unreal-engine/camera-color-calibration-for-in-camera-vfx-in-unreal-engine).

2. Keying. The goal here is to "unmix" green and "color of an object". Semitransparent objects, edges, motion blur, defocus are examples of physical color mixtures.
When the keying algorithm sees a color somewhere between green and object color, it interprets it as a mixture of green and object color, so it can "unmix" it, reconstructing object color and alpha.
These color blends are done by physics, they are linear.
So it is crucial to provide the keying algorithm with a linear color input.

3. Lighting calibration and reconstruction. This part is related to our product [CyberGaffer](https://www.cybergaffer.com).
The goal here is very similar to LED wall calibration.
The software should be able to predict lighting on the virtual production set from parameters (channels) of light fixtures.
This ability is required to solve the inverse problem: to calculate light fixture parameters from desired lighting on the set.
This desired lighting is captured in a virtual scene in Unreal Engine, so the software can copy that lighting to the physical world.
Linearity here is critical for calibration.
During the calibration, we measure how the reference sphere is illuminated by each channel of each light fixture.
Each measurement is an image of a sphere (sample).
So to calculate how this sphere will look when we turn on a light fixture at, let's say, 50% red, 25% green, and 100% blue, we need to multiply each sample by the corresponding channel intensity and sum them up.
This is only possible if samples were captured in linear space, or we know how to convert them to linear space.




Q:
Oh - and the image used at the top of this thread and the article is false, as all calibration is subtractive.
Calibration can never increase any aspect of colour, especially extend gamut as the image shows.

Working in 'linear light' is a totally different subject, and has nothing to do with gamut, and nothing to do with Virtual Set issues.

A:
Regarding the image at the top of the article: the plots shown are the actual measurements and the result of the calculated transform. In the article, I explain how these measurements were taken and how the transform was computed step by step.


Q:
Sigh…. What are the primaries of the RGB LED light sources you are using? What are the primaries of the bayer filter on the sensor? Are you sure you’re looking at raw sensor values or have they been gamut mapped into a standard gamut already by the raw processing pipeline? Looks to me like you’re getting distortion around the far saturated values where either the lights or the filters are degenerate and effectively clipping. If your paper were serious and you knew what you were doing, you’d have provided Yxy values for all of this already.

A:
> What are the primaries of the RGB LED light sources you are using?
Thank you for noting this. I'll add this information to the article. [Here](https://www.lcsc.com/datasheet/C784541.pdf) is the datasheet of the LED modules we used. On page 5 you can find the emission spectra of R, G, and B LEDs. But the choice of LED wavelengths is not critical. If we perform the same experiment with different LEDs, we will find the same non-linear transform in a different coordinate system.

> What are the primaries of the bayer filter on the sensor?
There is no such thing as "primaries of the bayer filter". Response of each color filter (R, G, B) to the spectrum of light is defined by the spectral sensitivity function of the sensor. The vendor does not provide this information.
Fortunately, we do not need this information to find the non-linear transform.

> Are you sure you're looking at raw sensor values or have they been gamut mapped into a standard gamut already by the raw processing pipeline?
That is the thing. By using cinema cameras we can only get the image provided through SDI/HDMI and we are not able to say what transforms were applied inside the camera. The article is about how to measure and cancel out these unknown transforms.
That is standard gamut? If you mean any matrix 3x3 multiplication usually used for gamut manipulations, then, since matrix multiplication is a linear operation, it does not affect linearity. The article is about non-linear transforms on top of that.
In this article, I mentioned that BRAW SDK allows to read values without that nonlinear transform applied. Unfortunately, this does not help with SDI/HDMI output. It works with braw files only.

> Looks to me like you're getting distortion around the far saturated values where either the lights or the filters are degenerate and effectively clipping.
That is not a physical effect. The transform is digitally applied by BRAW SDK during raw decoding (can be disabled) and by the firmware for other formats and outputs (cannot be disabled).


Q:
Sorry, but I do not understand.
As gamut can never be expanded, how are the plots actual measurements, unless the scales are different?

A: At [this](https://petrsevostianov.github.io/3DLUTViewer.html?showChromaTriangle=0&lut=CinemaCameraBeautifiers/bmpcc6k.cube) plot, you can see the measured colors.
How are they measured?
The calibration device displays multiple colors by setting RGB values using LEDs. Each LED (r, g, b) can be set from 0 to 100% intensity. So we can divide that range into 15 steps linearly spaced. For every channel we have 16 levels of intensity (including 0). By combining these levels for R, G, and B channels, we can display 16x16x16 = 4096 colors. Next we record a video of those 4096 samples and pick camera responses for each sample in the form of R, G, B values. Now to visualize these measurements, we can plot them in 3D space with R, G, B axes. This is what you see in the plot.
Next, we remove the transfer function described in **Blackmagic Generation 5 Color Science Technical Reference**. That means we apply the inverse transfer function to each measured R, G, B value. That gives us [this](https://petrsevostianov.github.io/3DLUTViewer.html?showChromaTriangle=0&lut=CinemaCameraBeautifiers/bmpcc6k_TransferFunctionRemoved.cube) plot. In this plot, we can already see that the camera responds nonlinearly to linear changes of LED intensities.

Please look closely at the plot. You will find curved lines where you would expect straight lines if the camera were linear.


Q: KaurTheColorist
Don't newer BMD cameras already have turning this off as an option?
https://www.bhphotovideo.com/lit_files/1162746.pdf#page=72

Α:
Wow! They do! Thank you so much for the info! I didn't know that. I am 99% sure that with `GAMUT COMPRESSION = OFF` it would produce linear responses. I'll add this to the article.

KaurTheColorist, do you know what other cameras have an option to turn off gamut compression? Or will it come with future firmware updates to older BMD cameras?

Do you also know if they published this transform? It might be necessary to apply it in post if we want to keep the initial look from BMD.

Q: onejaguar
You're skipping over the reference to OpenVPCal, which I think they're using as the basis for correcting metamerism, but since OpenVPCal uses 1D tranforms and 3x3 matricies it doesn't correct for non-linearity in the system.
My question for OP would be why is he using a different device with a different spectral emission to measure the camera? Why not just use the camera to measure more patches on the wall to correct for non-linearity or clipping in both the wall and the spectral response and color processing of the camera?

Q: onejaguar
I don't think you can just ignore the spectral sensitivity of the camera sensor. How can you be sure the non-linearity you're measuring isn't caused by cross-talk between the spectral emission of your LED device and the spectral response of the camera? And if that is part of what you're measuring it will be different for different LED devices (e.g. the wall in a virtual production stage).

A:
That is an excellent question!
Long story short, for that specific goal (measuring non-linearity) we can ignore the spectral sensitivity of the camera sensor (with a single "However"). 

Let me explain why.
A camera has subpixels with color filters. Each filter has a spectral sensitivity function defining how it responds to different wavelengths of light. `S_R(λ), S_G(λ), and S_B(λ)`
Also, we have 3 types of LEDs with their emission spectra: `E_R(λ), E_G(λ), and E_B(λ)`
We can drive each LED from 0 to 100% intensity. Let's denote intensities as `I_R`, `I_G`, and `I_B` (I stands for Input). We can read camera responses as `O_R`, `O_G`, and `O_B` (O stands for Output).

How to calculate camera response to, let's say, red LED only?

`O_R = ∫ S_R(λ) * [I_R * E_R(λ)] dλ`

`O_G = ∫ S_G(λ) * [I_R * E_R(λ)] dλ`

`O_B = ∫ S_B(λ) * [I_R * E_R(λ)] dλ`

In this formula, `I_R * E_R(λ)` is the spectrum of light emitted by the red LED at intensity `I_R`.

Next, we multiply this spectrum by the spectral sensitivity function of each color filter and integrate over all wavelengths λ.
The same applies to green and blue LEDs.
Since light is additive, we can calculate the camera response to any combination of LED intensities by summing up responses to each LED:

`O_R = ∫ S_R(λ) * [I_R * E_R(λ) + I_G * E_G(λ) + I_B * E_B(λ)] dλ`

`O_G = ∫ S_G(λ) * [I_R * E_R(λ) + I_G * E_G(λ) + I_B * E_B(λ)] dλ`

`O_B = ∫ S_B(λ) * [I_R * E_R(λ) + I_G * E_G(λ) + I_B * E_B(λ)] dλ`

We can rewrite these equations as a vector of outputs `O = [O_R, O_G, O_B]` equal to some `Matrix3x3 M` multiplied by a vector of inputs `I = [I_R, I_G, I_B]`:

`O = M * I`

That means that no matter what spectral sensitivity functions S and emission spectra E are, the relation between input intensities I and output responses O is always linear and can be described by a matrix M.

As for the cross-talk you mention, you can see that every LED affects all 3 outputs `O_R`, `O_G`, and `O_B`. So cross-talk is already included in matrix M and is linear.

If our goal is to find non-linearity, we can pick any 3 LEDs with any spectral emission and perform the measurements described in the article.

**However!!** The choice of LEDs is still important. And here is why.
Imagine a gamut 2D space of camera responses. It is similar to the CIE 1931 chromaticity diagram for human vision, but for camera responses.

Our 3 LEDs will produce 3 points in this space. All responses we can get by mixing these LEDs will lie inside the triangle formed by these 3 points. So we can explore only the responses inside this triangle; the remaining area of the gamut will be covered by the fog of war.

So if we pick 3 very similar LEDs, the triangle will be very small, and most of the gamut will be unexplored. 
In this study, the LEDs we used have the same spectrum as typical virtual production LED walls. Fortunately, there are not that many LEDs that exist because their spectrum depends on the semiconductor material they are made of (not applicable to phosphor-based LEDs such as white LEDs).

There are still some areas of the gamut unexplored; for instance, we can imagine a monochromatic laser that will produce a point outside of the triangle we explored. So, to be precise, we don't know how the Beautifier behaves in that spot. But for practical purposes, the explored area covers all camera responses we can achieve in the virtual production set, including LED lights reflected from any possible physical surface.

Sorry for the long answer! It is not easy to explain these things without visuals or interactive elements.



Q: Thank you for writing this article. We are streamlining our greenscreen-VP workflow and the gamut-compression has exactly highlighted one of the challenges we’re still facing, but I couldn’t put my finger on what the issue was. Sad it’s a Saturday, but I’m excited to test it all out next week!

Edit: ps. Love the work on CyberGaffer. Would love to learn more about it. Can I get in contact with you?

Thank you so much! I'm glad you found this research useful. Probably the best way to get in touch is the CyberGaffer Discord server: https://discord.gg/hRgKPkSAZm
We have the entire team there. Please mention or DM me there (petr.sevostianov).






